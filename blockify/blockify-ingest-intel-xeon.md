# Blockify Ingest on Intel Xeon CPU

This guide runs the Blockify Ingest model on an Intel Xeon Linux host using
vLLM CPU inference.

CPU inference works, but it is slow. For production latency, use an accelerator.

## 1. Install Dependencies

Use a standard x86_64 Linux distro with:

- Docker
- The vLLM CPU container image, tagged as `vllm-cpu-env`

On Ubuntu, the basic packages are:

```bash
sudo apt-get update
sudo apt-get install -y docker.io curl
sudo systemctl enable --now docker
sudo docker --version
```

## 2. Copy the Model

Create a local model directory and copy the Blockify Ingest 8B model weights
onto the host.

```bash
sudo mkdir -p /models/blockify-ingest-8b-v1
# Copy the model files into /models/blockify-ingest-8b-v1
```

The final path should be:

```text
/models/blockify-ingest-8b-v1
```

## 3. Start vLLM

The supported default configuration uses a 16k token context window and a
64 GB CPU KV cache. Use this configuration on a 32 vCPU / 128 GB Xeon host:

```bash
sudo docker run -d --rm \
  --name vllm-cpu \
  --cap-add=sys_nice \
  --net=host \
  --ipc=host \
  --shm-size=4g \
  -v /models:/models \
  -e VLLM_CPU_OMP_THREADS_BIND=0-31 \
  -e VLLM_CPU_KVCACHE_SPACE=64 \
  vllm-cpu-env \
  --model /models/blockify-ingest-8b-v1 \
  --served-model-name blockify-ingest-8b-v1 \
  --trust-remote-code \
  --dtype bfloat16 \
  --max-model-len 16384 \
  --tensor-parallel-size 1 \
  --block-size 128 \
  --port 8000
```

For an 8 vCPU / 32 GB host, use the constrained configuration below. This keeps
the same 16k token context window, but reduces CPU threads and KV cache size:

```bash
sudo docker run -d --rm \
  --name vllm-cpu \
  --cap-add=sys_nice \
  --net=host \
  --ipc=host \
  --shm-size=4g \
  --cpus=8 \
  --memory=32g \
  -v /models:/models \
  -e VLLM_CPU_OMP_THREADS_BIND=0-7 \
  -e VLLM_CPU_KVCACHE_SPACE=16 \
  vllm-cpu-env \
  --model /models/blockify-ingest-8b-v1 \
  --served-model-name blockify-ingest-8b-v1 \
  --trust-remote-code \
  --dtype bfloat16 \
  --max-model-len 16384 \
  --tensor-parallel-size 1 \
  --block-size 128 \
  --port 8000
```

Configuration defaults:

- `--max-model-len 16384` is the supported default context length.
- `VLLM_CPU_KVCACHE_SPACE=64` is the supported default for a 128 GB host.
- `VLLM_CPU_OMP_THREADS_BIND` must match the CPU cores assigned to the
  container.
- `bfloat16` is the tested dtype for this model.

If the container cannot start because of KV cache pressure, reduce
`--max-model-len` to `8192` or increase `VLLM_CPU_KVCACHE_SPACE`.

## 4. Test the API

Watch startup logs:

```bash
sudo docker logs -f vllm-cpu
```

Send an OpenAI-compatible chat request:

```bash
curl http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "blockify-ingest-8b-v1",
    "messages": [
      {
        "role": "user",
        "content": "The capital of France is Paris."
      }
    ],
    "temperature": 0.0,
    "max_tokens": 128,
    "stream": false
  }'
```

Stop the server:

```bash
sudo docker stop vllm-cpu
```
