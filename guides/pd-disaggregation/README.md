# Well-lit Path: P/D Disaggregation

## Overview

This guide demonstrates how to deploy Llama-70B using vLLM's P/D disaggregation support with NIXL. This guide has been validated on:

* an 8xH200 cluster with InfiniBand networking
* an 8xH200 cluster on GKE with RoCE networking

> WARNING: We are still investigating and optimizing performance for other hardware and networking configurations

In this example, we will demonstrate a deployment of `Llama-3.3-70B-Instruct-FP8` with:

* 4 TP=1 Prefill Workers
* 1 TP=4 Decode Worker

## P/D Best Practices

P/D disaggregation provides more flexibility in navigating the trade-off between throughput and interactivity([ref](https://arxiv.org/html/2506.05508v1)).
In particular, due to the elimination of prefill interference to the decode phase, P/D disaggregation can achieve lower inter token latency (ITL), thus
improving interactivity. For a given ITL goal, P/D disaggregation can benefit overall throughput by:

* Specializing P and D workers for compute-bound vs latency-bound workloads
* Reducing the number of copies of the model (increasing KV cache RAM) with wide parallelism

However, P/D disaggregation is not a target for all workloads. We suggest exploring P/D disaggregation for workloads with:

* Large models (e.g. Llama-70B+, not Llama-8B)
* Longer input sequence lengths (e.g 10k ISL | 1k OSL, not 200 ISL | 200 OSL)
* Sparse MoE architectures with opportunities for wide-EP

As a result, as you tune your P/D deployments, we suggest focusing on the following parameters:

* **Heterogeneous Parallelism**: deploy P workers with less parallelism and more replicas and D workers with more parallelism and fewer replicas
* **xPyD Ratios**: tuning the ratio of P workers to D workers to ensure balance for your ISL|OSL ratio

For very large models leveraging wide-EP, traffic for KV cache transfer may contend with expert parallelism when the ISL|OSL ratio is also high. We recommend starting with RDMA for KV cache transfer before attempting to leverage TCP, as TCP transfer requires more tuning of UCX under NIXL.

## Hardware Requirements

This guide expects 8 Nvidia GPUs of any kind, and RDMA via InfiniBand or RoCE between all pods in the workload.

## Prerequisites

* Have the [proper client tools installed on your local system](../prereq/client-setup/README.md) to use this guide.
* Ensure your cluster infrastructure is sufficient to [deploy high scale inference](../prereq/infrastructure)
* Configure and deploy your [Gateway control plane](../prereq/gateway-provider/README.md).
* Have the [Monitoring stack](../../docs/monitoring/README.md) installed on your system.
* Create a namespace for installation.

  ```
  export NAMESPACE=llm-d-pd # or any other namespace (shorter names recommended)
  kubectl create namespace ${NAMESPACE}
  ```

* [Create the `llm-d-hf-token` secret in your target namespace with the key `HF_TOKEN` matching a valid HuggingFace token](../prereq/client-setup/README.md#huggingface-token) to pull models.
* [Choose an llm-d version](../prereq/client-setup/README.md#llm-d-version)

## Installation

Use the helmfile to compose and install the stack. The Namespace in which the stack will be deployed will be derived from the `${NAMESPACE}` environment variable. If you have not set this, it will default to `llm-d-pd` in this example.

### Deploy

```bash
cd guides/pd-disaggregation
helmfile apply -n ${NAMESPACE}
```

**_NOTE:_** You can set the `$RELEASE_NAME_POSTFIX` env variable to change the release names. This is how we support concurrent installs. Ex: `RELEASE_NAME_POSTFIX=pd-2 helmfile apply -n ${NAMESPACE}`

**_NOTE:_** This uses Istio as the default provider, see [Gateway Options](./README.md#gateway-options) for installing with a specific provider.

### Gateway options

To see specify your gateway choice you can use the `-e <gateway option>` flag, ex:

```bash
helmfile apply -e kgateway -n ${NAMESPACE}
```

To see what gateway options are supported refer to our [gateway provider prereq doc](../prereq/gateway-provider/README.md#supported-providers). Gateway configurations per provider are tracked in the [gateway-configurations directory](../prereq/gateway-provider/common-configurations/).

You can also customize your gateway, for more information on how to do that see our [gateway customization docs](../../docs/customizing-your-gateway.md).

#### Infrastructure provider specifics

This guide uses RDMA via InfiniBand or RoCE for disaggregated serving kv-cache transfer. The resource attributes required to configure accelerator networking are not yet standardized via [Kubernetes Dynamic Resource Allocation](https://kubernetes.io/docs/concepts/scheduling-eviction/dynamic-resource-allocation/) and so are parameterized per infra provider in the Helm charts. If your provider has a custom setting you will need to update the charts before deploying.

### Install HTTPRoute

Follow provider specific instructions for installing HTTPRoute.

#### Install for "kgateway" or "istio"

```bash
kubectl apply -f httproute.yaml -n ${NAMESPACE}
```

#### Install for "gke"

```bash
kubectl apply -f httproute.gke.yaml -n ${NAMESPACE}
```

## Verify the Installation

* Firstly, you should be able to list all helm releases to view the 3 charts got installed into your chosen namespace:

```bash
helm list -n ${NAMESPACE}
NAME        NAMESPACE   REVISION    UPDATED                                 STATUS      CHART                       APP VERSION
gaie-pd     llm-d-pd    1           2025-08-24 12:54:51.231537 -0700 PDT    deployed    inferencepool-v1.2.0        v1.2.0
infra-pd    llm-d-pd    1           2025-08-24 12:54:46.983361 -0700 PDT    deployed    llm-d-infra-v1.3.6          v0.3.0
ms-pd       llm-d-pd    1           2025-08-24 12:54:56.736873 -0700 PDT    deployed    llm-d-modelservice-v0.3.17  v0.3.0
```

* Out of the box with this example you should have the following resources:

```bash
kubectl get all -n ${NAMESPACE}
NAME                                                    READY   STATUS    RESTARTS   AGE
pod/gaie-pd-epp-54444ddc66-qv6ds                        1/1     Running   0          2m35s
pod/infra-pd-inference-gateway-istio-56d66db57f-zwtzn   1/1     Running   0          2m41s
pod/ms-pd-llm-d-modelservice-decode-84bf6d5bdd-jzfjn    2/2     Running   0          2m30s
pod/ms-pd-llm-d-modelservice-prefill-86f6fb7cdc-8kfb8   1/1     Running   0          2m30s
pod/ms-pd-llm-d-modelservice-prefill-86f6fb7cdc-g6wmp   1/1     Running   0          2m30s
pod/ms-pd-llm-d-modelservice-prefill-86f6fb7cdc-jx2w2   1/1     Running   0          2m30s
pod/ms-pd-llm-d-modelservice-prefill-86f6fb7cdc-vzcb8   1/1     Running   0          2m30s

NAME                                       TYPE           CLUSTER-IP    EXTERNAL-IP   PORT(S)                        AGE
service/gaie-pd-epp                        ClusterIP      10.16.0.255   <none>        9002/TCP,9090/TCP              2m35s
service/gaie-pd-ip-bb618139                ClusterIP      None          <none>        54321/TCP                      2m35s
service/infra-pd-inference-gateway-istio   ClusterIP      10.16.3.74    10.16.4.3     15021:31707/TCP,80:34096/TCP   2m41s

NAME                                               READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/gaie-pd-epp                        1/1     1            1           2m36s
deployment.apps/infra-pd-inference-gateway-istio   1/1     1            1           2m42s
deployment.apps/ms-pd-llm-d-modelservice-decode    1/1     1            1           2m31s
deployment.apps/ms-pd-llm-d-modelservice-prefill   4/4     4            4           2m31s

NAME                                                          DESIRED   CURRENT   READY   AGE
replicaset.apps/gaie-pd-epp-54444ddc66                        1         1         1       2m36s
replicaset.apps/infra-pd-inference-gateway-istio-56d66db57f   1         1         1       2m42s
replicaset.apps/ms-pd-llm-d-modelservice-decode-84bf6d5bdd    1         1         1       2m31s
replicaset.apps/ms-pd-llm-d-modelservice-prefill-86f6fb7cdc   4         4         4       2m31s
```

**_NOTE:_** This assumes no other guide deployments in your given `${NAMESPACE}` and you have not changed the default release names via the `${RELEASE_NAME}` environment variable.

## Using the stack

For instructions on getting started making inference requests see [our docs](../../docs/getting-started-inferencing.md)

## Tuning Selective PD

Selective PD is a feature in the `inference-scheduler` within the context of prefill-decode dissagregation, although it is disabled by default. This features enables routing to just decode even with the P/D deployed. To enable it, you will need to set `threshold` value for the `pd-profile-handler` plugin, in the [GAIE values file](./gaie-pd/values.yaml). You can see the value of this here:

```bash
cat gaie-pd/values.yaml | yq '.inferenceExtension.pluginsCustomConfig."pd-config.yaml"' | yq '.plugins[] | select(.type == "pd-profile-handler")'
type: pd-profile-handler
parameters:
  threshold: 0 # update this
  hashBlockSize: 5
```

Some examples in which you might want to do selective PD might include:

* When the prompt is short enough that the amount of work split inference into prefill and decode phases, and then open a kv transfer between those two GPUs is greater than the amount of work to do both phases on the same decode inference worker.
* When Prefill units are at full capacity.

For information on this plugin, see our [`pd-profile-handler` docs in the inference-scheduler](https://github.com/llm-d/llm-d-inference-scheduler/blob/v0.3.0/docs/architecture.md?plain=1#L205-L210)

## Cleanup

To remove the deployment:

```bash
# Remove the model services
helmfile destroy -n ${NAMESPACE}

# Remove the infrastructure
helm uninstall ms-pd -n ${NAMESPACE}
helm uninstall gaie-pd -n ${NAMESPACE}
helm uninstall infra-pd -n ${NAMESPACE}
```

**_NOTE:_** If you set the `$RELEASE_NAME_POSTFIX` environment variable, your release names will be different from the command above: `infra-$RELEASE_NAME_POSTFIX`, `gaie-$RELEASE_NAME_POSTFIX` and `ms-$RELEASE_NAME_POSTFIX`.

### Cleanup HTTPRoute

Follow provider specific instructions for deleting HTTPRoute.

#### Cleanup for "kgateway" or "istio"

```bash
kubectl delete -f httproute.yaml -n ${NAMESPACE}
```

#### Cleanup for "gke"

```bash
kubectl delete -f httproute.gke.yaml -n ${NAMESPACE}
```

## Customization

For information on customizing a guide and tips to build your own, see [our docs](../../docs/customizing-a-guide.md)

## Benchmarking

To run benchmarks against the installed llm-d stack, you need [run_only.sh](https://github.com/llm-d/llm-d-benchmark/blob/main/existing_stack/run_only.sh), a template file from [guides/benchmark](../benchmark/), and a Persistent Volume Claim (PVC) to store the results. Follow the instructions in the [benchmark doc](../benchmark/README.md). 


### Example

This example uses [run_only.sh](https://github.com/llm-d/llm-d-benchmark/blob/main/existing_stack/run_only.sh) with the template [pd_vllm_bench_random_concurrent_template.yaml](../benchmark/pd_vllm_bench_random_concurrent_template.yaml).

The benchmark launches a pod (`llmdbench-harness-launcher`) that, in this case, uses `vllm-bench` with a random synthetic workload named `random_concurrent`. 
The results will be stored under on the provided PVC, accessible through the `llmdbench-harness-launcher` pod. Each experiment is saved under the `requests` folder, e.g.,/`requests/vllm-benchmark_<experiment ID>_random_concurrent_<model name>` folder. 

Several results files will be created (see [Benchmark doc](../benchmark/README.md)), including a yaml file in a "standard" benchmark report format (see [Benchmark Report](https://github.com/llm-d/llm-d-benchmark/blob/main/docs/benchmark_report.md)).


  ```bash
  curl -L -O https://raw.githubusercontent.com/llm-d/llm-d-benchmark/main/existing_stack/run_only.sh
  chmod u+x run_only.sh
  LLMD_BRANCH=847c8b1
  select f in $(
      curl -s https://api.github.com/repos/llm-d/llm-d/contents/guides/benchmark?ref=${LLMD_BRANCH} | 
      sed -n '/[[:space:]]*"name":[[:space:]][[:space:]]*"\(pd.*\_template\.yaml\)".*/ s//\1/p'
    ); do 
    curl -LJO "https://raw.githubusercontent.com/llm-d/llm-d/${LLMD_BRANCH}/guides/benchmark/$f"
    break
  done
  ```
  
Choose the `pd_vllm_bench_random_concurrent_template.yaml` template, then run:

  ```bash
  export NAMESPACE=dpikus-pd     # replace with your namespace
  export BENCHMARK_PVC=workload-pvc   # replace with your PVC name
  export GATEWAY_SVC=infra-pd-inference-gateway-istio  # replace with your exact service name
  envsubst < pd_vllm_bench_random_concurrent_template.yaml > config.yaml
  ```

After that, edit `config.yaml` if needed and run the command
  ```bash
  ./run_only.sh -c config.yaml
  ```

Here is an example of a benchmark_report for this run (using a lighter llm-d stack, serving `Qwen2-0.5B-Instruct-FP8` with a single GPU per pod):
  ```yaml
  metrics:
    latency:
      inter_token_latency:
        mean: 1.7027542555110875
        p0p1: 0.005726298317313194
        p1: 0.08259909227490425
        p10: 1.5963684767484665
        p5: 1.4918556436896324
        p90: 1.7450897954404356
        p95: 1.900194305926561
        p99: 3.7971320003271103
        p99p9: 4.350280856713653
        stddev: 0.491742694784182
        units: ms/token
      request_latency:
        mean: 2061.5178888838273
        p0p1: 1799.0411128033884
        p1: 1865.8342311391607
        p10: 2044.959097262472
        p5: 2034.0365397045389
        p90: 2096.155120478943
        p95: 2100.8053639205173
        p99: 2122.9221803275873
        p99p9: 2131.13206202304
        stddev: 53.00134750612688
        units: ms
      time_per_output_token:
        mean: 1.6655221995758918
        p0p1: 1.6548033180908561
        p1: 1.655552857364337
        p10: 1.657608111255669
        p5: 1.657460162600687
        p50: 1.6634504793694727
        p75: 1.6677861516312644
        p90: 1.6712081555679843
        p95: 1.678933067717128
        p99: 1.6985431693641617
        p99p9: 1.7042389566016323
        stddev: 0.009151976293312795
        units: ms/token
      time_to_first_token:
        mean: 397.6612115075113
        p0p1: 132.28069308632985
        p1: 201.22961815912277
        p10: 380.79249653965235
        p5: 376.0226161684841
        p50: 403.82984187453985
        p75: 422.16307553462684
        p90: 426.5187329147011
        p95: 428.2202695729211
        p99: 456.76445202901965
        p99p9: 468.13354993797856
        stddev: 52.687202002113494
        units: ms
    requests:
      input_length:
        mean: 10000.0
        units: count
      output_length:
        mean: 1000.0
        units: count
      total: 32
    throughput:
      output_tokens_per_sec: 484.96805257844136
      requests_per_sec: 0.48496805257844133
      total_tokens_per_sec: 5334.648578362855
    time:
      duration: 65.98372785560787
      start: 1768132346.0
  scenario:
    load:
      args:
        burstiness: 1.0
        max_concurrency: 1
        num_prompts: 32
        request_rate: inf
      name: vllm-benchmark
    model:
      name: RedHatAI/Qwen2-0.5B-Instruct-FP8
  version: '0.1'
  ```
