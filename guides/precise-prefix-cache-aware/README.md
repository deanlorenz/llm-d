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

#### Intel XPU deployment

```bash
helmfile apply -e xpu -n ${NAMESPACE} # targets istio as gateway provider with Intel XPU hardware
```

You can also combine Intel XPU hardware with different gateway providers:

```bash
helmfile apply -e xpu-kgateway -n ${NAMESPACE} # targets kgateway as gateway provider with Intel XPU hardware
```

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
gaie-kv-events  llm-d-precise   1               2025-09-25 21:36:52.452999581 +0000 UTC deployed        inferencepool-v1.2.0            v1.2.0
infra-kv-events llm-d-precise   1               2025-09-25 21:36:50.848300265 +0000 UTC deployed        llm-d-infra-v1.3.6              v0.3.0     
ms-kv-events    llm-d-precise   1               2025-09-25 21:36:55.955958022 +0000 UTC deployed        llm-d-modelservice-v0.3.17      v0.3.0 
```

- Out of the box with this example you should have the following resources:

```bash
kubectl get all -n ${NAMESPACE}
NAME                                                          READY   STATUS    RESTARTS   AGE
pod/gaie-kv-events-epp-687b78968b-wvswh                       1/1     Running   0          80s
pod/infra-kv-events-inference-gateway-istio-949d87f84-zvsp2   1/1     Running   0          85s
pod/ms-kv-events-llm-d-modelservice-decode-b874d48d9-bgm5r    1/1     Running   0          75s
pod/ms-kv-events-llm-d-modelservice-decode-b874d48d9-ph64c    1/1     Running   0          75s

NAME                                              TYPE           CLUSTER-IP   EXTERNAL-IP   PORT(S)                        AGE
service/gaie-kv-events-epp                        ClusterIP      10.16.2.44   <none>        9002/TCP,9090/TCP,5557/TCP     81s
service/gaie-kv-events-ip-805c964d                ClusterIP      None         <none>        54321/TCP                      75s
service/infra-kv-events-inference-gateway-istio   ClusterIP      10.16.1.30   10.16.4.2     15021:32033/TCP,80:39332/TCP   86s

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

## Benchmarking

To run benchmarks against the installed llm-d stack, you need [run_only.sh](https://github.com/llm-d/llm-d-benchmark/blob/main/existing_stack/run_only.sh), a template file from [guides/benchmark](../benchmark/), and a Persistent Volume Claim (PVC) to store the results. Follow the instructions in the [benchmark doc](../benchmark/README.md). 

### Example

This example uses [run_only.sh](https://github.com/llm-d/llm-d-benchmark/blob/main/existing_stack/run_only.sh) with the template [precise_guide_template.yaml](../benchmark/precise_guide_template.yaml).

The benchmark launches a pod (`llmdbench-harness-launcher`) that, in this case, uses `inference-perf` with a shared prefix synthetic workload named `shared_prefix_synthetic`. This workload runs several stages with different rates. The results will be stored on the provided PVC, accessible through the `llmdbench-harness-launcher` pod. Each experiment is saved under the `requests` folder, e.g.,/`requests/inference-perf_<experiment ID>_shared_prefix_precise-guide-<model name>` folder. 

Several results files will be created (see [Benchmark doc](../benchmark/README.md)), including a yaml file in a "standard" benchmark report format (see [Benchmark Report](https://github.com/llm-d/llm-d-benchmark/blob/main/docs/benchmark_report.md)).


  ```bash
  curl -L -O https://raw.githubusercontent.com/llm-d/llm-d-benchmark/main/existing_stack/run_only.sh
  chmod u+x run_only.sh
  select f in $(
      curl -s https://api.github.com/repos/llm-d/llm-d/contents/guides/benchmark?ref=main | 
      sed -n '/[[:space:]]*"name":[[:space:]][[:space:]]*"\(precise.*\_template\.yaml\)".*/ s//\1/p'
    ); do 
    curl -LJO "https://raw.githubusercontent.com/llm-d/llm-d/main/guides/benchmark/$f"
    break
  done
  ```
  
Choose the `precise_guide_template.yaml` template, then run:

  ```bash
  export NAMESPACE=llm-d-inference-scheduler     # replace with your namespace
  export BENCHMARK_PVC=workload-pvc   # replace with your PVC name
  export GATEWAY_SVC=infra-inference-scheduling-inference-gateway-istio  # replace with your exact service name
  envsubst < precise_guide_template.yaml > config.yaml
  ```

Edit `config.yaml` if further customization is needed, and then run the command
  ```bash
  ./run_only.sh -c config.yaml
  ```

The output will show the progress of the `inference-perf` benchmark as it runs 
<details>
<summary><b><i>Click</i></b> here to view the expected output</summary>

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

</details>


### Benchmarking Report
  
There is a report for each stage. 
<details>
<summary><b><i>Click</i></b> here to view the report for `rate=10` from the above example</summary>
  
  ```yaml
  metrics:
    latency:
      inter_token_latency:
        max: 0.6592390739824623
        mean: 0.02900245059172652
        min: 6.791960913687944e-06
        p0p1: 3.033865801990032e-05
        p1: 0.015563686798559502
        p10: 0.019321368023520337
        p25: 0.021469394996529445
        p5: 0.018202303716680034
        p50: 0.023767696984577924
        p75: 0.025065310997888446
        p90: 0.027191252261400223
        p95: 0.027449985832208767
        p99: 0.15992223912733625
        p99p9: 0.6424302404634074
        units: s/token
      normalized_time_per_output_token:
        max: 1.130066460411078
        mean: 0.07455394562117204
        min: 0.021596552523126054
        p0p1: 0.021836634834898814
        p1: 0.022878874791001626
        p10: 0.026802186641450707
        p25: 0.028771440401583526
        p5: 0.026354957595733888
        p50: 0.03114238682377349
        p75: 0.03549204396843876
        p90: 0.040007868857358464
        p95: 0.27433672404418163
        p99: 0.9176630568937185
        p99p9: 1.0930528030979383
        units: s/token
      request_latency:
        max: 39.88552725099726
        mean: 31.201138343144265
        min: 21.51016631303355
        p0p1: 21.513322889837728
        p1: 22.639596664793206
        p10: 26.65310029689572
        p25: 28.59833365402301
        p5: 26.180801641245488
        p50: 30.826933004515013
        p75: 33.71675194177078
        p90: 36.34111310068401
        p95: 38.48921600250468
        p99: 39.51040306784445
        p99p9: 39.86624288078066
        units: s
      time_per_output_token:
        max: 0.03929050936899148
        mean: 0.029002450591726524
        min: 0.020733375451993198
        p0p1: 0.02076210723397689
        p1: 0.02197605639129935
        p10: 0.023985525072098245
        p25: 0.025961909717472736
        p5: 0.023094925146602326
        p50: 0.028610963689017808
        p75: 0.031485105915271566
        p90: 0.0345776737117616
        p95: 0.03695713067791075
        p99: 0.0387846155043639
        p99p9: 0.039217750429807116
        units: s/token
      time_to_first_token:
        max: 7.810191813041456
        mean: 2.1675998359240474
        min: 0.061178788950201124
        p0p1: 0.06309354188054567
        p1: 0.07586976393766236
        p10: 0.5658667664858513
        p25: 0.9366489192761946
        p5: 0.541157353669405
        p50: 1.947954576491611
        p75: 3.071206600754522
        p90: 4.036028380162316
        p95: 5.002833548447233
        p99: 7.078293362634135
        p99p9: 7.7845550583496825
        units: s
    requests:
      failures: 0
      input_length:
        max: 7669.0
        mean: 7583.23
        min: 7523.0
        p0p1: 7523.597
        p1: 7526.0
        p10: 7545.0
        p25: 7560.25
        p5: 7536.0
        p50: 7583.5
        p75: 7603.0
        p90: 7622.1
        p95: 7636.05
        p99: 7649.02
        p99p9: 7665.418000000001
        units: count
      output_length:
        max: 1000.0
        mean: 922.125
        min: 33.0
        p0p1: 33.0
        p1: 33.0
        p10: 940.1
        p25: 990.75
        p5: 122.60000000000005
        p50: 997.0
        p75: 1000.0
        p90: 1000.0
        p95: 1000.0
        p99: 1000.0
        p99p9: 1000.0
        units: count
      total: 200
    throughput:
      output_tokens_per_sec: 3502.384763911175
      requests_per_sec: 3.798167020643812
      total_tokens_per_sec: 32304.758859867947
    time:
      duration: 22.336875702952966
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
            num_groups: 150
            num_prompts_per_group: 5
            output_len: 1000
            question_len: 1200
            system_prompt_len: 6000
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
            rate: 15.0
          - concurrency_level: null
            duration: 20
            num_requests: null
            rate: 3.0
          - concurrency_level: null
            duration: 20
            num_requests: null
            rate: 10.0
          - concurrency_level: null
            duration: 20
            num_requests: null
            rate: 15.0
          - concurrency_level: null
            duration: 38
            num_requests: null
            rate: 20.0
          - concurrency_level: null
            duration: 34
            num_requests: null
            rate: 22.0
          - concurrency_level: null
            duration: 30
            num_requests: null
            rate: 25.0
          - concurrency_level: null
            duration: 25
            num_requests: null
            rate: 30.0
          - concurrency_level: null
            duration: 21
            num_requests: null
            rate: 35.0
          - concurrency_level: null
            duration: 38
            num_requests: null
            rate: 40.0
          - concurrency_level: null
            duration: 36
            num_requests: null
            rate: 43.0
          - concurrency_level: null
            duration: 33
            num_requests: null
            rate: 46.0
          - concurrency_level: null
            duration: 30
            num_requests: null
            rate: 49.0
          - concurrency_level: null
            duration: 29
            num_requests: null
            rate: 52.0
          - concurrency_level: null
            duration: 27
            num_requests: null
            rate: 55.0
          - concurrency_level: null
            duration: 26
            num_requests: null
            rate: 57.0
          - concurrency_level: null
            duration: 25
            num_requests: null
            rate: 60.0
          sweep: null
          trace: null
          type: poisson
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
          base_url: http://infra-kv-events-inference-gateway-istio.dpikus-precise.svc.cluster.local:80
          ignore_eos: true
          model_name: Qwen/Qwen3-32B
          type: vllm
        storage:
          google_cloud_storage: null
          local_storage:
            path: /requests/inference-perf_1769545356_Shared_prefix_gateway_precise-guide-Qwen3-32B
            report_file_prefix: null
          simple_storage_service: null
        tokenizer:
          pretrained_model_name_or_path: Qwen/Qwen3-32B
          token: null
          trust_remote_code: null
      metadata:
        stage: 2
      name: inference-perf
    model:
      name: unknown
  version: '0.1'
  ```

</details>

### Comparing LLM-d scheduling to a simple kubernetes service

We examine the overall behavior of the entire workload of the example above, using the `summary_lifecycle_metrics.json` produced by 
`inference-perf`.   
For comparison, we ran the same workload on a k8s service endpoint that directly uses the vLLM pods as backends.

- **Throughput**: Requests/sec 0.8% ; Output tokens/sec 0.5% 
- **Latency**: TTFT 6.0% slower ; E2E request latency 3.8% slower
- **Per-token speed**: Time per output token ‑0.2% (slightly faster)

| Metric                                                           | k8s      | llmd      | Δ (llmd - k8s) | Δ% vs k8s |
|------------------------------------------------------------------|----------|-----------|----------------|-----------|
| Requests/sec                                                     | 5.0788   | 5.1203    | 0.0415         | 0.8%      |
| Input tokens/sec                                                 | 38,491.14| 38,825.59 | 334.45         | 0.9%      |
| Output tokens/sec                                                | 4,773.61 | 4,795.12  | 21.51          | 0.5%      |
| Total tokens/sec                                                 | 43,264.75| 43,620.71 | 355.96         | 0.8%      |
| Approx. gen speed (1/median time_per_output_token) [tok/s]       | 18.870   | 18.913    | 0.0430         | 0.2%      |
| Request latency (s)                                              | 108.052  | 112.138   | 4.087          | 3.8%      |
| TTFT (s)                                                         | 55.996   | 59.362    | 3.366          | 6.0%      |
| Time/output token (ms)                                           | 52.99    | 52.87     | -0.121         | -0.2%     |
| Inter-token latency (ms)                                         | 31.92    | 31.77     | -0.153         | -0.5%     |


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
