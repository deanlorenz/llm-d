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

For example, if `<destination-path>` is `/tmp/test` and `<results-folder>` is `inference-perf_1765274928_sanity_random_inference-scheduling-Qwen3-32B` then after copy we can list the results:

```bash
$ ls -ltn /tmp/test/
total 136
-rw-r--r-- 1 1000 1000 92999 Dec 22 20:31 per_request_lifecycle_metrics.json
-rw-r--r-- 1 1000 1000   745 Dec 22 20:31 sanity_random.yaml
-rw-r--r-- 1 1000 1000  4474 Dec 22 20:31 stage_0_lifecycle_metrics.json
-rw-r--r-- 1 1000 1000  4367 Dec 22 20:31 summary_lifecycle_metrics.json
-rw-r--r-- 1 1000 1000  1208 Dec 22 20:31 config.yaml
-rw-r--r-- 1 1000 1000  2083 Dec 22 20:31 stderr.log
-rw-r--r-- 1 1000 1000  2834 Dec 22 20:31 stdout.log
-rw-r--r-- 1 1000 1000  4934 Dec 22 20:31 benchmark_report,_stage_0_lifecycle_metrics.json.yaml
drwxr-xr-x 2 1000 1000  4096 Dec 22 20:31 analysis
```

<details>

<summary>Click for sample contents of <code>/tmp/test/summary_lifecycle_metrics.json</code></summary>

  ```json
  $ cat /tmp/test/summary_lifecycle_metrics.json
  {
    "load_summary": {
      "count": 30,
      "schedule_delay": {
        "mean": 0.0009949469116691035,
        "min": 5.5570453696418554e-05,
        "p0.1": 6.324866517388727e-05,
        "p1": 0.00013235256847110577,
        "p5": 0.00032730887578509283,
        "p10": 0.0003592936380300671,
        "p25": 0.0006020650107529946,
        "median": 0.000765682605560869,
        "p75": 0.0011245777877775254,
        "p90": 0.0014910372890881262,
        "p95": 0.0016743282074457957,
        "p99": 0.004784760244001521,
        "p99.9": 0.005873430990723152,
        "max": 0.005994394407025538
      }
    },
    "successes": {
      "count": 30,
      "latency": {
        "request_latency": {
          "mean": 0.6588773118998991,
          "min": 0.1586868219965254,
          "p0.1": 0.15869474296964473,
          "p1": 0.15876603172771867,
          "p5": 0.1589744984987192,
          "p10": 0.1592453661025502,
          "p25": 0.16611412924794422,
          "median": 0.43602975699832314,
          "p75": 1.3402625747512502,
          "p90": 1.4376812317983423,
          "p95": 1.4399501692012564,
          "p99": 1.4428512091104495,
          "p99.9": 1.4437475199112304,
          "max": 1.4438471100002062
        },
        "normalized_time_per_output_token": {
          "mean": 0.01674332175508386,
          "min": 0.014335748899975442,
          "p0.1": 0.014336704613826077,
          "p1": 0.014345306038481795,
          "p5": 0.014371501398989494,
          "p10": 0.014435703177000422,
          "p25": 0.014619622932853567,
          "median": 0.015881223829154016,
          "p75": 0.016693644060641027,
          "p90": 0.020294773005207392,
          "p95": 0.02274349231242922,
          "p99": 0.025181103562708813,
          "p99.9": 0.02557737823157641,
          "max": 0.02562140875033947
        },
        "time_per_output_token": {
          "mean": 0.006559230440518787,
          "min": 0.0060569573335149994,
          "p0.1": 0.006057008771227889,
          "p1": 0.006057471710643898,
          "p5": 0.006063760290531458,
          "p10": 0.006083691500021987,
          "p25": 0.006124783118918588,
          "median": 0.006748013784410417,
          "p75": 0.00695824464904582,
          "p90": 0.006978150976107795,
          "p95": 0.006985430946021656,
          "p99": 0.007002392395553935,
          "p99.9": 0.007008618324614845,
          "max": 0.007009310094510501
        },
        "time_to_first_token": {
          "mean": 0.03326361546690653,
          "min": 0.02732928300247295,
          "p0.1": 0.02734817798135191,
          "p1": 0.027518232791262563,
          "p5": 0.02798178529847064,
          "p10": 0.028100511400407414,
          "p25": 0.02906892950340989,
          "median": 0.03012613499959116,
          "p75": 0.03321451375086326,
          "p90": 0.03665650219700184,
          "p95": 0.0515788085005624,
          "p99": 0.06579398802816287,
          "p99.9": 0.06834677500107504,
          "max": 0.06863041799806524
        },
        "inter_token_latency": {
          "mean": 0.006865140741710124,
          "min": 1.5460027498193085e-06,
          "p0.1": 1.661555506871082e-06,
          "p1": 2.1122003818163647e-06,
          "p5": 5.4873464250704275e-06,
          "p10": 7.376601570285857e-06,
          "p25": 1.510375295765698e-05,
          "median": 4.050750067108311e-05,
          "p75": 0.01413311349824653,
          "p90": 0.014454104701144388,
          "p95": 0.014726091798729612,
          "p99": 0.016374180317507123,
          "p99.9": 0.020914695926352915,
          "max": 0.039189089999126736
        }
      },
      "throughput": {
        "input_tokens_per_sec": 49.76859460193433,
        "output_tokens_per_sec": 47.065738269461434,
        "total_tokens_per_sec": 96.83433287139577,
        "requests_per_sec": 1.08114253298916
      },
      "prompt_len": {
        "mean": 46.03333333333333,
        "min": 10.0,
        "p0.1": 10.0,
        "p1": 10.0,
        "p5": 10.0,
        "p10": 10.0,
        "p25": 10.0,
        "median": 10.0,
        "p75": 108.0,
        "p90": 108.4,
        "p95": 112.0,
        "p99": 112.0,
        "p99.9": 112.0,
        "max": 112.0
      },
      "output_len": {
        "mean": 43.53333333333333,
        "min": 7.0,
        "p0.1": 7.029,
        "p1": 7.29,
        "p5": 8.0,
        "p10": 8.0,
        "p25": 10.0,
        "median": 27.0,
        "p75": 85.5,
        "p90": 99.1,
        "p95": 100.0,
        "p99": 100.0,
        "p99.9": 100.0,
        "max": 100.0
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
