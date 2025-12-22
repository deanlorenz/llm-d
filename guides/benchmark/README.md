# How to run a benchmark workload against your LLM-d stack

## Overview

This document describes how to run benchmarks against a deployed llm-d stack.  
For full, customizable benchmarking, please refer to [llm-d-benchmark](https://github.com/llm-d/llm-d-benchmark). `llm-d-benchmark` includes advanced features, such as automatic stack creation, sweeping of configuration parameters, recommendations, etc.   

## Requirements

- Install `yq` (YAML processor) - version>=4 (see [install-deps.sh](../guides/prereq/client-setup/install-deps.sh))
- Download the benchmark script [run_only.sh](https://github.com/dmitripikus/llm-d-benchmark/blob/well-lit-path/existing_stack/run_only.sh) and make it executable.
    ```bash
    curl -L -O https://raw.githubusercontent.com/dmitripikus/llm-d-benchmark/refs/tags/well-lit-path/existing_stack/run_only.sh
    chmod u+x run_only.sh
    ```

## Set your stack namespace and benchmark root directory

  ```bash
  export NAMESPACE="<your namespace>"
  export TEMPLATE_DIR=../benchmark
  ```

## Set your stack type and gateway name 

Set your gateway service name and corresponding benchmark template file (available in [guides/benchmark](./)).

<table>
<td>Click to expand:</td>
<td>
<details>
<summary><b>Inference Scheduling</b></summary>

```bash
export GATEWAY_NAME=infra-inference-scheduling-inference-gateway
export BENCHMARK_TEMPLATE=${TEMPLATE_DIR}/inference_scheduling_template.yaml
```

</details>
</td>

<td>
<details>
<summary><b>P/D Disaggregation</b></summary>

```bash
export GATEWAY_NAME=infra-pd-inference-gateway
export BENCHMARK_TEMPLATE=${TEMPLATE_DIR}/pd_template.yaml
```

</details>
</td>

<td>
<details>
<summary><b>Wide EP</b></summary>

```bash
export GATEWAY_NAME=infra-wide-wp-inference-gateway
export BENCHMARK_TEMPLATE=${TEMPLATE_DIR}/wide_ep_template.yaml
```

</details>
</td>
</table>

## Set your gateway service name

This bash snippet finds the service name from the gateway name.

  ```bash
  export GATEWAY_SVC=$(
    kubectl get svc \
    -n ${NAMESPACE} \
    -l gateway.networking.k8s.io/gateway-name=${GATEWAY_NAME} \
    --no-headers  -o=custom-columns=:metadata.name \
  )
  if [[ $(wc -l <<<"${GATEWAY_SVC}") == 1 ]]; then
      echo "using gateway service '${GATEWAY_SVC}' on namespace '${NAMESPACE}'"
  else
      echo "Warning: problems matching a service gateway name '${GATEWAY_NAME}'"
      echo "Please 'run kubectl get svc' and set GATEWAY_SVC manually."
  fi
  ```

## Setup a PVC to save your benchmark results

The benchmark results are stored in a Persistent Volume Claim (mounted by the benchmark launcher pod).  
The PVC must have `RWX` write permissions and be large enough (`200Gi` recommended).  
You need to let the benchmark know which PVC your are using (in this example `workload-pvc`):

  ```bash
  export BENCHMARK_PVC=workload-pvc
  ```

If `workload-pvc` does not exist, create one with `RWX` permissions.

  - <details>
    <summary>Click to expand</summary>

    ```yaml 
    cat <<YAML | kubectl -n ${NAMESPACE} apply -f -
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: ${BENCHMARK_PVC}
    spec:
      accessModes:
        - ReadWriteMany
      resources:
        requests:
          storage: 200Gi
      # storageClassName: <change the default storage class if needed>
    YAML
    ```
    Alternatively, a PVC can be created via the UI of the cluster dashboard.

    </details>


## Create a yaml configuration file for the benchmark

  ```bash
  envsubst < ${BENCHMARK_TEMPLATE} > config.yaml
  ```

## Run

  ```bash
  ./run_only.sh -c config.yaml
  ```

The benchmarks will create a launcher pod to run and the resulted would be stored on the PVC.  
You can try running with different workload configuration. Just edit the `workload` section in `config.yaml` and rerun (for details, see [Advanced.workload](#workload) below).

## Analyze Results

You can access the results PVC through the benchmark launcher pod.
  ```bash
  export HARNESS_POD=$(kubectl get pods -n ${NAMESPACE} -l app --show-labels | awk -v p='lmdbench-.*-launcher' '$0~p {print $1; exit}')
  kubectl exec $HARNESS_POD -n $NAMESPACE -- ls /requests
  ```

To copy a results directory to your local machine use:
  ```bash
  kubectl cp ${NAMESPACE}/${HARNESS_POD}:/requests/<results-folder> <destination-path>
  ```

For example, the `<results-folder>` named `inference-perf_1765442721_shared_prefix_synthetic_inference-scheduling-Qwen3-32B` indicates `inference-perf` was used as harness, the workload was `shared_prefix_synthetic` and the user-defined stack name was `inference-scheduling-Qwen3-32B`. After copying to `<destination-path>` at `/tmp/test`, we can list the results:

  ```ls
  $ ls -lnR /tmp/test
  .:
  total 131248
  drwxr-xr-x 2 1000 1000      4096 Dec 22 21:46 analysis
  -rw-r--r-- 1 1000 1000      5172 Dec 22 21:46 benchmark_report,_stage_0_lifecycle_metrics.json.yaml
  -rw-r--r-- 1 1000 1000      5147 Dec 22 21:46 benchmark_report,_stage_1_lifecycle_metrics.json.yaml
  -rw-r--r-- 1 1000 1000      5152 Dec 22 21:46 benchmark_report,_stage_2_lifecycle_metrics.json.yaml
  -rw-r--r-- 1 1000 1000      5149 Dec 22 21:46 benchmark_report,_stage_3_lifecycle_metrics.json.yaml
  -rw-r--r-- 1 1000 1000      5134 Dec 22 21:46 benchmark_report,_stage_4_lifecycle_metrics.json.yaml
  -rw-r--r-- 1 1000 1000      5152 Dec 22 21:46 benchmark_report,_stage_5_lifecycle_metrics.json.yaml
  -rw-r--r-- 1 1000 1000      5131 Dec 22 21:46 benchmark_report,_stage_6_lifecycle_metrics.json.yaml
  -rw-r--r-- 1 1000 1000      1379 Dec 22 21:46 config.yaml
  -rw-r--r-- 1 1000 1000 134211261 Dec 22 21:46 per_request_lifecycle_metrics.json
  -rw-r--r-- 1 1000 1000       913 Dec 22 21:46 shared_prefix_synthetic.yaml
  -rw-r--r-- 1 1000 1000      4503 Dec 22 21:46 stage_0_lifecycle_metrics.json
  -rw-r--r-- 1 1000 1000      4482 Dec 22 21:46 stage_1_lifecycle_metrics.json
  -rw-r--r-- 1 1000 1000      4484 Dec 22 21:46 stage_2_lifecycle_metrics.json
  -rw-r--r-- 1 1000 1000      4487 Dec 22 21:46 stage_3_lifecycle_metrics.json
  -rw-r--r-- 1 1000 1000      4469 Dec 22 21:46 stage_4_lifecycle_metrics.json
  -rw-r--r-- 1 1000 1000      4488 Dec 22 21:46 stage_5_lifecycle_metrics.json
  -rw-r--r-- 1 1000 1000      4470 Dec 22 21:46 stage_6_lifecycle_metrics.json
  -rw-r--r-- 1 1000 1000     32786 Dec 22 21:46 stderr.log
  -rw-r--r-- 1 1000 1000      5517 Dec 22 21:46 stdout.log
  -rw-r--r-- 1 1000 1000      4372 Dec 22 21:46 summary_lifecycle_metrics.json

  ./analysis:
  total 272
  -rw-r--r-- 1 1000 1000 90845 Dec 22 21:46 latency_vs_qps.png
  -rw-r--r-- 1 1000 1000 92088 Dec 22 21:46 throughput_vs_latency.png
  -rw-r--r-- 1 1000 1000 89975 Dec 22 21:46 throughput_vs_qps.png
  ```

In this example there are 6 workload stages. For each of these stages, there is a results `json` file in harness-specific format and a standardized benchmark `yaml` report in a harness-agnostic format. In this case, the inference-perf benchmark also creates a summary report and a (huge) detailed per-request report. The `analysis` would include plots of the same data.

<details>

<summary>Click for sample contents of <code>/tmp/test/shared_prefix_synthetic.yaml</code></summary>

  ```yaml
  load:
    type: constant
    stages:
      - rate: 2
        duration: 50
      - rate: 5
        duration: 50
      - rate: 8
        duration: 50
      - rate: 10
        duration: 50
      - rate: 12
        duration: 50
      - rate: 15
        duration: 50
      - rate: 20
        duration: 50
  api:
    type: completion
    streaming: true
  server:
    type: vllm
    model_name: Qwen/Qwen3-32B
    base_url: http://infra-inference-scheduling-inference-gateway.dpikus-ns.svc.cluster.local:80
    ignore_eos: true
  tokenizer:
    pretrained_model_name_or_path: Qwen/Qwen3-32B
  data:
    type: shared_prefix
    shared_prefix:
      num_groups: 32
      num_prompts_per_group: 32
      system_prompt_len: 2048
      question_len: 256
      output_len: 256
  report:
    request_lifecycle:
      summary: true
      per_stage: true
      per_request: true
  storage:
    local_storage:
      path: /requests/inference-perf_1765442721_shared_prefix_synthetic_inference-scheduling-Qwen3-32B
  ```

</details>
<details>

<summary>Click for sample contents of <code>/tmp/test/summary_lifecycle_metrics.json</code></summary>

  ```json
  $ cat /tmp/test/summary_lifecycle_metrics.json
  {
    "load_summary": {
      "count": 3600,
      "schedule_delay": {
        "mean": 0.0005517881022468726,
        "min": -0.0009677917696535587,
        "p0.1": -0.0009261268951522652,
        "p1": -0.0006993622815934941,
        "p5": -0.00036710660060634837,
        "p10": -0.0001909452490508556,
        "p25": 0.0001798685480025597,
        "median": 0.0005617527785943821,
        "p75": 0.0009258257632609457,
        "p90": 0.0012554632412502542,
        "p95": 0.0014667386640212496,
        "p99": 0.001798744505795184,
        "p99.9": 0.002180055778066162,
        "max": 0.0024819674144964665
      }
    },
    "successes": {
      "count": 3600,
      "latency": {
        "request_latency": {
          "mean": 5.018124350203054,
          "min": 3.8039849390042946,
          "p0.1": 3.8743090889458545,
          "p1": 3.953152789860906,
          "p5": 4.157410570966022,
          "p10": 4.314808080092189,
          "p25": 4.572383339735097,
          "median": 4.921685875495314,
          "p75": 5.447979573007615,
          "p90": 5.874509802411194,
          "p95": 6.23043658035167,
          "p99": 6.898871109607862,
          "p99.9": 7.155581975026522,
          "max": 7.1762416810088325
        },
        "normalized_time_per_output_token": {
          "mean": 0.03036492437930885,
          "min": 0.007583161046942186,
          "p0.1": 0.010184849169221934,
          "p1": 0.015474372415765174,
          "p5": 0.01634922173616652,
          "p10": 0.01703432744081125,
          "p25": 0.018072373271508013,
          "median": 0.019470786924703526,
          "p75": 0.02165297018970986,
          "p90": 0.023846092055134705,
          "p95": 0.0264264020348377,
          "p99": 0.4516598727074727,
          "p99.9": 0.5963934792099037,
          "max": 1.7235658823337872
        },
        "time_per_output_token": {
          "mean": 0.009651459220661362,
          "min": 0.007256438481493456,
          "p0.1": 0.007323875762569309,
          "p1": 0.007555858870462394,
          "p5": 0.007952370352942776,
          "p10": 0.008277119267264429,
          "p25": 0.008787224616948537,
          "median": 0.009469471096488253,
          "p75": 0.010491207710992169,
          "p90": 0.01131961066218535,
          "p95": 0.012025573761348197,
          "p99": 0.013288400162375152,
          "p99.9": 0.013780982321925176,
          "max": 0.013837818343071613
        },
        "time_to_first_token": {
          "mean": 0.055621399055906094,
          "min": 0.03300976799800992,
          "p0.1": 0.03474102120747557,
          "p1": 0.03800362152425805,
          "p5": 0.04069916713197017,
          "p10": 0.04262162119266577,
          "p25": 0.0469227115099784,
          "median": 0.05276561250502709,
          "p75": 0.05953402600425761,
          "p90": 0.06766136341320816,
          "p95": 0.0781531358021311,
          "p99": 0.1207716233390948,
          "p99.9": 0.1945496345278814,
          "max": 0.28962455500732176
        },
        "inter_token_latency": {
          "mean": 0.009651459220661362,
          "min": 1.1920055840164423e-06,
          "p0.1": 1.969980075955391e-06,
          "p1": 4.66001802124083e-06,
          "p5": 5.8680016081780195e-06,
          "p10": 7.030001142993569e-06,
          "p25": 1.3053999282419682e-05,
          "median": 4.527249257080257e-05,
          "p75": 0.018683608002902474,
          "p90": 0.021172565198503433,
          "p95": 0.023382977170695075,
          "p99": 0.031130830741021784,
          "p99.9": 0.05123655588901624,
          "max": 0.164861869008746
        }
      },
      "throughput": {
        "input_tokens_per_sec": 22133.556296544484,
        "output_tokens_per_sec": 2253.085197994428,
        "total_tokens_per_sec": 24386.64149453891,
        "requests_per_sec": 9.129153381242592
      },
      "prompt_len": {
        "mean": 2424.491666666667,
        "min": 2387.0,
        "p0.1": 2387.0,
        "p1": 2390.0,
        "p5": 2399.0,
        "p10": 2403.0,
        "p25": 2413.0,
        "median": 2426.0,
        "p75": 2435.0,
        "p90": 2443.0,
        "p95": 2450.0,
        "p99": 2468.0,
        "p99.9": 2474.0,
        "max": 2474.0
      },
      "output_len": {
        "mean": 246.8011111111111,
        "min": 3.0,
        "p0.1": 8.599,
        "p1": 11.0,
        "p5": 238.0,
        "p10": 248.0,
        "p25": 253.0,
        "median": 255.0,
        "p75": 256.0,
        "p90": 256.0,
        "p95": 256.0,
        "p99": 257.0,
        "p99.9": 511.0,
        "max": 511.0
      }
    },
    "failures": {
      "count": 0,
      "request_latency": null,
      "prompt_len": null
    }
  }  
  ```

</details>

---

# Advanced

## Customizing the config file

This section describes the details of the configuration `config.yaml` file. You may edit it as needed to match your stack (e.g., to change the model name). If you followed the guideline to create your stack then you should be able to run without any modification.   
**Do not edit** unless you know what you are doing.

The configuration is divided into sections, each with a different scope.

### Endpoint

These are the properties of the stack (`envsubst` would replace `NAMESPACE` and `GATEWAY_SVC` to match your env). Gated models need a Hugging Face token to access. Your stack should already have a token secret under the name `llm-d-hf-token`. `stack_name` is a user-defined arbitrary name that will be attached to the benchmark results. You can use `stack_name` to help you identify the results of different experiments. The `model` must match your stack. Please note the `yaml` tags -- other section of this `yaml` reference them (e.g., the tokenizer reference the model).   
  ```yaml
  endpoint:
    stack_name: &stack_name inference-scheduling-Qwen3-0.6B  # user defined name for the stack (results prefix)
    model: &model Qwen/Qwen3-0.6B                      # Exact HuggingFace model name. Must match stack deployed.
    namespace: &namespace $NAMESPACE
    base_url: &url http://${GATEWAY_SVC}.${NAMESPACE}.svc.cluster.local:80  # Base URL of inference endpoint
    hf_token_secret: llm-d-hf-token   # The name of secret that contains the HF token of the stack
  ```

### Control

These define the local target directory for temporary files and for fetching results.
The `kubectl` entru allows you to change the k8s control command (e.g., to `oc`).

  ```yaml
  control:
    work_dir: $HOME/llm-d-bench-work  # working directory to store temporary and autogenerated files. 
                                      # Do not edit content manually.
                                      # If not set, a temp directory will be created.
    kubectl: kubectl                  # kubectl command: kubectl or oc                                   
  ```

### Harness

Harness refers to the specific benchmarking tool used. Several harnesses are supported, including [inference-perf](https://github.com/kubernetes-sigs/inference-perf), [guidellm](https://github.com/vllm-project/guidellm), [InferenceMAX](https://github.com/InferenceMAX/InferenceMAX) and [vLLM Benchmarks](https://github.com/vllm-project/vllm/tree/main/benchmarks). The `results_pvc` should be set to the PVC you created above. The benchmark is run from one or more pods inside the cluster. The image for this pod is from [llm-d-benchmark](https://github.com/llm-d/llm-d-benchmark). Typically, you do not have to change the `namespace` or the `image` 

  ```yaml
  harness:
    name: &harness_name inference-perf
    results_pvc: ${BENCHMARK_PVC}   # PVC where benchmark results are stored
    namespace: *namespace           # Namespace where harness is deployed. Typically with stack.
    parallelism: 1                  # Number of parallel workload launcher pods to create.  
    wait_timeout: 600               # Time (in seconds) to wait for workload launcher pod to complete before terminating.
                                    # Set to 0 to disable timeout.
    image: ghcr.io/llm-d/llm-d-benchmark:v0.4.0
    # dataset_url: https://huggingface.co/datasets/anon8231489123/ShareGPT_Vicuna_unfiltered/resolve/main/ShareGPT_V3_unfiltered_cleaned_split.json
  ```

### Extra environment variables

This sections allows you to add arbitrary environment variable to the harness pod. This is mostly useful to change behavior for a specific harness. For example, limit the number of threads the Rust Rayon thread pool should use in inference-perf harness.

  ```yaml
  env:
    - name: RAYON_NUM_THREADS
      value: "4"
  ```

### Workload 

These settings characterize of the workload used to benchmark the stack. Each harness supports different configuration parameters for setting the workload. These are described in detail in their documentations (see, e.g., [inference-perf configuration guide](https://github.com/kubernetes-sigs/inference-perf/blob/main/docs/config.md)).
While the details are different for each harness, the concepts are similar.  
A workload specification typically includes:
 - **Data specification**: How to generated the contents of the inference queries. For example, the distribution of input and output lengths or a path to a HF trace.   
 - **Load specification**: Timing for sending queries. E.g., rate and duration. Some harnesses support "stages", each with its own load specification.
 - **Control**: Which API to use, target endpoint, tokenizers, etc.
 - **Output**: The types of reports to produce and where to store them. **Do not change** -- the benchmark tools will set these automatically. 

Several workload can be specified, each with a different name. The benchmark would run all the workloads against the stack. 

  ```yaml
  workload:                         # yaml configuration for harness workload(s)
    
    # an example workload using random synthetic data
    sanity_random:
      load:
        type: constant
        stages:
        - rate: 1
          duration: 30
      api:
        type: completion
        streaming: true
      server:
        type: vllm
        model_name: *model
        base_url: *url
        ignore_eos: true
      tokenizer:
        pretrained_model_name_or_path: *model
      data:
        type: random
        input_distribution:
          min: 10             # min length of the synthetic prompts
          max: 100            # max length of the synthetic prompts
          mean: 50            # mean length of the synthetic prompts
          std: 10             # standard deviation of the length of the synthetic prompts
          total_count: 100    # total number of prompts to generate to fit the above mentioned distribution constraints
        output_distribution:
          min: 10             # min length of the output to be generated
          max: 100            # max length of the output to be generated
          mean: 50            # mean length of the output to be generated
          std: 10             # standard deviation of the length of the output to be generated
          total_count: 100    # total number of output lengths to generate to fit the above mentioned distribution constraints
        # path: /workload/ShareGPT_V3_unfiltered_cleaned_split.json   # file name should match dataset_url above
      report:
        request_lifecycle:
          summary: true
          per_stage: true
          per_request: true
      storage:
        local_storage:
          path: /workspace

    # an example workload using shared prefix synthetic data
    shared_prefix_synthetic:
      load:
        type: constant
        stages:
        - rate: 2
          duration: 40
        - rate: 5
          duration: 50
        - rate: 8
          duration: 60
      api:
        type: completion
        streaming: true
      server:
        type: vllm
        model_name: *model
        base_url: *url
        ignore_eos: true
      tokenizer:
        pretrained_model_name_or_path: *model
      data:
        type: shared_prefix
        shared_prefix:
          num_groups: 32                # Number of distinct shared prefixes
          num_prompts_per_group: 32     # Number of unique questions per shared prefix
          system_prompt_len: 2048       # Length of the shared prefix (in tokens)
          question_len: 256             # Length of the unique question part (in tokens)
          output_len: 256               # Target length for the model's generated output (in tokens)
      report:
        request_lifecycle:
          summary: true
          per_stage: true
          per_request: true
      storage:
        local_storage:
          path: /workspace
  ```
