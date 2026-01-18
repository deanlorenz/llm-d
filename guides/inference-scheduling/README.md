# Well-lit Path: Intelligent Inference Scheduling

## Overview

This guide deploys the recommended out of the box [scheduling configuration](https://github.com/llm-d/llm-d-inference-scheduler/blob/main/docs/architecture.md) for most vLLM deployments, reducing tail latency and increasing throughput through load-aware and prefix-cache aware balancing. This can be run on a single GPU that can load [Qwen/Qwen3-0.6B](https://huggingface.co/Qwen/Qwen3-0.6B).

This profile defaults to the approximate prefix cache aware scorer, which only observes request traffic to predict prefix cache locality. The [precise prefix cache aware routing feature](../precise-prefix-cache-aware) improves hit rate by introspecting the vLLM instances for cache entries and will become the default in a future release.

## Hardware Requirements

This example out of the box requires 2 GPUs of any supported kind:
- **NVIDIA GPUs**: Any NVIDIA GPU (support determined by the inferencing image used)
- **Intel XPU/GPUs**: Intel Data Center GPU Max 1550 or compatible Intel XPU device
- **TPUs**: Google Cloud TPUs (when using GKE TPU configuration)

**Alternative CPU Deployment**: For CPU-only deployment (no GPUs required), see the [Hardware Backends](#hardware-backends) section for CPU-specific deployment instructions. CPU deployment requires Intel/AMD CPUs with 64 cores and 64GB RAM per replica.

## Prerequisites

- Have the [proper client tools installed on your local system](../prereq/client-setup/README.md) to use this guide.
- Ensure your cluster infrastructure is sufficient to [deploy high scale inference](../prereq/infrastructure)
- Have the [Monitoring stack](../../docs/monitoring/README.md) installed on your system.
- Create a namespace for installation.
  
  ```
  export NAMESPACE=llm-d-inference-scheduler # or any other namespace (shorter names recommended)
  kubectl create namespace ${NAMESPACE}
  ```

- [Create the `llm-d-hf-token` secret in your target namespace with the key `HF_TOKEN` matching a valid HuggingFace token](../prereq/client-setup/README.md#huggingface-token) to pull models.
- [Choose an llm-d version](../prereq/client-setup/README.md#llm-d-version)
- [Skip if using standalone-inference-scheduling] Configure and deploy your [Gateway control plane](../prereq/gateway-provider/README.md)


## Installation

Use the helmfile to compose and install the stack. The Namespace in which the stack will be deployed will be derived from the `${NAMESPACE}` environment variable. If you have not set this, it will default to `llm-d-inference-scheduler` in this example.

**_IMPORTANT:_** When using long namespace names (like `llm-d-inference-scheduler`), the generated pod hostnames may become too long and cause issues due to Linux hostname length limitations (typically 64 characters maximum). It's recommended to use shorter namespace names (like `llm-d`) and set `RELEASE_NAME_POSTFIX` to generate shorter hostnames and avoid potential networking or vLLM startup problems.

### Deploy

```bash
cd guides/inference-scheduling
```

<!-- TABS:START -->
<!-- TAB:GPU deployment  -->

**GPU deployment**
```bash
helmfile apply -n ${NAMESPACE}
```

<!-- TAB:CPU deployment  -->
**CPU-only deployment:**
```bash
helmfile apply -e cpu -n ${NAMESPACE}
```

<!-- TABS:END -->

**_NOTE:_** You can set the `$RELEASE_NAME_POSTFIX` env variable to change the release names. This is how we support concurrent installs. Ex: `RELEASE_NAME_POSTFIX=inference-scheduling-2 helmfile apply -n ${NAMESPACE}`

### Inference Request Scheduler and Hardware Options

#### Inference Request Scheduler
<!-- TABS:START -->

<!-- TAB:Gateway Option -->
##### Gateway Option

**_NOTE:_** This uses Istio as the default gateway provider, see [Gateway Options](./README.md#gateway-options) for installing with a specific provider.

To specify your gateway choice you can use the `-e <gateway option>` flag, ex:

```bash
helmfile apply -e kgateway -n ${NAMESPACE}
```


For DigitalOcean Kubernetes Service (DOKS):

```bash
helmfile apply -e digitalocean -n ${NAMESPACE}
```

 **_NOTE:_** DigitalOcean deployment uses public Qwen/Qwen3-0.6B model (no HuggingFace token required) and is optimized for DOKS GPU nodes with automatic tolerations and node selectors. Gateway API v1 compatibility fixes are automatically included.

To see what gateway options are supported refer to our [gateway provider prereq doc](../prereq/gateway-provider/README.md#supported-providers). Gateway configurations per provider are tracked in the [gateway-configurations directory](../prereq/gateway-provider/common-configurations/).

You can also customize your gateway, for more information on how to do that see our [gateway customization docs](../../docs/customizing-your-gateway.md).

<!-- TAB: Standalone Option -->
##### Standalone Option
With this option, the inference scheduler is deployed along with a sidecar Envoy proxy instead of a proxy provisioned using the Kubernetes Gateway API.

To deploy as a standalone inference scheduler, use the `-e standalone` flag, ex:

```bash
helmfile apply -e standalone -n ${NAMESPACE}
```

<!-- TABS:END -->

#### Hardware Backends

Currently in the `inference-scheduling` example we suppport configurations for `xpu`, `tpu`, `cpu`, and `cuda` GPUs. By default we use modelserver values supporting `cuda` GPUs, but to deploy on one of the other hardware backends you may use:

```bash
helmfile apply -e xpu  -n ${NAMESPACE} # targets istio as gateway provider with XPU hardware
# or
helmfile apply -e gke_tpu  -n ${NAMESPACE} # targets GKE externally managed as gateway provider with TPU hardware
# or
helmfile apply -e cpu  -n ${NAMESPACE} # targets istio as gateway provider with CPU hardware
```

##### CPU Inferencing
This case expects using 4th Gen Intel Xeon processors (Sapphire Rapids) or later. 

### Install HTTPRoute When Using Gateway option

Follow provider specific instructions for installing HTTPRoute.

#### Install for "kgateway" or "istio"

```bash
kubectl apply -f httproute.yaml -n ${NAMESPACE}
```

#### Install for "gke"

```bash
kubectl apply -f httproute.gke.yaml -n ${NAMESPACE}
```

#### Install for "digitalocean"

```bash
kubectl apply -f httproute.yaml -n ${NAMESPACE}
```

## Verify the Installation

<!-- TABS:START -->

<!-- TAB:Gateway Option -->
### Gateway option

- Firstly, you should be able to list all helm releases to view the 3 charts got installed into your chosen namespace:

```bash
helm list -n ${NAMESPACE}
NAME                        NAMESPACE                 REVISION  UPDATED                               STATUS    CHART                     APP VERSION
gaie-inference-scheduling   llm-d-inference-scheduler 1         2025-08-24 11:24:53.231918 -0700 PDT  deployed  inferencepool-v1.2.0-rc.1 v1.2.0-rc.1
infra-inference-scheduling  llm-d-inference-scheduler 1         2025-08-24 11:24:49.551591 -0700 PDT  deployed  llm-d-infra-v1.3.4        v0.3.0
ms-inference-scheduling     llm-d-inference-scheduler 1         2025-08-24 11:24:58.360173 -0700 PDT  deployed  llm-d-modelservice-v0.3.8 v0.3.0
```

- Out of the box with this example you should have the following resources:

```bash
kubectl get all -n ${NAMESPACE}
NAME                                                                  READY   STATUS    RESTARTS   AGE
pod/gaie-inference-scheduling-epp-f8fbd9897-cxfvn                     1/1     Running   0          3m59s
pod/infra-inference-scheduling-inference-gateway-istio-6787675b9swc   1/1     Running   0          4m3s
pod/ms-inference-scheduling-llm-d-modelservice-decode-8ff7fd5b58lw9   2/2     Running   0          3m55s
pod/ms-inference-scheduling-llm-d-modelservice-decode-8ff7fd5bt5f9s   2/2     Running   0          3m55s

NAME                                                         TYPE           CLUSTER-IP    EXTERNAL-IP   PORT(S)                        AGE
service/gaie-inference-scheduling-epp                        ClusterIP      10.16.3.151   <none>        9002/TCP,9090/TCP              3m59s
service/gaie-inference-scheduling-ip-18c12339                ClusterIP      None          <none>        54321/TCP                      3m59s
service/infra-inference-scheduling-inference-gateway-istio   LoadBalancer   10.16.1.195   10.16.4.2     15021:30274/TCP,80:32814/TCP   4m3s

NAME                                                                 READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/gaie-inference-scheduling-epp                        1/1     1            1           4m
deployment.apps/infra-inference-scheduling-inference-gateway-istio   1/1     1            1           4m4s
deployment.apps/ms-inference-scheduling-llm-d-modelservice-decode    2/2     2            2           3m56s

NAME                                                                           DESIRED   CURRENT   READY   AGE
replicaset.apps/gaie-inference-scheduling-epp-f8fbd9897                        1         1         1       4m
replicaset.apps/infra-inference-scheduling-inference-gateway-istio-678767549   1         1         1       4m4s
replicaset.apps/ms-inference-scheduling-llm-d-modelservice-decode-8ff7fd5b8    2         2         2       3m56s
```

<!-- TAB: Standalone Option -->
### Standalone option

- Firstly, you should be able to list all helm releases to view the 2 charts got installed into your chosen namespace:

```bash
helm list -n ${NAMESPACE}
NAME                        NAMESPACE                 REVISION  UPDATED                               STATUS    CHART                     APP VERSION
gaie-inference-scheduling   llm-d-inference-scheduler 1         2025-08-24 11:24:53.231918 -0700 PDT  deployed  inferencepool-v1.2.0-rc.1 v1.2.0-rc.1
ms-inference-scheduling     llm-d-inference-scheduler 1         2025-08-24 11:24:58.360173 -0700 PDT  deployed  llm-d-modelservice-v0.3.8 v0.3.0
```

- Out of the box with this example you should have the following resources:

```bash
kubectl get all -n ${NAMESPACE}
NAME                                                                  READY   STATUS    RESTARTS   AGE
pod/gaie-inference-scheduling-epp-f8fbd9897-cxfvn                     1/1     Running   0          3m59s
pod/ms-inference-scheduling-llm-d-modelservice-decode-8ff7fd5b58lw9   2/2     Running   0          3m55s
pod/ms-inference-scheduling-llm-d-modelservice-decode-8ff7fd5bt5f9s   2/2     Running   0          3m55s

NAME                                                         TYPE           CLUSTER-IP    EXTERNAL-IP   PORT(S)                        AGE
service/gaie-inference-scheduling-epp                        ClusterIP      10.16.3.151   <none>        9002/TCP,9090/TCP              3m59s
service/gaie-inference-scheduling-ip-18c12339                ClusterIP      None          <none>        54321/TCP                      3m59s

NAME                                                                 READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/gaie-inference-scheduling-epp                        1/1     1            1           4m
deployment.apps/ms-inference-scheduling-llm-d-modelservice-decode    2/2     2            2           3m56s

NAME                                                                           DESIRED   CURRENT   READY   AGE
replicaset.apps/gaie-inference-scheduling-epp-f8fbd9897                        1         1         1       4m
replicaset.apps/ms-inference-scheduling-llm-d-modelservice-decode-8ff7fd5b8    2         2         2       3m56s
```
**_NOTE:_** This assumes no other guide deployments in your given `${NAMESPACE}` and you have not changed the default release names via the `${RELEASE_NAME}` environment variable.

<!-- TABS:END -->

## Using the stack

For instructions on getting started making inference requests see [our docs](../../docs/getting-started-inferencing.md)

## Cleanup

To remove the deployment:

```bash
# From examples/inference-scheduling
helmfile destroy -n ${NAMESPACE}

# Or uninstall manually
helm uninstall infra-inference-scheduling -n ${NAMESPACE} --ignore-not-found
helm uninstall gaie-inference-scheduling -n ${NAMESPACE}
helm uninstall ms-inference-scheduling -n ${NAMESPACE}
```

**_NOTE:_** If you set the `$RELEASE_NAME_POSTFIX` environment variable, your release names will be different from the command above: `infra-$RELEASE_NAME_POSTFIX`, `gaie-$RELEASE_NAME_POSTFIX` and `ms-$RELEASE_NAME_POSTFIX`.

### Cleanup HTTPRoute when using Gateway option

Follow provider specific instructions for deleting HTTPRoute.
#### Cleanup for "kgateway" or "istio"

```bash
kubectl delete -f httproute.yaml -n ${NAMESPACE}
```

#### Cleanup for "gke"

```bash
kubectl delete -f httproute.gke.yaml -n ${NAMESPACE}
```

#### Cleanup for "digitalocean"

```bash
kubectl delete -f httproute.yaml -n ${NAMESPACE}
```
## Customization

For information on customizing a guide and tips to build your own, see [our docs](../../docs/customizing-a-guide.md)

## Benchmarking

To run benchmarks against the installed llm-d stack, you need [run_only.sh](https://github.com/llm-d/llm-d-benchmark/blob/main/existing_stack/run_only.sh), a template file from [guides/benchmark](../benchmark/), and a Persistent Volume Claim (PVC) to store the results. Follow the instructions in the [benchmark doc](../benchmark/README.md). 

### Example

This example uses [run_only.sh](https://github.com/llm-d/llm-d-benchmark/blob/main/existing_stack/run_only.sh) with the template [precise_guidellm_template.yaml](../benchmark/inference_scheduling_shared_prefix_template.yaml).


  ```bash
  curl -L -O https://raw.githubusercontent.com/llm-d/llm-d-benchmark/main/existing_stack/run_only.sh
  chmod u+x run_only.sh
  LLMD_BRANCH=11ef16b
  select f in $(
      curl -s https://api.github.com/repos/llm-d/llm-d/contents/guides/benchmark?ref=${LLMD_BRANCH} | 
      sed -n '/[[:space:]]*"name":[[:space:]][[:space:]]*"\(inference_scheduling.*\_template\.yaml\)".*/ s//\1/p'
    ); do 
    curl -LJO "https://raw.githubusercontent.com/llm-d/llm-d/${LLMD_BRANCH}/guides/benchmark/$f"
    break
  done
  ```
  
  Choose the `inference_scheduling_shared_prefix_template.yaml` template, then run:

  ```bash
  export NAMESPACE=dpikus-intel-inf     # replace with your namespace
  export BENCHMARK_PVC=workload-pvc   # replace with your PVC name
  export GATEWAY_SVC=infra-inference-scheduling-inference-gateway-istio  # replace with your exact service name
  ```

After that, run the command
  ```bash
  envsubst < inference_scheduling_shared_prefix_template.yaml > config.yaml
  ./run_only.sh -c config.yaml
  ```

  The results from inference-perf run are:

  ```
  ...
  2026-01-14 12:58:15,472 - inference_perf.client.filestorage.local - INFO - Report files will be stored at: /requests/inference-perf_1768395442_shared_prefix_synthetic_inference-scheduling-Qwen3-0.6B
  2026-01-14 12:58:18,414 - inference_perf.loadgen.load_generator - INFO - Stage 0 - run started
  Stage 0 progress: 100%|██████████| 1.0/1.0 [00:52<00:00, 52.06s/it] 
  2026-01-14 12:59:10,503 - inference_perf.loadgen.load_generator - INFO - Stage 0 - run completed
  2026-01-14 12:59:11,504 - inference_perf.loadgen.load_generator - INFO - Stage 1 - run started
  Stage 1 progress: 100%|██████████| 1.0/1.0 [00:52<00:00, 52.05s/it]   
  2026-01-14 13:00:03,566 - inference_perf.loadgen.load_generator - INFO - Stage 1 - run completed
  2026-01-14 13:00:04,569 - inference_perf.loadgen.load_generator - INFO - Stage 2 - run started
  Stage 2 progress: 100%|██████████| 1.0/1.0 [00:52<00:00, 52.05s/it]   
  2026-01-14 13:00:56,620 - inference_perf.loadgen.load_generator - INFO - Stage 2 - run completed
  Stage 3 progress:   0%|          | 0/1.0 [00:00<?, ?it/s]2026-01-14 13:00:57,621 - inference_perf.loadgen.load_generator - INFO - Stage 3 - run started
  Stage 3 progress: 100%|██████████| 1.0/1.0 [00:52<00:00, 52.14s/it]  2026-01-14 13:01:49,675 - inference_perf.loadgen.load_generator - INFO - Stage 3 - run completed
  Stage 3 progress: 100%|██████████| 1.0/1.0 [00:52<00:00, 52.05s/it]
  2026-01-14 13:01:50,677 - inference_perf.loadgen.load_generator - INFO - Stage 4 - run started
  Stage 4 progress:  98%|█████████▊| 0.975/1.0 [00:51<00:01, 53.81s/it]2026-01-14 13:02:42,726 - inference_perf.loadgen.load_generator - INFO - Stage 4 - run completed
  Stage 4 progress: 100%|██████████| 1.0/1.0 [00:52<00:00, 52.05s/it]  
  2026-01-14 13:02:43,727 - inference_perf.loadgen.load_generator - INFO - Stage 5 - run started
  Stage 5 progress:  98%|█████████▊| 0.976/1.0 [00:51<00:01, 47.18s/it]             2026-01-14 13:03:35,770 - inference_perf.loadgen.load_generator - INFO - Stage 5 - run completed
  Stage 5 progress: 100%|██████████| 1.0/1.0 [00:52<00:00, 52.04s/it]  
  2026-01-14 13:03:36,771 - inference_perf.loadgen.load_generator - INFO - Stage 6 - run started
  Stage 6 progress: 100%|██████████| 1.0/1.0 [00:52<00:00, 52.05s/it]   
  2026-01-14 13:04:28,826 - inference_perf.loadgen.load_generator - INFO - Stage 6 - run completed
  2026-01-14 13:04:29,932 - inference_perf.reportgen.base - INFO - Generating Reports...
  ...
  ```

  Benchmarking Report (for rate=10):

  ```yaml
  metrics:
    latency:
      inter_token_latency:
        max: 0.05777170229703188
        mean: 0.0028540196591486065
        min: 4.38326969742775e-06
        p0p1: 4.994813352823258e-06
        p1: 0.0002613391727209091
        p10: 0.0023753787390887737
        p25: 0.0025633699260652065
        p5: 0.0020118332467973232
        p50: 0.0027567152865231037
        p75: 0.0030960491858422756
        p90: 0.0035414122976362705
        p95: 0.003907579928636551
        p99: 0.005562575720250604
        p99p9: 0.007995677366852985
        units: s/token
      normalized_time_per_output_token:
        max: 0.7239119322039187
        mean: 0.013230468383828011
        min: 0.0012901967719534603
        p0p1: 0.0013735272390653008
        p1: 0.002475093616671815
        p10: 0.002669081147359975
        p25: 0.002781771464626337
        p5: 0.0025888777642649073
        p50: 0.0030131658049867838
        p75: 0.003294003644896293
        p90: 0.05602821090935993
        p95: 0.08938193067442622
        p99: 0.09968416544550564
        p99p9: 0.4370293470854131
        units: s/token
      request_latency:
        max: 0.9693497093394399
        mean: 0.7592659112159162
        min: 0.6293312991037965
        p0p1: 0.6321117059849203
        p1: 0.6379123754054308
        p10: 0.6786826994735747
        p25: 0.7066267926711589
        p5: 0.6639299782225863
        p50: 0.7455745704937726
        p75: 0.8018235113704577
        p90: 0.8615147506818176
        p95: 0.8907231074757874
        p99: 0.9520903471671045
        p99p9: 0.9693295779344626
        units: s
      time_per_output_token:
        max: 0.0036182620824547485
        mean: 0.0028531568113432273
        min: 0.002386520749756268
        p0p1: 0.0023887520123694445
        p1: 0.002406616739970671
        p10: 0.002551740905867373
        p25: 0.002654319440489598
        p5: 0.002492177187595631
        p50: 0.002806923320160422
        p75: 0.0030105473229014024
        p90: 0.003242099913000215
        p95: 0.0033541438971445257
        p99: 0.0035732898914102407
        p99p9: 0.0036182615814177553
        units: s/token
      time_to_first_token:
        max: 0.04281094064936042
        mean: 0.022396996933966875
        min: 0.01594797894358635
        p0p1: 0.01681194866169244
        p1: 0.01792819821741432
        p10: 0.01954832240007818
        p25: 0.02079031872563064
        p5: 0.019093053368851542
        p50: 0.022028648993000388
        p75: 0.023641378502361476
        p90: 0.025527860410511496
        p95: 0.026530779246240855
        p99: 0.03026527327019721
        p99p9: 0.03792530749272588
        units: s
    requests:
      failures: 0
      input_length:
        max: 2468.0
        mean: 2430.7
        min: 2391.0
        p0p1: 2391.499
        p1: 2394.0
        p10: 2409.0
        p25: 2418.0
        p5: 2403.95
        p50: 2429.0
        p75: 2444.25
        p90: 2454.0
        p95: 2458.0
        p99: 2462.0
        p99p9: 2466.503
        units: count
      output_length:
        max: 510.0
        mean: 228.03
        min: 1.0
        p0p1: 2.996
        p1: 8.0
        p10: 13.0
        p25: 255.0
        p5: 8.0
        p50: 256.0
        p75: 256.0
        p90: 256.0
        p95: 257.0
        p99: 268.50999999999954
        p99p9: 509.00200000000007
        units: count
      total: 500
    throughput:
      output_tokens_per_sec: 2254.4490015619667
      requests_per_sec: 9.88663334456855
      total_tokens_per_sec: 26285.888672204743
    time:
      duration: 49.86801022803411
  scenario:
    load:
      args:
        api:
          headers: null
          streaming: true
          type: completion
        circuit_breakers: null
        data:
          input_distribution: null
          output_distribution: null
          path: null
          shared_prefix:
            enable_multi_turn_chat: false
            num_groups: 32
            num_prompts_per_group: 32
            output_len: 256
            question_len: 256
            system_prompt_len: 2048
          trace: null
          type: shared_prefix
        load:
          circuit_breakers: []
          interval: 1.0
          num_workers: 224
          request_timeout: null
          stages:
          - concurrency_level: null
            duration: 50
            num_requests: null
            rate: 2.0
          - concurrency_level: null
            duration: 50
            num_requests: null
            rate: 5.0
          - concurrency_level: null
            duration: 50
            num_requests: null
            rate: 8.0
          - concurrency_level: null
            duration: 50
            num_requests: null
            rate: 10.0
          - concurrency_level: null
            duration: 50
            num_requests: null
            rate: 12.0
          - concurrency_level: null
            duration: 50
            num_requests: null
            rate: 15.0
          - concurrency_level: null
            duration: 50
            num_requests: null
            rate: 20.0
          sweep: null
          trace: null
          type: constant
          worker_max_concurrency: 100
          worker_max_tcp_connections: 2500
        metrics: null
        report:
          prometheus:
            per_stage: false
            summary: true
          request_lifecycle:
            per_request: true
            per_stage: true
            summary: true
        server:
          api_key: null
          base_url: http://infra-inference-scheduling-inference-gateway-istio.dpikus-intel-inf.svc.cluster.local:80
          ignore_eos: true
          model_name: Qwen/Qwen3-0.6B
          type: vllm
        storage:
          google_cloud_storage: null
          local_storage:
            path: /requests/inference-perf_1768395442_shared_prefix_synthetic_inference-scheduling-Qwen3-0.6B
            report_file_prefix: null
          simple_storage_service: null
        tokenizer:
          pretrained_model_name_or_path: Qwen/Qwen3-0.6B
          token: null
          trust_remote_code: null
      metadata:
        stage: 3
      name: inference-perf
    model:
      name: unknown
  version: '0.1'
  ```