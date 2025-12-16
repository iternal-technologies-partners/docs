# Blockify Ingest (Gaudi/HPU) Configuration

This reference details the specific Environment Variables and vLLM arguments required to run the Blockify Ingest model on Intel Gaudi accelerators.

## 1. Environment Variables
Ensure these variables are set in the container environment (via ConfigMap or Deployment env).

```bash
# HPU Optimization Flags
export EXPERIMENTAL_WEIGHT_SHARING=0
export PT_HPU_LAZY_MODE=1

# vLLM Scheduling & Bucketing
export VLLM_DECODE_BLOCK_BUCKET_MAX=8320
export VLLM_DECODE_BLOCK_BUCKET_STEP=256
export VLLM_DECODE_BS_BUCKET_STEP=32
export VLLM_DELAYED_SAMPLING=true
export VLLM_GRAPH_PROMPT_RATIO=0.5
export VLLM_GRAPH_RESERVED_MEM=0.04
export VLLM_PROMPT_BS_BUCKET_STEP=32
export VLLM_PROMPT_SEQ_BUCKET_MAX=16640
export VLLM_PROMPT_SEQ_BUCKET_STEP=256
export VLLM_SKIP_WARMUP=false
```

## 2. vLLM Launch Command

```bash
python -m vllm.entrypoints.openai.api_server \
        --model /path/to/mounted/weights \
        --block-size 128 \
        --dtype bfloat16 \
        --tensor-parallel-size 1 \
        --download-dir /hf_cache \
        --max-model-len 16640 \
        --gpu-memory-utilization 0.99 \
        --use-padding-aware-scheduling \
        --max-num-seqs 64 \
        --max-num-prefill-seqs 16 \
        --num-scheduler-steps 1 \
        --disable-log-requests
```

