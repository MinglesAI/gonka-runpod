# Deploying Inference Nodes on RunPod

## Quick Start

### 1. Use the ready-made RunPod template:
```
https://console.runpod.io/deploy?template=vuf848tt4c&ref=1pxn4obn
```

### 2. Register the node with Network Node:

```bash
curl -X POST http://NETWORK_NODE_HOST:9200/admin/v1/nodes \
     -H "Content-Type: application/json" \
     -d '{
       "id": "node1",
       "host": "NETWORK_NODE_HOST",
       "inference_port": 5000,
       "poc_port": 8080,
       "max_concurrent": 500,
       "models": {
         "Qwen/Qwen3-235B-A22B-Instruct-2507-FP8": {
           "args": [
             "--max-model-len",
             "240000",
             "--tensor-parallel-size",
             "4"
           ]
         }
       }
     }'
```

## Detailed Documentation

For a detailed deployment guide, see [README_RU.md](./README_RU.md) (in Russian).

## Useful Links

- [Official Documentation: Multiple nodes](https://gonka.ai/host/multiple-nodes/)
- [Network Node API](https://gonka.ai/host/network-node-api/)
