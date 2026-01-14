# Feature: Precise Prefix Cache Aware Routing

## Overview

This guide demonstrates how to configure the inference scheduler to use the new precise prefix cache aware routing based on [vLLM KV-Events](https://github.com/vllm-project/vllm/issues/16669) data. Precise prefix cache aware routing pulls up-to-date prefix cache status from serving instances, eliminating the need for additional indexing services and increasing cache hit rate at high throughput.

## Prerequisites

- Have the [proper client tools installed on your local system](../prereq/client-setup/README.md) to use this guide.
- Configure and deploy your [Gateway control plane](../prereq/gateway-provider/README.md).
- Have the [Monitoring stack](../../docs/monitoring/README.md) installed on your system.
- Create a namespace for installation.
  
  ```
  export NAMESPACE=llm-d-precise # or any other namespace (shorter names recommended)
  kubectl create namespace ${NAMESPACE}
  ```

- [Create the `llm-d-hf-token` secret in your target namespace with the key `HF_TOKEN` matching a valid HuggingFace token](../prereq/client-setup/README.md#huggingface-token) to pull models.
- [Choose an llm-d version](../prereq/client-setup/README.md#llm-d-version)

## Installation

Use the helmfile to compose and install the stack. The Namespace in which the stack will be deployed will be derived from the `${NAMESPACE}` environment variable. If you have not set this, it will default to `llm-d-precise` in this example.

### Deploy

```bash
cd guides/precise-prefix-cache-aware
helmfile apply -n ${NAMESPACE}
```

**_NOTE:_** You can set the `$RELEASE_NAME_POSTFIX` env variable to change the release names. This is how we support concurrent installs. Ex: `RELEASE_NAME_POSTFIX=kv-events-2 helmfile apply -n ${NAMESPACE}`

**_NOTE:_** This uses Istio as the default provider, see [Gateway Options](./README.md#gateway-options) for installing with a specific provider.

### Gateway options

To see specify your gateway choice you can use the `-e <gateway option>` flag, ex:

```bash
helmfile apply -e kgateway -n ${NAMESPACE}
```

To see what gateway options are supported refer to our [gateway provider prereq doc](../prereq/gateway-provider/README.md#supported-providers). Gateway configurations per provider are tracked in the [gateway-configurations directory](../prereq/gateway-provider/common-configurations/).

You can also customize your gateway, for more information on how to do that see our [gateway customization docs](../../docs/customizing-your-gateway.md).

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

- Firstly, you should be able to list all helm releases to view the 3 charts got installed into your chosen namespace:

```bash
helm list -n ${NAMESPACE}
NAME            NAMESPACE       REVISION        UPDATED                                 STATUS          CHART                           APP VERSION
gaie-kv-events  llm-d-precise   1               2025-09-25 21:36:52.452999581 +0000 UTC deployed        inferencepool-v1.2.0-rc.1       v1.2.0-rc.1
infra-kv-events llm-d-precise   1               2025-09-25 21:36:50.848300265 +0000 UTC deployed        llm-d-infra-v1.3.4              v0.3.0     
ms-kv-events    llm-d-precise   1               2025-09-25 21:36:55.955958022 +0000 UTC deployed        llm-d-modelservice-v0.3.8       v0.3.0 
```

- Out of the box with this example you should have the following resources:

```bash
kubectl get all -n ${NAMESPACE}
NAME                                                          READY   STATUS    RESTARTS   AGE
pod/gaie-kv-events-epp-687b78968b-wvswh                       1/1     Running   0          80s
pod/infra-kv-events-inference-gateway-istio-949d87f84-zvsp2   1/1     Running   0          85s
pod/ms-kv-events-llm-d-modelservice-decode-b874d48d9-bgm5r    2/2     Running   0          75s
pod/ms-kv-events-llm-d-modelservice-decode-b874d48d9-ph64c    2/2     Running   0          75s

NAME                                              TYPE           CLUSTER-IP   EXTERNAL-IP   PORT(S)                        AGE
service/gaie-kv-events-epp                        ClusterIP      10.16.2.44   <none>        9002/TCP,9090/TCP,5557/TCP     81s
service/gaie-kv-events-ip-805c964d                ClusterIP      None         <none>        54321/TCP                      75s
service/infra-kv-events-inference-gateway-istio   LoadBalancer   10.16.1.30   10.16.4.2     15021:32033/TCP,80:39332/TCP   86s

NAME                                                      READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/gaie-kv-events-epp                        1/1     1            1           81s
deployment.apps/infra-kv-events-inference-gateway-istio   1/1     1            1           86s
deployment.apps/ms-kv-events-llm-d-modelservice-decode    2/2     2            2           76s

NAME                                                                DESIRED   CURRENT   READY   AGE
replicaset.apps/gaie-kv-events-epp-687b78968b                       1         1         1       81s
replicaset.apps/infra-kv-events-inference-gateway-istio-949d87f84   1         1         1       86s
replicaset.apps/ms-kv-events-llm-d-modelservice-decode-b874d48d9    2         2         2       76s
```

**_NOTE:_** This assumes no other guide deployments in your given `${NAMESPACE}` and you have not changed the default release names via the `${RELEASE_NAME}` environment variable.

## Testing this "well lit path"

We have docs on getting started sending inference requests [available here](../../docs/getting-started-inferencing.md) that are general to all examples. However, this example has unique instructions to interact with it which will be provided here:

1. First, you will need to send a basic inference request to your gateway. For in depth documentation on how to do this, please see the link above, but a command will be provided to work out of the box with default settings:

```bash
kubectl port-forward -n ${NAMESPACE} service/infra-kv-events-inference-gateway-istio 8000:80
export LONG_TEXT_200_WORDS="Lorem ipsum dolor sit amet, consectetur adipiscing elit. Sed do eiusmod tempor incididunt ut labore et dolore magna aliqua. Ut enim ad minim veniam, quis nostrud exercitation ullamco laboris nisi ut aliquip ex ea commodo consequat. Duis aute irure dolor in reprehenderit in voluptate velit esse cillum dolore eu fugiat nulla pariatur. Excepteur sint occaecat cupidatat non proident, sunt in culpa qui officia deserunt mollit anim id est laborum. Lorem ipsum dolor sit amet, consectetur adipiscing elit. Sed do eiusmod tempor incididunt ut labore et dolore magna aliqua. Ut enim ad minim veniam, quis nostrud exercitation ullamco laboris nisi ut aliquip ex ea commodo consequat. Duis aute irure dolor in reprehenderit in voluptate velit esse cillum dolore eu fugiat nulla pariatur. Excepteur sint occaecat cupidatat non proident, sunt in culpa qui officia deserunt mollit anim id est laborum."

curl -s http://localhost:8000/v1/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "Qwen/Qwen3-0.6B",
    "prompt": "'"$LONG_TEXT_200_WORDS"'",
    "max_tokens": 50
  }' | jq
```

1. Check the inference-scheduler's prefix-cache-scorer's scores with the following command:

```bash
kubectl logs -l inferencepool=gaie-kv-events-epp -n ${NAMESPACE} --tail 100 | grep "Calculated score" | grep "precise-prefix-cache-scorer/precise-prefix-cache-scorer"
```

You should see output similar to:

```json
{"level":"Level(-4)","ts":"2025-10-07T16:07:36Z","caller":"framework/scheduler_profile.go:165","msg":"Calculated score","x-request-id":"77790804-deb4-441a-9a03-d771d8e20778","objectiveKey":"","incomingModelName":"Qwen/Qwen3-0.6B","targetModelName":"Qwen/Qwen3-0.6B","priority":0,"plugin":"precise-prefix-cache-scorer/precise-prefix-cache-scorer","endpoint":{"name":"ms-kv-events-llm-d-modelservice-decode-75499f8dc5-pbp84","namespace":"llm-d-precise"},"score":0}
{"level":"Level(-4)","ts":"2025-10-07T16:07:36Z","caller":"framework/scheduler_profile.go:165","msg":"Calculated score","x-request-id":"77790804-deb4-441a-9a03-d771d8e20778","objectiveKey":"","incomingModelName":"Qwen/Qwen3-0.6B","targetModelName":"Qwen/Qwen3-0.6B","priority":0,"plugin":"precise-prefix-cache-scorer/precise-prefix-cache-scorer","endpoint":{"name":"ms-kv-events-llm-d-modelservice-decode-75499f8dc5-kgnqh","namespace":"llm-d-precise"},"score":0}
```

1. Repeat the steps above to see the prefix-cache-scorer in action

You should see output similar to:

```json
{"level":"Level(-4)","ts":"2025-10-07T16:09:21Z","caller":"framework/scheduler_profile.go:165","msg":"Calculated score","x-request-id":"f4c967aa-ad15-4be2-8640-55164da18dfa","objectiveKey":"","incomingModelName":"Qwen/Qwen3-0.6B","targetModelName":"Qwen/Qwen3-0.6B","priority":0,"plugin":"precise-prefix-cache-scorer/precise-prefix-cache-scorer","endpoint":{"name":"ms-kv-events-llm-d-modelservice-decode-75499f8dc5-pbp84","namespace":"llm-d-precise"},"score":0}
{"level":"Level(-4)","ts":"2025-10-07T16:09:21Z","caller":"framework/scheduler_profile.go:165","msg":"Calculated score","x-request-id":"f4c967aa-ad15-4be2-8640-55164da18dfa","objectiveKey":"","incomingModelName":"Qwen/Qwen3-0.6B","targetModelName":"Qwen/Qwen3-0.6B","priority":0,"plugin":"precise-prefix-cache-scorer/precise-prefix-cache-scorer","endpoint":{"name":"ms-kv-events-llm-d-modelservice-decode-75499f8dc5-kgnqh","namespace":"llm-d-precise"},"score":1}
```

**_NOTE:_** These logs will only appear for unique requests, so if you don't see repeated instances of these logs make sure to redo them in a unique way.

Notice that the second time we called the `/v1/completions` endpoint, the prefix-cache-scorer was able to return a score for the pod,
indicating that it had cached the KV-blocks from the first call.

## Cleanup

To remove the deployment:

```bash
# Remove the model services
# From examples/precise-prefix-cache-aware
helmfile destroy -n ${NAMESPACE}

# Or uninstall manually
helm uninstall infra-kv-events -n ${NAMESPACE}
helm uninstall gaie-kv-events -n ${NAMESPACE}
helm uninstall ms-kv-events -n ${NAMESPACE}
```

**_NOTE:_** If you set the `$RELEASE_NAME_POSTFIX` environment variable, your release names will be different from the command above: `infra-$RELEASE_NAME_POSTFIX`, `gaie-$RELEASE_NAME_POSTFIX` and `ms-$RELEASE_NAME_POSTFIX`.

## Customization

For information on customizing a guide and tips to build your own, see [our docs](../../docs/customizing-a-guide.md)

## Benchmarking

To run benchmarks against the installed llm-d stack, you need [run_only.sh](https://github.com/llm-d/llm-d-benchmark/blob/main/existing_stack/run_only.sh), a template file from [guides/benchmark](../benchmark/), and a Persistent Volume Claim (PVC) to store the results. Follow the instructions in the [benchmark doc](../benchmark/README.md). 

### Example

This example uses [run_only.sh](https://github.com/llm-d/llm-d-benchmark/blob/main/existing_stack/run_only.sh) with the template [precise_guidellm_template.yaml](../benchmark/precise_guidellm_template.yaml).


  ```bash
  curl -L -O https://raw.githubusercontent.com/llm-d/llm-d-benchmark/main/existing_stack/run_only.sh
  chmod u+x run_only.sh
  LLMD_BRANCH=ec13ae7
  select f in $(
      curl -s https://api.github.com/repos/llm-d/llm-d/contents/guides/benchmark?ref=${LLMD_BRANCH} | 
      sed -n '/[[:space:]]*"name":[[:space:]][[:space:]]*"\(precise.*\_template\.yaml\)".*/ s//\1/p'
    ); do 
    curl -LJO "https://raw.githubusercontent.com/llm-d/llm-d/${LLMD_BRANCH}/guides/benchmark/$f"
    break
  done
  ```

  Choose the `precise_guidellm_template.yaml` template, then run:

  ```bash
  export NAMESPACE=dpikus-precise     # replace with your namespace
  export BENCHMARK_PVC=workload-pvc   # replace with your PVC name
  export GATEWAY_SVC=infra-kv-events-inference-gateway-istio  # replace with your exact service name
  ```

After that run the command
  ```bash
  envsubst < precise_guidellm_template.yaml > config.yaml
  ./run_only.sh -c config.yaml
  ```

  The results from GuideLLM run are:

```  
ℹ Run Summary Info
|===========|==========|==========|======|======|======|=========|======|=====|=========|=====|=====|
| Benchmark | Timings                              ||||| Input Tokens       ||| Output Tokens     |||
| Strategy  | Start    | End      | Dur  | Warm | Cool | Comp    | Inc  | Err | Comp    | Inc | Err |
|           |          |          | Sec  | Sec  | Sec  | Tot     | Tot  | Tot | Tot     | Tot | Tot |
|-----------|----------|----------|------|------|------|---------|------|-----|---------|-----|-----|
| constant  | 16:10:58 | 16:11:28 | 30.0 | 0.0  | 0.0  | 1450.0  | 0.0  | 0.0 | 1450.0  | 0.0 | 0.0 |
| constant  | 16:11:30 | 16:12:00 | 30.0 | 0.0  | 0.0  | 7450.0  | 0.0  | 0.0 | 7450.0  | 0.0 | 0.0 |
| constant  | 16:12:01 | 16:12:31 | 30.0 | 0.0  | 0.0  | 14900.0 | 50.0 | 0.0 | 14900.0 | 0.0 | 0.0 |
|===========|==========|==========|======|======|======|=========|======|=====|=========|=====|=====|


ℹ Text Metrics Statistics (Completed Requests)
|===========|=======|======|=======|========|=======|======|=======|========|=======|=======|========|========|
| Benchmark | Input Tokens               |||| Input Words                |||| Input Characters             ||||
| Strategy  | Per Request || Per Second    || Per Request || Per Second    || Per Request  || Per Second     ||
|           | Mdn   | p95  | Mdn   | Mean   | Mdn   | p95  | Mdn   | Mean   | Mdn   | p95   | Mdn    | Mean   |
|-----------|-------|------|-------|--------|-------|------|-------|--------|-------|-------|--------|--------|
| constant  | 50.0  | 50.0 | 50.0  | 51.8   | 41.0  | 43.0 | 41.0  | 42.4   | 267.0 | 286.0 | 268.6  | 278.3  |
| constant  | 50.0  | 50.0 | 250.1 | 288.7  | 41.0  | 42.0 | 204.3 | 234.8  | 262.0 | 287.0 | 1310.7 | 1515.5 |
| constant  | 50.0  | 50.0 | 500.0 | 1468.7 | 40.0  | 42.0 | 401.5 | 1188.1 | 261.0 | 286.0 | 2584.3 | 7676.5 |
|===========|=======|======|=======|========|=======|======|=======|========|=======|=======|========|========|
| Benchmark | Output Tokens              |||| Output Words               |||| Output Characters            ||||
| Strategy  | Per Request || Per Second    || Per Request || Per Second    || Per Request  || Per Second     ||
|           | Mdn   | p95  | Mdn   | Mean   | Mdn   | p95  | Mdn   | Mean   | Mdn   | p95   | Mdn    | Mean   |
|-----------|-------|------|-------|--------|-------|------|-------|--------|-------|-------|--------|--------|
| constant  | 50.0  | 50.0 | 50.0  | 51.8   | 40.0  | 45.0 | 40.0  | 39.7   | 200.0 | 257.0 | 197.0  | 204.2  |
| constant  | 50.0  | 50.0 | 250.1 | 288.7  | 41.0  | 47.0 | 203.9 | 224.9  | 206.0 | 281.0 | 1040.9 | 1171.1 |
| constant  | 50.0  | 50.0 | 500.0 | 1468.7 | 41.0  | 47.0 | 403.9 | 1122.5 | 203.0 | 260.0 | 1976.1 | 5790.5 |
|===========|=======|======|=======|========|=======|======|=======|========|=======|=======|========|========|


ℹ Request Token Statistics (Completed Requests)
|===========|======|======|======|======|=======|=======|=======|=======|=========|========|
| Benchmark | Input Tok  || Output Tok || Total Tok    || Stream Iter  || Output Tok      ||
| Strategy  | Per Req    || Per Req    || Per Req      || Per Req      || Per Stream Iter ||
|           | Mdn  | p95  | Mdn  | p95  | Mdn   | p95   | Mdn   | p95   | Mdn     | p95    |
|-----------|------|------|------|------|-------|-------|-------|-------|---------|--------|
| constant  | 50.0 | 50.0 | 50.0 | 50.0 | 100.0 | 100.0 | 102.0 | 104.0 | 1.0     | 1.1    |
| constant  | 50.0 | 50.0 | 50.0 | 50.0 | 100.0 | 100.0 | 102.0 | 104.0 | 1.0     | 1.1    |
| constant  | 50.0 | 50.0 | 50.0 | 50.0 | 100.0 | 100.0 | 104.0 | 104.0 | 1.0     | 1.0    |
|===========|======|======|======|======|=======|=======|=======|=======|=========|========|


ℹ Request Latency Statistics (Completed Requests)
|===========|=========|========|======|======|=====|=====|=====|=====|
| Benchmark | Request Latency || TTFT       || ITL      || TPOT     ||
| Strategy  | Sec             || ms         || ms       || ms       ||
|           | Mdn     | p95    | Mdn  | p95  | Mdn | p95 | Mdn | p95 |
|-----------|---------|--------|------|------|-----|-----|-----|-----|
| constant  | 0.1     | 0.1    | 15.8 | 19.8 | 2.2 | 2.3 | 2.5 | 2.6 |
| constant  | 0.1     | 0.2    | 12.1 | 19.8 | 2.2 | 2.8 | 2.4 | 3.1 |
| constant  | 0.3     | 0.3    | 25.5 | 61.1 | 5.3 | 6.5 | 6.1 | 6.9 |
|===========|=========|========|======|======|=====|=====|=====|=====|


ℹ Server Throughput Statistics
|===========|=====|======|=======|======|=======|========|=======|========|=======|========|
| Benchmark | Requests               |||| Input Tokens  || Output Tokens || Total Tokens  ||
| Strategy  | Per Sec   || Concurrency || Per Sec       || Per Sec       || Per Sec       ||
|           | Mdn | Mean | Mdn   | Mean | Mdn   | Mean   | Mdn   | Mean   | Mdn   | Mean   |
|-----------|-----|------|-------|------|-------|--------|-------|--------|-------|--------|
| constant  | 1.0 | 1.0  | 0.0   | 0.1  | 50.0  | 51.8   | 2.3   | 51.6   | 112.9 | 103.1  |
| constant  | 5.0 | 5.0  | 1.0   | 0.6  | 250.0 | 288.5  | 442.7 | 287.4  | 451.5 | 574.7  |
| constant  | 0.1 | 9.9  | 0.0   | 2.5  | 500.5 | 1440.5 | 448.0 | 1425.1 | 448.0 | 2850.3 |
|===========|=====|======|=======|======|=======|========|=======|========|=======|========|

```

Benchmarking Report (for rate=5):
  ```yaml
  metrics:
    latency:
      inter_token_latency:
        max: 2.8968051988251355
        mean: 2.266296702302526
        min: 2.0669625729930643
        mode: 2.224036625453404
        p0p1: 2.0669625729930643
        p1: 2.0714341377725405
        p10: 2.1386779084497567
        p25: 2.16323015641193
        p5: 2.12330234294035
        p50: 2.224036625453404
        p75: 2.2419082875154457
        p90: 2.7029173714773997
        p95: 2.8220488100635763
        p99: 2.8944112816635443
        p99p9: 2.8968051988251355
        stddev: 0.1888687167084799
        units: ms/token
      request_latency:
        max: 0.16229867935180664
        mean: 0.12442724816751159
        min: 0.1169126033782959
        mode: 0.12063169479370117
        p0p1: 0.1169126033782959
        p1: 0.11745572090148926
        p10: 0.11835432052612305
        p25: 0.119232177734375
        p5: 0.11796379089355469
        p50: 0.12030243873596191
        p75: 0.12152981758117676
        p90: 0.14955878257751465
        p95: 0.15656781196594238
        p99: 0.16179728507995605
        p99p9: 0.16229867935180664
        stddev: 0.01149985833529716
        units: ms
      time_per_output_token:
        max: 3.24312686920166
        mean: 2.4851865576417653
        min: 2.3328113555908203
        mode: 2.3328113555908203
        p0p1: 2.3328113555908203
        p1: 2.3458051681518555
        p10: 2.3644018173217773
        p25: 2.3810243606567383
        p5: 2.3552989959716797
        p50: 2.4030351638793945
        p75: 2.427539825439453
        p90: 2.988924980163574
        p95: 3.1288528442382812
        p99: 3.232693672180176
        p99p9: 3.24312686920166
        stddev: 0.2300663268959325
        units: ms/token
      time_to_first_token:
        max: 23.354291915893555
        mean: 13.210789469264498
        min: 9.773492813110352
        mode: 9.773492813110352
        p0p1: 9.773492813110352
        p1: 10.175943374633789
        p10: 10.506391525268555
        p25: 11.036157608032227
        p5: 10.413408279418945
        p50: 12.093305587768555
        p75: 14.092445373535156
        p90: 18.43094825744629
        p95: 19.814729690551758
        p99: 22.95851707458496
        p99p9: 23.354291915893555
        stddev: 2.977245302854723
        units: ms
    requests:
      failures: 0
      incomplete: 0
      input_length:
        max: 50.0
        mean: 50.0
        min: 50.0
        mode: 50.0
        p0p1: 50.0
        p1: 50.0
        p10: 50.0
        p25: 50.0
        p5: 50.0
        p50: 50.0
        p75: 50.0
        p90: 50.0
        p95: 50.0
        p99: 50.0
        p99p9: 50.0
        stddev: 0.0
        units: count
      output_length:
        max: 50.0
        mean: 50.0
        min: 50.0
        mode: 50.0
        p0p1: 50.0
        p1: 50.0
        p10: 50.0
        p25: 50.0
        p5: 50.0
        p50: 50.0
        p75: 50.0
        p90: 50.0
        p95: 50.0
        p99: 50.0
        p99p9: 50.0
        stddev: 0.0
        units: count
      total: 149
    throughput:
      output_tokens_per_sec: 287.3587596346145
      requests_per_sec: 4.966666666666667
      total_tokens_per_sec: 574.7175192692264
    time:
      duration: 30.0
      start: 1768320690.3882165
      stop: 1768320720.3882165
  scenario:
    load:
      args:
        backend: openai_http
        backend_kwargs: null
        cooldown: null
        data:
        - '{''prompt_tokens'': 50, ''output_tokens'': 50}'
        data_args: []
        data_collator: generative
        data_column_mapper: generative_column_mapper
        data_num_workers: 1
        data_request_formatter: text_completions
        data_sampler: null
        data_samples: -1
        dataloader_kwargs: null
        max_error_rate: null
        max_errors: null
        max_global_error_rate: null
        max_requests: null
        max_seconds: 30
        model: Qwen/Qwen3-0.6B
        output_dir: /requests/guidellm_1768320625_rate_comparison_precise-Qwen3-0.6B
        outputs:
        - results.json
        over_saturation: null
        prefer_response_metrics: true
        processor: null
        processor_args: null
        profile: constant
        rampup: 0.0
        random_seed: 42
        rate:
        - 1.0
        - 5.0
        - 10.0
        sample_requests: 10
        target: http://infra-kv-events-inference-gateway-istio.dpikus-precise.svc.cluster.local:80
        warmup: null
      metadata:
        stage: 1
      name: guidellm
    model:
      name: Qwen/Qwen3-0.6B
  version: '0.1'
  ```