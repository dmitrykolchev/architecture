### Check Nvidia runtime 
```
sudo docker run --rm --runtime=nvidia --gpus all ubuntu nvidia-smi
```
Sample output:

```
Mon Feb  9 21:56:16 2026
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 580.102.01             Driver Version: 581.57         CUDA Version: 13.0     |
+-----------------------------------------+------------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id          Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |           Memory-Usage | GPU-Util  Compute M. |
|                                         |                        |               MIG M. |
|=========================================+========================+======================|
|   0  NVIDIA GeForce RTX 4070        On  |   00000000:01:00.0  On |                  N/A |
|  0%   37C    P8             11W /  200W |    2111MiB /  12282MiB |      0%      Default |
|                                         |                        |                  N/A |
+-----------------------------------------+------------------------+----------------------+

+-----------------------------------------------------------------------------------------+
| Processes:                                                                              |
|  GPU   GI   CI              PID   Type   Process name                        GPU Memory |
|        ID   ID                                                               Usage      |
|=========================================================================================|
|  No running processes found                                                             |
+-----------------------------------------------------------------------------------------+
```

### Install vLLM & Qwen/Qwen2.5-Coder-7B-Instruct-GPTQ-Int4 for RTX4070
```
docker run -d \
  --name vllm-server \
  --runtime nvidia \
  --gpus all \
  -v /mnt/e/.cache/huggingface:/root/.cache/huggingface \
  -p 8000:8000 \
  --ipc=host \
  vllm/vllm-openai:latest \
  --model Qwen/Qwen2.5-Coder-7B-Instruct-GPTQ-Int4 \
  --dtype auto \
  --api-key secret-token-123 \
  --gpu-memory-utilization 0.85 \
  --max-model-len 8192 \
  --kv-cache-dtype fp8

```
to view log
```
docker logs -f vllm-server
```

wait for
```
INFO 02-09 15:19:00 [api_server.py:946] Starting vLLM API server 0 on http://0.0.0.0:8000
```

now you can run to test vLLM

```
curl http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer secret-token-123" \
  -d '{
    "model": "Qwen/Qwen2.5-Coder-7B-Instruct-GPTQ-Int4",
    "messages": [
      {"role": "system", "content": "Ты дружелюбный помощник."},
      {"role": "user", "content": "Как ты думаешь, что лучше использовать для обучения модели с нуля -  компьютер с RTX PRO 6000 Workstation 96GB vRAM или Mac Studio M3 Ultra c 512GB RAM?"}
    ],
    "temperature": 0
  }'
```
