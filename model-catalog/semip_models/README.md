# semip_models: job lists for semi-persistence (semi_p)

Each `.jsonl` file here is a list of H200 sampling jobs whose semi_p images (skeletons and weight
sets) are published to a cluster's model bucket, so that later jobs with the same config restore
in seconds instead of cold-starting. The lists are the input of the `semip-publish` workflow
(Jenkins `K8S/neutrino/semip-publish`, or Arctic-Platform's `semip_job` driver run locally), passed
as a GitHub link in its `CONFIGS` parameter.

## Line format

One job per line, exactly as a user submits it for a semi_p job: the model-catalog entry
(`model-catalog/models/<model>/<context>/h200-inference-<full|lora>.json`) on one line, with
`"semi_p": true` added to `inference_config`. Copy a line and submit it to get a restore from the
published image.

```json
{"hardware":"H200","sub_job_configs":[{"job_type":"sampling","model_name":"Qwen/Qwen3-0.6B","dtype":"bfloat16","seed":42,"inference_config":{"max_seq_len":32768,"n_gpus":1,"vllm_config":{"tensor_parallel_size":1,"gpu_memory_utilization":0.8},"semi_p":true}}]}
```

The workflow adds what publishing needs (`SEMIP_DUMP_PUBLISH`, `MODEL_BUCKET`, the image pin, and
`semi_p: false` for the plain-vLLM baseline when tests run). A skeleton matches only the exact
config it was dumped from, on the exact backend image, so a line must not differ from what users
submit.

## Lists

| File | Jobs | Contents |
|---|---|---|
| `jobs_catalog.jsonl` | 55 | Every H200 inference entry of the catalog, as the catalog has it. Some fail: see `jobs_unsupported.jsonl`, and GLM-5.3 long runs out of memory with the catalog's config. |
| `jobs_batch1.jsonl` | 21 | Arctic-Platform#407's batch 1 without the custom jobs: the large and MoE models (DeepSeek, GLM-5.3 standard, GLM-5.3-Flash, GLM-5.3-Flash-BF16, gemma-4-31B, gpt-oss-120b, the Nemotrons, Qwen3-30B-A3B, Qwen3.6-35B-A3B, Qwen3.8-27B). Release backend. |
| `jobs_batch2.jsonl` | 25 | Arctic-Platform#407's batch 2: the small models (gemma-3, medgemma, Qwen3 0.6B to 8B, Qwen3.5). Release backend. |
| `jobs_custom.jsonl` | 5 | Jobs that need a custom backend or config: GLM-5.3 long with cortex-training#271's config (CUDA graphs capped at 64, `gpu_memory_utilization` 0.943, 4096-token steps), and the four Inkling-2-Small-MXFP8 jobs, which need the image with the `inkling2` vLLM fork. |
| `jobs_unsupported.jsonl` | 4 | The `n_gpus=4`, TP=4 configs (GLM-5.3-Flash standard full and LoRA, Nemotron-3-Nano and Nemotron-3.5-Lightning standard). At TP>1 a skeleton is bound to the pod's GPU devices, and a 4-GPU pod gets whichever 4 GPUs are free, so its skeleton would rarely match. They are not published. |
| `jobs_smoke.jsonl` | 2 | Qwen3-0.6B full and LoRA, 1 GPU each: a quick end-to-end test of the workflow. |

`jobs_batch1`, `jobs_batch2` and `jobs_custom` together cover the 51 configs #407 onboarded, each
once; with `jobs_unsupported` they cover `jobs_catalog`. TP=1 jobs on several GPUs (for example
`n_gpus=2` as two TP=1 replicas) are supported; only TP>1 in a pod smaller than a whole node is not.

## Using a list

Pass a GitHub link to the file as `CONFIGS`, preferably a permalink (press `y` on GitHub) so the
run is pinned to an exact version of the list:

```bash
sf jenkins submit --controller https://neeva-001.jenkinsdev1.us-west-2.aws-dev.app.snowflake.com \
  --job K8S/job/neutrino/job/semip-publish \
  --params 'CLUSTER=qa6&IMAGE_TAG=<release_*-h tag>&CONFIGS=<link to a .jsonl here>&MAX_GPUS=64&DRY_RUN=true'
```

`DRY_RUN=true` parses the list and prints the plan without submitting anything. The
`FUNCTIONAL`, `CORRECTNESS` and `PERFORMANCE` checkboxes add restore tests after publishing.

Lists on `snowflakedb/cortex-training` `main` are the official catalog: a dry run there reports
every existing skeleton for each job's config. For any other list (a fork or a branch) it reports
only each job's own exact key on the chosen image.
