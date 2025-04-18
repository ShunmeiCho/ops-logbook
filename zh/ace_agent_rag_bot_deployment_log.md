# ACE Agent rag_bot 示例部署日志

本文档详细记录了 NVIDIA ACE Agent `rag_bot` 示例的完整部署过程，包括环境配置、服务启动、问题排查及解决方案。

## 1. RAG Chain Server 部署

### 1.1 部署方案选择

经评估，选择了**方案一**：使用 NVIDIA NIM 托管的 `meta/llama3-8b-instruct` LLM 和 embedding 模型端点。此方案无需本地部署大型模型，降低了硬件要求，但需要依赖 NVIDIA API 服务。

### 1.2 环境准备

环境变量配置：
```bash
cd /home/cho/workspace/ACE/microservices/ace_agent/4.1
export DOCKER_VOLUME_DIRECTORY=$(pwd)
# NVIDIA_API_KEY 通过 deploy/docker/.env 或环境变量设置
```

验证 `DOCKER_VOLUME_DIRECTORY` 已正确设置为 `/home/cho/workspace/ACE/microservices/ace_agent/4.1`。

### 1.3 配置文件修改

针对 NIM 托管模型要求，对 `samples/rag_bot/RAG/basic_rag/docker-compose.yaml` 中的 `chain-server` 服务环境变量进行了以下修改：

```diff
-       APP_LLM_SERVERURL: ${APP_LLM_SERVERURL:-"nemollm-inference:8000"}
-       APP_EMBEDDINGS_SERVERURL: ${APP_EMBEDDINGS_SERVERURL:-"nemollm-embedding:8000"}
+       # APP_LLM_SERVERURL: ${APP_LLM_SERVERURL:-"nemollm-inference:8000"} # 注释本地 NIM 默认配置
+       APP_LLM_SERVERURL: "" # 显式设置为空值以使用 NIM 托管 API
+       # APP_EMBEDDINGS_SERVERURL: ${APP_EMBEDDINGS_SERVERURL:-"nemollm-embedding:8000"} # 注释本地 NIM 默认配置
+       APP_EMBEDDINGS_SERVERURL: "" # 显式设置为空值以使用 NIM 托管 API
```

### 1.4 RAG 服务启动过程

#### 1.4.1 初始尝试（尝试 1-2，失败）

* **命令**：`docker compose -f samples/rag_bot/RAG/basic_rag/docker-compose.yaml up -d --build`
* **问题**：
  * 第一次失败原因：RAG compose 文件包含了对本地 NIM 的 `include` (`../docker-compose-nim-ms.yaml`) 引用，但未设置相关环境变量（如 `MODEL_DIRECTORY`）。
  * 第二次失败原因：`chain-server` 服务仍然 `depends_on` 未定义的本地 NIM 服务。
* **解决方案**：修改 `docker-compose.yaml` 文件，注释掉对 `docker-compose-nim-ms.yaml` 的引用和 `depends_on` 部分。

#### 1.4.2 持续尝试（尝试 3-6，失败）

* **命令**：`docker compose -f samples/rag_bot/RAG/basic_rag/docker-compose.yaml up -d --build`
* **问题**：（环境特定）出现容器命名冲突，表明环境中存在先前部署的残留容器和网络。
* **解决方案**：执行全面清理：
  ```bash
  # 清理 ACE Agent 服务
  cd /home/cho/workspace/ACE/microservices/ace_agent/4.1
  docker compose -f deploy/docker/docker-compose.yml down
  
  # 清理 RAG 服务
  docker compose -f samples/rag_bot/RAG/basic_rag/docker-compose.yaml down
  
  # 强制移除冲突容器和网络
  docker rm -f 4c1500383781e3756e9edf8443e1ebce6c7a4cfa02cfc3a25b3e0b786107de58 # milvus-etcd
  docker network rm nvidia-rag
  docker rm -f bc81ea32bc3d47c45b9dd07cc3204043037c4f7614f6b50e907e3363939b006b # rag-playground
  docker rm -f f4b022534113 96e7ed819984 f8c2c46d0df5 491494421d26 # 其他残留容器
  ```

#### 1.4.3 最终成功部署（尝试 7）

* **命令**：
  ```bash
  cd /home/cho/workspace/ACE/microservices/ace_agent/4.1
  docker compose -f samples/rag_bot/RAG/basic_rag/docker-compose.yaml up -d --build
  ```
* **结果**：所有 RAG 相关服务（`milvus-minio`、`milvus-standalone`、`rag-application-text-chatbot-langchain`、`milvus-etcd`、`rag-playground`) 成功启动。

### 1.5 文档上传与知识库构建

#### 1.5.1 初始上传尝试

* **访问界面**：RAG 知识库 UI (`http://10.204.222.147:3001/kb`)
* **问题**：多次尝试上传各种格式文件（`.txt`、`.pdf`）均失败，界面显示错误：`Error: Failed to upload document. Please upload an unstructured text document.`

#### 1.5.2 系统性故障排查

通过分层分析定位了根本原因：

1. **日志分析**：初始日志显示 `[401] Unauthorized` 错误
2. **配置检查**：发现 RAG 的 `docker-compose.yaml` 未能正确加载包含 `NVIDIA_API_KEY` 的 `.env` 文件
3. **配置调整**：修正 `env_file` 路径并重启后，401 错误消失，但上传仍失败
4. **深入分析**：`chain-server` 日志显示其默认使用 `snowflake/arctic-embed-l` 作为 embedding 模型
5. **模型指定**：修改 `docker-compose.yaml`，明确设置 `APP_EMBEDDINGS_MODELNAME` 为 `nvidia/nv-embedqa-e5-v5`
6. **持续调试**：上传仍失败，日志再次显示 `[401] Unauthorized`
7. **容器内部检查**：
   ```bash
   docker exec -it <chain-server容器ID> /bin/bash
   echo $NVIDIA_API_KEY # 输出为空 ""
   echo $APP_EMBEDDINGS_MODELNAME # 输出为 "nvidia/nv-embedqa-e5-v5"
   ```

#### 1.5.3 根本原因与最终解决方案

**根本原因**：`NVIDIA_API_KEY` 未能成功注入到 `chain-server` 容器的运行时环境中。

**最终解决方案**：
1. 确保 `NVIDIA_API_KEY` 在主机 Shell 中通过 `export` 正确设置
2. 停止 RAG 服务
3. 使用同时强制传递两个关键环境变量的方式启动 RAG 服务：
   ```bash
   cd /home/cho/workspace/ACE/microservices/ace_agent/4.1
   # 确保主机 Shell 中 API Key 已导出: export NVIDIA_API_KEY="<您的 Key>"
   NVIDIA_API_KEY=$NVIDIA_API_KEY APP_EMBEDDINGS_MODELNAME="nvidia/nv-embedqa-e5-v5" docker compose -f samples/rag_bot/RAG/basic_rag/docker-compose.yaml up -d --build
   ```

**验证结果**：使用上述命令重启服务后，成功上传 `test.txt` 文件，知识库文档上传功能已验证可用。

## 2. ACE Agent rag_bot 部署

### 2.1 环境准备

在 `/home/cho/workspace/ACE/microservices/ace_agent/4.1` 目录下执行：
```bash
export BOT_PATH=./samples/rag_bot/
source deploy/docker/docker_init.sh
```

### 2.2 Riva 语音模型部署

执行部署 Riva ASR 和 TTS 模型的命令：
```bash
docker compose -f deploy/docker/docker-compose.yml up model-utils-speech
```

**遇到的问题**：
* 初次执行出现错误：`unable to get image '/model-utils:': Error response from daemon: invalid reference format`
* 伴随大量警告显示环境变量未设置

**原因分析**：
环境变量设置仅在命令执行期间有效，未持续到后续的 Shell 会话中，导致 Docker 尝试拉取的镜像路径格式无效。

**解决方案**：
在同一个 Shell 会话中按顺序执行以下步骤：
```bash
cd /home/cho/workspace/ACE/microservices/ace_agent/4.1
export BOT_PATH=./samples/rag_bot/
source deploy/docker/docker_init.sh
```

确认关键环境变量已正确设置：
```
DOCKER_REGISTRY=nvcr.io/nvidia/ace
TAG=4.1.0
```

重新启动 Riva 模型部署：
```bash
docker compose -f deploy/docker/docker-compose.yml up model-utils-speech
```

**结果**：Riva 语音服务器 `riva-speech-server` (基于 `nvcr.io/nvidia/riva/riva-speech:2.17.0` 镜像) 成功部署。

> **注意**：此过程需要下载模型，可能耗时较长。命令在前台执行直至完成。

### 2.3 rag_bot 配置调整

检查并修改了 `ACE/microservices/ace_agent/4.1/samples/rag_bot/plugin_config.yaml` 文件，更新 `RAG_SERVER_URL`：
```diff
-       RAG_SERVER_URL: "http://localhost:8081"
+       RAG_SERVER_URL: "http://rag-application-text-chatbot-langchain:8081"
```

### 2.4 ACE Agent 核心服务部署

**首次尝试（失败）**：
直接执行 `docker compose -f deploy/docker/docker-compose.yml up speech-event-bot -d` 会遇到与部署 Riva 模型相同的问题：镜像引用格式错误。

**成功的部署方法**：
在同一个 Shell 会话中确保环境变量正确设置后执行：
```bash
cd /home/cho/workspace/ACE/microservices/ace_agent/4.1
export BOT_PATH=./samples/rag_bot/
source deploy/docker/docker_init.sh
docker compose -f deploy/docker/docker-compose.yml up speech-event-bot -d
```

**结果**：成功启动以下 ACE Agent 核心微服务：
* `redis-server` - 会话状态和事件管理
* `plugin-server` - RAG 插件服务器
* `bot-web-ui-client` - Web UI 客户端
* `chat-controller` - 聊天控制器
* `nlp-server` - NLP 服务器
* `bot-web-ui-speech-event` - 语音 Web UI
* `chat-engine-event-speech` - 聊天引擎

**部署状态**：ACE Agent `rag_bot` 的所有必要服务已成功部署，可通过 `http://10.204.222.147:7006/` 访问 Web UI 进行测试。

## 3. 测试与故障排查

### 3.1 Web UI 交互测试

**首次测试（失败）**：
访问 Web UI (`http://10.204.222.147:7006/`)，输入消息 "hi" 后，收到错误提示：
"Sorry I could not connect to the RAG endpoint"

**问题分析**：
1. 通过 `docker network ls` 和 `docker inspect` 检查 Docker 网络配置
2. 发现 RAG Chain Server (`rag-application-text-chatbot-langchain`) 运行在 `nvidia-rag` 网络中
3. Plugin Server (`plugin-server`) 运行在 `host` 网络中
4. 由于网络隔离，Plugin Server 无法通过容器名称解析并访问 RAG Chain Server

**解决方案**：
1. 修改 `plugin_config.yaml`，将 `RAG_SERVER_URL` 改为使用主机 IP 地址：
   ```diff
   -       RAG_SERVER_URL: "http://rag-application-text-chatbot-langchain:8081"
   +       RAG_SERVER_URL: "http://10.204.222.147:8081"
   ```

2. 重启 Plugin Server 以应用新配置：
   ```bash
   cd /home/cho/workspace/ACE/microservices/ace_agent/4.1
   export BOT_PATH=./samples/rag_bot/
   source deploy/docker/docker_init.sh
   docker compose -f deploy/docker/docker-compose.yml restart plugin-server
   ```

**测试结果**：Web UI 现可正常连接 RAG 服务，可以进行文本和语音交互测试。

## 4. 系统架构与部署模式分析

### 4.1 系统架构与部署模式

**RAG Chain Server 部分：**
* **向量数据库 (Milvus)：** 本地部署在 Docker 容器中，用于存储和检索文档向量。
* **Embedding 模型：** 使用远程 NVIDIA API 服务。在 `chain-server` 配置中，将 `APP_EMBEDDINGS_SERVERURL` 设置为空字符串 `""`，并将 `APP_EMBEDDINGS_MODELNAME` 设置为 `"nvidia/nv-embedqa-e5-v5"`，表示使用 NVIDIA 托管的 embedding 模型而非本地部署。
* **大语言模型 (LLM)：** 同样使用远程 NVIDIA API 服务。在配置中，`APP_LLM_SERVERURL` 被设置为空字符串 `""`，`APP_LLM_MODELNAME` 设置为 `"meta/llama3-8b-instruct"`，表示使用 NVIDIA 托管的 Llama3 模型。
* **文档处理和 RAG 逻辑：** 本地运行在 `chain-server` 容器中，但调用远程 API 进行嵌入向量化和生成回答。

**ACE Agent 部分：**
* **语音识别和合成 (Riva)：** 本地部署在 Docker 容器中。Riva 模型已经下载并在本地 `riva-speech-server` 容器中运行，提供 ASR 和 TTS 功能。
* **聊天引擎和控制器：** 本地部署在 Docker 容器中，但通过插件系统连接到 RAG Chain Server。
* **插件服务：** 本地部署在 Docker 容器中，负责连接 ACE Agent 和 RAG Chain Server。
* **Web UI：** 本地部署在 Docker 容器中，提供用户界面。

**API 依赖：**
* 系统高度依赖于 NVIDIA API（通过 `NVIDIA_API_KEY` 访问）来使用远程的嵌入和语言模型服务。这是实现 RAG 功能的核心，没有这些 API 服务，系统将无法正常工作。
* 这种混合架构（本地部署 + 远程 API）可以平衡计算资源和部署复杂度，适合开发和测试环境。

### 4.2 部署过程关键点与挑战

* **环境变量管理：** 确保在同一个 Shell 会话中正确设置环境变量是成功部署的关键。
* **Docker 网络隔离：** RAG Chain Server 和 ACE Agent 服务在不同的 Docker 网络中，需要通过主机 IP 而非容器名称进行通信。
* **API 认证：** 成功传递 `NVIDIA_API_KEY` 到 `chain-server` 容器是启用远程 API 服务的关键步骤。

### 4.3 优化建议

* 考虑将所有服务部署在同一个 Docker 网络中，简化容器间通信。
* 对于生产环境，可以考虑本地部署更多组件（如嵌入模型和语言模型）以减少对外部 API 的依赖，提高系统稳定性和响应速度。
* 进一步优化文档处理和检索逻辑，提高 RAG 的准确性和效率。

## 5. 技术工作流程详解

### 5.1 整体架构

整个系统由两个主要组件构成：
1. **RAG Chain Server** - 负责文档处理、向量化、存储和检索
2. **ACE Agent** - 负责语音处理、用户交互和对话管理

系统架构图（逻辑视图）：

```
[用户] <---> [Web UI] <---> [ACE Agent 核心组件] <---> [RAG Chain Server] <---> [NVIDIA API 服务]
                              |                             |
                              |                             |
                              v                             v
                      [Riva 语音服务器]             [Milvus 向量数据库]
```

### 5.2 数据流程

#### 5.2.1 文档准备与向量化（预处理阶段）

1. **文档上传**：管理员通过 RAG 知识库 UI (`http://<IP>:3001/kb`) 上传文本文档
2. **文档处理**：
   - RAG 应用对文档内容进行解析和切分
   - 文本块被发送到 NVIDIA API 服务进行向量化（使用 `nvidia/nv-embedqa-e5-v5` 模型）
3. **向量存储**：生成的文本向量被存储在本地的 Milvus 向量数据库中

#### 5.2.2 用户查询处理流程

1. **用户输入**：
   - **文本输入**：用户在 Web UI 中输入问题
   - **语音输入**：用户通过麦克风说出问题
     - 语音被本地 Riva ASR 服务实时转换为文本
     - 支持 ASR 2 pass End of Utterance (EOU) 检测，提高响应速度
     - 支持 Barge-In 功能，允许用户随时打断机器人的回答

2. **查询处理**：
   - 用户查询文本由 Plugin Server 接收
   - Plugin Server 将查询转发给 RAG Chain Server 的 `/generate` 端点
   - RAG Chain Server 处理查询：
     1. 将查询文本通过 NVIDIA API 向量化（使用与文档相同的模型）
     2. 在 Milvus 向量数据库中检索相似的文本块（相关文档）
     3. 将检索到的文本块与原始查询一起构建 prompt
     4. 将 prompt 发送给 NVIDIA API 托管的 LLM (`meta/llama3-8b-instruct`) 生成回答
     5. 返回生成的回答给 Plugin Server

3. **响应处理**：
   - Plugin Server 接收回答并可能进行后处理
   - 回答被发送到 Chat Controller/Engine
   - 如果是语音交互：
     - 文本回答被发送到 Riva TTS 服务转换为语音
     - 合成的语音通过 Web UI 播放给用户
   - 同时，文本回答显示在 Web UI 的对话界面上

#### 5.2.3 特殊功能流程

1. **低延迟响应**：
   - ASR 2 pass EOU 技术可以在用户停顿时快速检测语音输入结束
   - LLM 输出流式传输（streaming）支持
   - 平均可以实现 500-600 毫秒的端到端延迟改进

2. **Barge-In 功能**：
   - 系统持续监听用户输入
   - 当用户开始新的语音输入时，系统立即停止当前响应
   - 开始处理新的查询

3. **长停顿处理**：
   - 如果用户在说话过程中停顿超过 240 毫秒
   - 系统可能会提前触发 RAG 查询
   - 如果用户继续说话，系统会重新触发查询
   - 这种设计可减少等待时间，但可能导致每个用户查询平均产生 2 个额外的 RAG 调用

### 5.3 关键技术详解

#### 5.3.1 RAG 技术原理

RAG (Retrieval Augmented Generation) 技术结合了检索系统和生成模型的优势：

1. **文档向量化**：将文档转换为高维向量表示，捕获语义信息
2. **相似度检索**：使用向量相似度查询找到与用户问题最相关的文档片段
3. **上下文增强生成**：将检索到的相关文档作为上下文，与用户问题一起发送给大语言模型
4. **知识增强回答**：LLM 基于检索的上下文生成更加准确、信息丰富的回答

这种方法解决了 LLM 的两个主要局限性：
- 缓解"幻觉"问题，因为回答基于检索到的事实
- 使 LLM 能够访问专业领域或最新的知识，超出其训练数据范围

#### 5.3.2 语音交互技术

本系统使用 NVIDIA Riva 语音 AI SDK 提供高质量、低延迟的语音交互体验：

1. **语音识别 (ASR)**：
   - 使用 Parakeet-ctc-1.1b 英语模型（默认）
   - 支持低延迟标点符号处理
   - 针对口音英语提供更高准确性

2. **语音合成 (TTS)**：
   - 多语言支持，包括英语、德语、西班牙语、中文等
   - 情感表达支持（通过 RADTTS++ 情感混合模型）

3. **低延迟技术**：
   - 两阶段 ASR 设计，实现快速响应
   - 并行处理流，减少端到端延迟

#### 5.3.3 组件间通信机制

1. **Docker 网络通信**：
   - RAG Chain Server 运行在 `nvidia-rag` 网络中
   - ACE Agent 组件运行在 `host` 网络中
   - 通过主机 IP 地址（而非容器名称）进行跨网络通信

2. **API 通信**：
   - Plugin Server 与 RAG Chain Server 通过 HTTP REST API 通信
   - ACE Agent 内部组件通过 gRPC 和事件接口通信
   - 系统与 NVIDIA API 服务通过 HTTP API 通信

### 5.4 典型用户交互场景

典型的用户交互场景步骤：

1. 用户通过 Web UI (`http://10.204.222.147:7006/`) 访问系统
2. 用户点击麦克风图标，启用语音输入（或直接输入文本）
3. 用户询问与上传文档相关的问题："这些文档中有关于什么内容的信息？"
4. Riva ASR 将语音转换为文本
5. 文本查询被发送到 Plugin Server，再转发到 RAG Chain Server
6. RAG Chain Server 检索相关文档并生成回答
7. 回答被转发回 Plugin Server，再到 Chat Controller
8. Riva TTS 将文本转换为语音响应
9. 用户听到语音回答，同时在屏幕上看到文本回答

如果用户想要打断机器人的回答，只需开始说话，系统会立即停止当前响应并处理新的查询。这种交互方式模拟了自然的人类对话流程。

## 6. 安全与性能考量

### 6.1 安全注意事项

* **API 密钥管理**：NVIDIA_API_KEY 等敏感信息应妥善管理，避免硬编码或明文存储。推荐使用环境变量、Docker secrets 或 Kubernetes secrets。
* **网络安全**：系统依赖于多个网络接口（包括外部 API 调用），应考虑适当的防火墙规则和网络策略。
* **认证授权**：Web UI 目前未设置访问控制，生产环境应考虑添加用户认证和授权机制。

### 6.2 性能优化建议

* **文档向量化批处理**：对于大量文档，应考虑批量处理以提高向量化效率。
* **向量数据库索引优化**：调整 Milvus 的索引参数，平衡查询速度和准确性。
* **缓存机制**：为频繁查询添加结果缓存，减少重复计算和 API 调用。
* **资源分配**：根据实际负载调整容器的资源限制和请求，特别是 Riva 语音服务器和向量数据库。

### 6.3 可扩展性考虑

* **水平扩展**：对于高并发场景，可考虑部署多个 Plugin Server 实例，通过负载均衡分发请求。
* **模型服务扩展**：根据需要，可迁移到更大/更小的模型，或混合使用不同的模型以平衡性能和资源消耗。
* **多语言支持**：当前部署主要支持英语，但 Riva 支持多语言配置，可根据需要启用其他语言。

## 7. 总结与最佳实践

本次部署成功实现了 NVIDIA ACE Agent `rag_bot` 示例，构建了一个集成 RAG 技术和语音交互的智能对话系统。部署过程中遇到了多个技术挑战，包括环境变量传递、Docker 网络隔离和 API 认证等问题，通过系统性排查均得到了有效解决。

**最佳实践建议**：

1. **环境统一性**：在同一个 Shell 会话中完成整个部署流程，避免环境变量丢失。
2. **网络规划**：预先规划 Docker 网络拓扑，尽量将相互依赖的服务部署在同一网络中。
3. **分层调试**：遇到问题时采用分层调试方法，从日志分析到配置检查，再到容器内部环境检验。
4. **安全管理**：妥善管理 API 密钥和敏感配置，使用环境变量注入或专门的密钥管理服务。
5. **文档详实**：详细记录每一步操作、遇到的问题和解决方案，为后续维护和扩展提供参考。

本部署为进一步探索 NVIDIA ACE 技术栈、自定义对话智能体和语音交互系统奠定了坚实基础。 