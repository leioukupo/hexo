---
abbrlink: vllm报错
categories: []
date: '2025-10-20T21:05:19.753540+08:00'
tags:
- vllm
title: Pascal-vllm报错
updated: '2025-10-20T21:05:21.363+08:00'
---
Pascal-vllm原版镜像报错aimv2被占用

通过挂载/usr/local/lib/python3.12/dist-packages/vllm/transformers\_utils/configs/ovis.py

进行修改

分别是

```python
31:    model_type: str = "aimv3"
76:    AutoConfig.register("aimv3", AIMv2Config)
# 原始是aimv2  修改成其他名字即可，两处名字要一样
# 最好指定一下--swap-space 0.5          太大也运行不了
```

参考的docker compose.yaml

```yaml
networks:
    1panel-network:
        external: true
services:
    vllm:
        command: --model ${MODEL}
        container_name: ${CONTAINER_NAME}
        deploy:
            resources:
                limits:
                    cpus: ${CPUS}
                    memory: ${MEMORY_LIMIT}
                reservations:
                    devices:
                        - capabilities:
                            - gpu
                          count: all
                          driver: nvidia
        environment:
            HF_ENDPOINT: https://hf-mirror.com
            HUGGING_FACE_HUB_TOKEN: ${HUGGING_FACE_HUB_TOKEN}
        image: ghcr.io/sasha0552/vllm:v0.9.1
        ipc: host
        networks:
            - 1panel-network
        ports:
            - ${HOST_IP}:${PANEL_APP_PORT_HTTP}:8000
        restart: on-failure:5
        runtime: nvidia
        volumes:
            - ./cache/huggingface:/root/.cache/huggingface
            -  /path/to/ovis.py://usr/local/lib/python3.12/dist-packages/vllm/transformers_utils/configs/ovis.py
```
