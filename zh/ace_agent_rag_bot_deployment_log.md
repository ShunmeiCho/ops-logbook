# ACE Agent rag_bot 示例部署日志

本文档记录部署 NVIDIA ACE Agent `rag_bot` 示例的详细步骤、遇到的问题和解决方案。

## 1. RAG Chain Server 部署

*(在此记录部署 RAG Chain Server 的步骤，包括选择的部署选项（NIM 托管/本地托管）、执行的命令、遇到的问题和解决方法)*

### 1.1 选择部署选项

选择了选项 1：使用 NIM 托管的 `meta/llama3-8b-instruct` LLM 和 embedding 模型端点。

*(例如：选择使用 NIM 托管的 meta/llama3-8b-instruct 和 embedding 模型)*

### 1.2 环境准备

执行了以下命令设置环境变量：
```bash
cd /home/cho/workspace/ACE/microservices/ace_agent/4.1
export DOCKER_VOLUME_DIRECTORY=$(pwd)
# 假设 NVIDIA_API_KEY 已通过 deploy/docker/.env 或环境变量设置
```
确认 `DOCKER_VOLUME_DIRECTORY` 已设置为 `/home/cho/workspace/ACE/microservices/ace_agent/4.1`。

*(记录设置的环境变量，如 DOCKER_VOLUME_DIRECTORY, NVIDIA_API_KEY 等)*

### 1.3 配置文件修改

检查了 `ACE/microservices/ace_agent/4.1/samples/rag_bot/RAG/basic_rag/docker-compose.yaml` 文件。
根据文档要求（使用 NIM 托管 API），对 `chain-server` 服务的环境变量进行了修改：

```diff
-       APP_LLM_SERVERURL: ${APP_LLM_SERVERURL:-"nemollm-inference:8000"}
-       APP_EMBEDDINGS_SERVERURL: ${APP_EMBEDDINGS_SERVERURL:-"nemollm-embedding:8000"}
+       # APP_LLM_SERVERURL: ${APP_LLM_SERVERURL:-"nemollm-inference:8000"} # Comment out default for local NIM
+       APP_LLM_SERVERURL: "" # Explicitly set to empty for NIM Hosted API
+       # APP_EMBEDDINGS_SERVERURL: ${APP_EMBEDDINGS_SERVERURL:-"nemollm-embedding:8000"} # Comment out default for local NIM
+       APP_EMBEDDINGS_SERVERURL: "" # Explicitly set to empty for NIM Hosted API
```

*(记录对 docker-compose.yaml 文件的修改)*

### 1.4 启动 RAG 服务

*   **尝试 1 & 2 (失败):**
    *   命令: `docker compose -f samples/rag_bot/RAG/basic_rag/docker-compose.yaml up -d --build`
    *   问题: 第一次因 RAG compose 文件包含了本地 NIM 的 `include` (`../docker-compose-nim-ms.yaml`) 及其所需但未设置的环境变量 (`MODEL_DIRECTORY` 等) 导致失败。第二次因 `chain-server` 仍然 `depends_on` 未定义的本地 NIM 服务 (`nemollm-embedding`, `nemollm-inference`) 导致失败。
    *   **解决方案:** 修改 `samples/rag_bot/RAG/basic_rag/docker-compose.yaml` 文件，注释掉了对 `docker-compose-nim-ms.yaml` 的 `include`，并注释掉了 `chain-server` 中的 `depends_on` 部分。

*   **尝试 3, 4, 5 & 6 (失败):**
    *   命令: `docker compose -f samples/rag_bot/RAG/basic_rag/docker-compose.yaml up -d --build`
    *   问题: **(环境特定问题)** 持续出现容器命名冲突错误 (`/milvus-etcd`, `/rag-playground`, `/milvus-minio`)，表明用户当前环境中存在来自先前部署尝试的残留容器和网络，需要手动清理。
    *   **解决方案:** 执行了 `down` 命令，并针对特定冲突容器和网络执行了强制清理：
        ```bash
        # 清理 ACE Agent 主要服务
        cd /home/cho/workspace/ACE/microservices/ace_agent/4.1
docker compose -f deploy/docker/docker-compose.yml down
        # 清理 RAG 相关服务
        cd /home/cho/workspace/ACE/microservices/ace_agent/4.1
docker compose -f samples/rag_bot/RAG/basic_rag/docker-compose.yaml down
        # 强制移除冲突容器和网络
        docker rm -f 4c1500383781e3756e9edf8443e1ebce6c7a4cfa02cfc3a25b3e0b786107de58 # milvus-etcd
docker network rm nvidia-rag
        docker rm -f bc81ea32bc3d47c45b9dd07cc3204043037c4f7614f6b50e907e3363939b006b # rag-playground
        docker rm -f f4b022534113 96e7ed819984 f8c2c46d0df5 491494421d26 # milvus-minio, milvus-standalone, chain-server, notebook-server
        ```

*   **尝试 7 (成功):**
    *   命令:
        ```bash
        cd /home/cho/workspace/ACE/microservices/ace_agent/4.1 
docker compose -f samples/rag_bot/RAG/basic_rag/docker-compose.yaml up -d --build
        ```
    *   **结果:** 所有 RAG 相关服务 (`milvus-minio`, `milvus-standalone`, `rag-application-text-chatbot-langchain`, `milvus-etcd`, `rag-playground`) 成功启动。

*(记录执行的 `docker compose up` 命令和结果)*

### 1.5 文档上传 (Ingestion)

访问 RAG 知识库 UI (`http://10.204.222.147:3001/kb`) 进行文档上传。

*   **尝试上传 (多次失败):** 尝试上传各种格式的文件（包括 `.txt`）。
*   **遇到的问题:** 界面始终提示错误 `Error: Failed to upload document. Please upload an unstructured text document.`
*   **日志分析 & 原因定位:**
    1.  最初日志显示 `[401] Unauthorized`，原因是 RAG 的 `docker-compose.yaml` 未能正确加载包含 `NVIDIA_API_KEY` 的主 `.env` 文件。
    2.  修正 `env_file` 路径并重启后，401 错误消失，但上传仍然失败。进一步分析 `chain-server` 日志发现，它默认尝试使用 `snowflake/arctic-embed-l` 作为 embedding 模型，用户的 API Key 可能无权访问此模型。
*   **解决方案:**
    1.  修改 `samples/.../basic_rag/docker-compose.yaml`，在 `chain-server` 服务下添加 `env_file` 指令，指向正确的 `.env` 文件相对路径。
    2.  再次修改该文件，在 `chain-server` 服务的 `environment` 部分明确添加 `APP_EMBEDDINGS_MODELNAME` 环境变量，并指定使用 NVIDIA 的 embedding 模型：
        ```diff
+       APP_EMBEDDINGS_MODELNAME: ${APP_EMBEDDINGS_MODELNAME:-"nvidia/nv-embedqa-e5-v5"} # Explicitly set NVIDIA embedding model
        ```
    3.  停止并重新启动 RAG 服务：
        ```bash
        cd /home/cho/workspace/ACE/microservices/ace_agent/4.1
docker compose -f samples/rag_bot/RAG/basic_rag/docker-compose.yaml down
docker compose -f samples/rag_bot/RAG/basic_rag/docker-compose.yaml up -d --build
        ```
*   **当前状态:** RAG 服务已使用正确的 API Key 和指定的 NVIDIA embedding 模型 (`nvidia/nv-embedqa-e5-v5`) 重新启动。
*   **后续尝试:** *(待用户再次尝试上传)*

*(记录访问知识库 UI 并上传文档的过程)*

## 2. ACE Agent rag_bot 部署

*(在此记录部署 ACE Agent 本身（包括 Riva 模型和核心服务）的步骤)*

### 2.1 环境准备

*(记录设置的环境变量，如 BOT_PATH, NGC_CLI_API_KEY 等)*

### 2.2 部署 Riva 模型

*(记录执行 `docker compose ... up model-utils-speech` 的命令和结果)*

### 2.3 配置 rag_bot

*(记录对 `plugin_config.yaml` 等文件的修改，例如 RAG_SERVER_URL)*

### 2.4 部署 ACE Agent 服务

*(记录执行 `docker compose ... up speech-event-bot` 的命令和结果)*

## 3. 测试与故障排查

*(记录测试 Web UI 交互的过程、遇到的问题及解决方法)*

### 3.1 测试文本/语音交互

### 3.2 问题与解决

*(详细记录遇到的问题、错误日志、分析过程和最终解决方案)*

## 4. 总结

*(对整个部署过程进行总结)* 