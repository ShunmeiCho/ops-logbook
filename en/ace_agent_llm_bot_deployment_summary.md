# ACE Agent llm_bot Sample Deployment Steps Summary

This document summarizes the operational steps and file changes performed to deploy the NVIDIA ACE Agent `llm_bot` sample.

## 1. System Environment Check

*   **Purpose:** Assess if the system meets the basic requirements for deploying NVIDIA ACE/Tokkio.
*   **Command:**
    ```bash
    (lsb_release -a || cat /etc/os-release) && echo '---' && (nvidia-smi || echo 'nvidia-smi not found') && echo '---' && free -h && echo '---' && df -h / && echo '---' && (docker --version || echo 'docker not found') && echo '---' && (kubectl version --client --short || echo 'kubectl not found')
    ```
*   **Results:**
    *   Operating System: Ubuntu 20.04.6 LTS (Note: Tokkio requires 22.04)
    *   GPU: NVIDIA RTX 6000 Ada Generation (48GB), NVIDIA T600 (4GB) (Meets requirements)
    *   Memory: 251Gi (Sufficient)
    *   Disk: 1.8T total, 205G available (Note: Tokkio recommends >700GB)
    *   Docker: Installed (Version 28.0.4)
    *   Kubernetes (kubectl): Not installed (Required for Tokkio, not needed for ACE Agent Docker Compose sample)
*   **Conclusion:** The system is suitable for deploying the ACE Agent Docker Compose sample, but there are obstacles to deploying the full Tokkio workflow (OS version, disk space, K8s).

## 2. Prepare ACE Agent Code

*   **Purpose:** Obtain the ACE Agent code and samples.
*   **Command:**
    ```bash
    # Clone the repository (if it doesn't exist) and navigate to the target directory
    if [ ! -d "ACE" ]; then git clone https://github.com/NVIDIA/ACE.git; fi && cd ACE/microservices/ace_agent/4.1 && pwd
    ```
*   **Current Directory:** `/home/cho/workspace/ACE/microservices/ace_agent/4.1`

## 3. Configure `llm_bot` to Use Hosted NIM

*   **Purpose:** Modify the sample code to connect to the hosted LLM model provided by the NVIDIA API Catalog instead of attempting to run the LLM locally.
*   **File:** `ACE/microservices/ace_agent/4.1/samples/llm_bot/actions.py`
*   **Changes:**
    *   Commented out the local NIM (NeMo Inference Microservice) `AsyncOpenAI` client configuration.
    *   Enabled the `AsyncOpenAI` client configuration that uses `https://integrate.api.nvidia.com/v1` as the `base_url` and authenticates via the `NVIDIA_API_KEY` environment variable.

## 4. Deploy Riva ASR/TTS Models

*   **Purpose:** Download and prepare the Automatic Speech Recognition (ASR) and Text-to-Speech (TTS) models, which are fundamental for voice interaction.
*   **Method:** Use Docker Compose and the `model-utils-speech` service.
*   **Attempt 1 & 2 (Failed):**
    *   Command: `export BOT_PATH=./samples/llm_bot/ && source deploy/docker/docker_init.sh` followed by `docker compose -f deploy/docker/docker-compose.yml up model-utils-speech`
    *   Issue: Failed the first time due to a path error. Failed the second time due to NGC authentication failure (`Invalid org`, `NGC_CLI_API_KEY` environment variable not passed to the container).
*   **Attempt 3 (Successful):**
    *   **Action:** Created the file `ACE/microservices/ace_agent/4.1/deploy/docker/.env` and instructed the user to fill in `NGC_CLI_API_KEY` and `NVIDIA_API_KEY`.
        ```        # Required for downloading ACE container images and models from NGC
        NGC_CLI_API_KEY="YOUR_NGC_API_KEY_HERE" # Replaced by user

        # Required when using NVIDIA API Catalog hosted NIM models (as configured in actions.py)
        NVIDIA_API_KEY="YOUR_NVIDIA_API_KEY_FROM_API_CATALOG_HERE" # Replaced by user
        ```
    *   **Command:**
        ```bash
        cd /home/cho/workspace/ACE/microservices/ace_agent/4.1 && docker compose -f deploy/docker/docker-compose.yml up model-utils-speech
        ```
    *   **Result:** Successfully authenticated, downloaded ASR (parakeet-1.1b) and TTS (fastpitch_hifigan) models. Processed the models using ServiceMaker, including building TensorRT engines. Finally, started the Riva Speech Server and loaded the models. The `model-utils-speech` container completed successfully and exited.

## 5. Deploy ACE Agent Core Services (`speech-event-bot`)

*   **Purpose:** Start the services containing core components like Chat Engine, Chat Controller, Web UI, etc.
*   **Attempt 1 (Failed):**
    *   Command: `cd /home/cho/workspace/ACE/microservices/ace_agent/4.1 && docker compose -f deploy/docker/docker-compose.yml up --build speech-event-bot -d`
    *   Issue: `docker_init.sh` was not executed in the same session, causing environment variables (like `TAG`, `DOCKER_REGISTRY`) to be lost, resulting in an `invalid reference format` error and inability to find the correct image name.
*   **Attempt 2 (Successful):**
    *   **Command:**
        ```bash
        # Ensure docker_init.sh is executed before running the up command
        cd /home/cho/workspace/ACE/microservices/ace_agent/4.1 && source deploy/docker/docker_init.sh && echo 'Environment re-initialized.' && docker compose -f deploy/docker/docker-compose.yml up --build speech-event-bot -d
        ```
    *   **Result:** Successfully built the relevant images and started all necessary containers (`chat-engine-event-speech`, `chat-controller`, `nlp-server`, `plugin-server`, `redis-server`, `bot-web-ui-client`, `bot-web-ui-speech-event`).

## 6. Testing and Troubleshooting

*   **Purpose:** Verify if the deployment was successful and resolve encountered issues.
*   **Issue 1 (Web UI Error & Unresponsive):**
    *   **Symptom:** After accessing the Web UI (`http://10.204.222.147:7006/`), an error `A fatal error occurred while running task GRPCSpeechTask. Message: "[canceled] This operation was aborted".` appeared, and the bot could not answer questions.
    *   **Log Diagnosis:**
        *   `chat-controller` logs showed failure to connect to Riva ASR service: `Unable to establish connection to server localhost:50051`.
        *   `chat-engine-event-speech` logs showed failure to load the Bot: `CRITICAL No bots available. Please register a bot by providing the path to its config folder.`
    *   **Cause Analysis:**
        *   The `chat-engine` error was due to the `BOT_PATH` environment variable not being correctly passed to the container.
        *   The cause of the `chat-controller` error was not entirely clear but indicated a problem in the voice service chain.
    *   **Solution Attempt:** After stopping the services, explicitly set `BOT_PATH` before the startup command, re-run `docker_init.sh`, and then start `speech-event-bot` again.
        ```bash
        cd /home/cho/workspace/ACE/microservices/ace_agent/4.1 && \
        export BOT_PATH=./samples/llm_bot/ && \
        source deploy/docker/docker_init.sh && \
        echo "BOT_PATH is set to: $BOT_PATH" && \
        docker compose -f deploy/docker/docker-compose.yml up --build speech-event-bot -d
        ```
    *   **Result:** Services started successfully, **text input functionality restored**.
*   **Issue 2 (Unable to Enable Microphone):**
    *   **Symptom:** In the Web UI, unable to click or enable the microphone for voice input.
    *   **Cause Analysis:** Browser security policy prevents microphone access over an insecure HTTP connection (`http://10.204.222.147:7006`).
    *   **Solution:** Modify browser settings to treat this address as a secure origin.
        1.  Visit `chrome://flags` or `edge://flags`.
        2.  Search for and enable the `#unsafely-treat-insecure-origin-as-secure` flag.
        3.  Add `http://10.204.222.147:7006` to the text box.
        4.  Restart the browser.
    *   **Result:** **Microphone successfully enabled**.

## 7. Current Status and Next Steps

*   **Current Status:** All services have started successfully. Both **text and voice input functions of the Web UI are enabled** and testable.
*   **Next Steps:**
    1.  Comprehensively test the text and voice interaction functions of the Web UI.
    2.  (Optional) Explore other ACE Agent features or samples.
    3.  After completing tests, stop all services: `cd /home/cho/workspace/ACE/microservices/ace_agent/4.1 && docker compose -f deploy/docker/docker-compose.yml down`.
*   **Restarting Services:** If you need to restart all ACE Agent services, follow these steps:
    1.  **Stop current services (if running):**
        ```bash
        cd /home/cho/workspace/ACE/microservices/ace_agent/4.1 && docker compose -f deploy/docker/docker-compose.yml down
        ```
    2.  **Restart services:**
        ```bash
        cd /home/cho/workspace/ACE/microservices/ace_agent/4.1 && \
        export BOT_PATH=./samples/llm_bot/ && \
        source deploy/docker/docker_init.sh && \
        docker compose -f deploy/docker/docker-compose.yml up --build speech-event-bot -d
        ```

## 8. Model Deployment and API Usage

### LLM Model
*   **Deployment Method:** NIM service hosted by NVIDIA API Catalog
*   **Model Used:** `meta/llama3-8b-instruct`
*   **Configuration Info:**
    ```python
    BASE_URL = "https://integrate.api.nvidia.com/v1"
    client = AsyncOpenAI(
        base_url=BASE_URL,
        api_key=os.getenv("NVIDIA_API_KEY")
    )
    MODEL = "meta/llama3-8b-instruct"
    TEMPERATURE = 0.2
    TOP_P = 0.7
    MAX_TOKENS = 1024
    SYSTEM_PROMPT = "You are a helpful AI assistant."
    ```
*   **Authentication Method:** Using the `NVIDIA_API_KEY` environment variable
*   **Features:**
    - Supports streaming output, reducing response latency
    - Supports conversation history management
    - May require 2-3 API calls per user query on average (due to the 240ms pause re-trigger mechanism)

### Speech Models (Local Deployment)
*   **ASR (Automatic Speech Recognition):**
    - Model: Riva ASR `parakeet-ctc-1.1b` English model
    - Features:
        * Supports low-latency ASR 2 pass End of Utterance (EOU)
        * Supports word boosting feature
        * Supports profanity filtering
        * Robust to English accents
*   **TTS (Text-to-Speech):**
    - Model: Riva TTS `fastpitch_hifigan` English model
    - Served via gRPC APIs
    - Supports real-time speech synthesis

### Deployment Architecture Features
1. **Hybrid Deployment Strategy:**
   - LLM: Uses cloud API (reduces computational resource requirements)
   - Speech Services: Deployed locally (ensures low latency)

2. **Optimization Measures:**
   - Model optimization using NVIDIA TensorRT
   - Model deployment via Triton Inference Server
   - Chat Controller optimization ensures low latency and high throughput

3. **Optional Configurations:**
   - Supports switching to a fully local deployment (requires A100 or H100 GPU)
   - Supports using OpenAI API instead of NVIDIA NIM
   - Supports using third-party TTS solutions (e.g., ElevenLabs)

4. **System Integration:**
   - Implementation of streaming low-latency speech services using gRPC APIs
   - Supports REST APIs for pure text chatbots
   - Supports integration with frameworks like LangChain, LlamaIndex

## 9. Analysis of Modifications for Japanese Support

This section analyzes the modification steps and considerations required to support Japanese interaction based on the current Docker Compose deployment of `llm_bot`.

### Core Strategy

ACE Agent supports multi-language deployment, but an instance is typically configured to handle one primary language. There are mainly two strategies to support Japanese:

1.  **Use a multilingual or Japanese LLM:** Directly use an LLM model that can understand and generate Japanese.
2.  **Use Neural Machine Translation (NMT):** Translate Japanese input into a language the LLM understands (like English), and then translate the LLM's response back into Japanese.

### Specific Modification Steps (Based on Docker Compose)

1.  **Check and Deploy Japanese Riva Models:**
    *   **ASR (Speech Recognition):**
        *   Check Riva documentation or the NGC catalog to confirm if a suitable Japanese ASR model is available (e.g., `ja-JP` related models).
        *   If found, modify the configuration of the `model-utils-speech` service (e.g., in `docker_init.sh` or `docker-compose.yml`) to replace the English ASR model with the Japanese model.
    *   **TTS (Speech Synthesis):**
        *   Check if Riva provides Japanese TTS models and voice names.
        *   If found, modify the `model-utils-speech` configuration to replace the English TTS model with the Japanese model.
        *   **Alternative:** If Riva does not support Japanese TTS, consider integrating a third-party TTS service (e.g., ElevenLabs), which would require modifying `nlp-server`.
    *   **NMT (Neural Machine Translation - if adopting Strategy 2):**
        *   Check if Riva provides an NMT model that supports Japanese translation.
        *   If choosing the NMT strategy, the NMT model needs to be added to the deployment list of `model-utils-speech`.

2.  **Modify Chat Controller Configuration:**
    *   **File:** Environment variables for the `chat-controller` service in `ACE/microservices/ace_agent/4.1/deploy/docker/docker-compose.yml` or its referenced configuration file.
    *   **ASR Language:** Modify the relevant configuration item (e.g., `RIVA_ASR_LANGUAGE_CODE`) from `"en-US"` to `"ja-JP"`.
    *   **TTS Language and Voice:** Modify the relevant configuration items (e.g., `RIVA_TTS_LANGUAGE_CODE` and `RIVA_TTS_VOICE_NAME`) to the Japanese code and available voice name.

3.  **Modify LLM Configuration or Bot Logic (`actions.py`):**
    *   **Strategy 1 (Using Japanese/Multilingual LLM):**
        *   **File:** `ACE/microservices/ace_agent/4.1/samples/llm_bot/actions.py`
        *   Change the `MODEL` variable to the identifier of an LLM supporting Japanese.
        *   If necessary, update the `BASE_URL` and authentication method.
        *   Adjust the `SYSTEM_PROMPT`.
    *   **Strategy 2 (Using NMT):**
        *   **File:** `ACE/microservices/ace_agent/4.1/samples/llm_bot/actions.py`
        *   Keep the LLM configuration unchanged.
        *   Modify the `call_nim_local_llm` action:
            *   Before calling the LLM, call Riva NMT to translate the Japanese query into English.
            *   Call the LLM using the translated English.
            *   Before returning the result, call Riva NMT to translate the LLM's English response back into Japanese.
            *   This requires understanding how to call the Riva NMT service via gRPC.

4.  **Rebuild and Restart Services:**
    *   After modifying configurations and code, stop the existing services:
        ```bash
        cd /home/cho/workspace/ACE/microservices/ace_agent/4.1 && docker compose -f deploy/docker/docker-compose.yml down
        ```
    *   Update `docker_init.sh` according to the new model configuration (if needed), then restart the services:
        ```bash
        cd /home/cho/workspace/ACE/microservices/ace_agent/4.1 && export BOT_PATH=./samples/llm_bot/ && source deploy/docker/docker_init.sh && docker compose -f deploy/docker/docker-compose.yml up --build speech-event-bot -d
        ```

### Important Considerations
*   **Model Availability:** Confirming whether Riva provides high-quality Japanese ASR and TTS models is crucial.
*   **LLM Capability:** The currently used Llama3 8B might not handle Japanese well enough; selecting a suitable multilingual or Japanese LLM is necessary.
*   **Single-Language Instance:** After modification, this deployment instance will primarily handle Japanese. 