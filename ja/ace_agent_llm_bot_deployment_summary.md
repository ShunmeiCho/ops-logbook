# ACE Agent llm_bot サンプルデプロイ手順概要

このドキュメントは、NVIDIA ACE Agent `llm_bot` サンプルのデプロイ時に実行された操作手順とファイル変更をまとめたものです。

## 1. システム環境チェック

*   **目的:** システムが NVIDIA ACE/Tokkio のデプロイに必要な基本要件を満たしているか評価する。
*   **コマンド:**
    ```bash
    (lsb_release -a || cat /etc/os-release) && echo '---' && (nvidia-smi || echo 'nvidia-smi not found') && echo '---' && free -h && echo '---' && df -h / && echo '---' && (docker --version || echo 'docker not found') && echo '---' && (kubectl version --client --short || echo 'kubectl not found')
    ```
*   **結果:**
    *   オペレーティングシステム: Ubuntu 20.04.6 LTS (注意: Tokkio は 22.04 を要求)
    *   GPU: NVIDIA RTX 6000 Ada Generation (48GB), NVIDIA T600 (4GB) (要件を満たす)
    *   メモリ: 251Gi (十分)
    *   ディスク: 合計 1.8T, 利用可能 205G (注意: Tokkio は >700GB を推奨)
    *   Docker: インストール済み (バージョン 28.0.4)
    *   Kubernetes (kubectl): 未インストール (Tokkio には必要だが、ACE Agent Docker Compose サンプルには不要)
*   **結論:** システムは ACE Agent Docker Compose サンプルのデプロイに適しているが、完全な Tokkio ワークフローのデプロイには障害がある (OS バージョン, ディスク容量, K8s)。

## 2. ACE Agent コードの準備

*   **目的:** ACE Agent のコードとサンプルを取得する。
*   **コマンド:**
    ```bash
    # リポジトリが存在しない場合はクローンし、ターゲットディレクトリに移動
    if [ ! -d "ACE" ]; then git clone https://github.com/NVIDIA/ACE.git; fi && cd ACE/microservices/ace_agent/4.1 && pwd
    ```
*   **現在のディレクトリ:** `/home/cho/workspace/ACE/microservices/ace_agent/4.1`

## 3. ホストされた NIM を使用するように `llm_bot` を設定

*   **目的:** サンプルコードを変更し、ローカルで LLM を実行しようとする代わりに、NVIDIA API Catalog が提供するホストされた LLM モデルに接続するようにする。
*   **ファイル:** `ACE/microservices/ace_agent/4.1/samples/llm_bot/actions.py`
*   **変更点:**
    *   ローカル NIM (NeMo Inference Microservice) の `AsyncOpenAI` クライアント設定をコメントアウトした。
    *   `https://integrate.api.nvidia.com/v1` を `base_url` として使用し、`NVIDIA_API_KEY` 環境変数による認証を行う `AsyncOpenAI` クライアント設定を有効にした。

## 4. Riva ASR/TTS モデルのデプロイ

*   **目的:** 音声インタラクションの基礎となる音声認識 (ASR) および音声合成 (TTS) モデルをダウンロードして準備する。
*   **方法:** Docker Compose と `model-utils-speech` サービスを使用する。
*   **試行 1 & 2 (失敗):**
    *   コマンド: `export BOT_PATH=./samples/llm_bot/ && source deploy/docker/docker_init.sh` の後に `docker compose -f deploy/docker/docker-compose.yml up model-utils-speech`
    *   問題: 1 回目はパスエラーで失敗。2 回目は NGC 認証失敗 (`Invalid org`, `NGC_CLI_API_KEY` 環境変数がコンテナに渡されなかった)。
*   **試行 3 (成功):**
    *   **操作:** `ACE/microservices/ace_agent/4.1/deploy/docker/.env` ファイルを作成し、ユーザーに `NGC_CLI_API_KEY` と `NVIDIA_API_KEY` を入力するように指示した。
        ```
        # Required for downloading ACE container images and models from NGC
        NGC_CLI_API_KEY="YOUR_NGC_API_KEY_HERE" # ユーザーが置き換え済み

        # Required when using NVIDIA API Catalog hosted NIM models (as configured in actions.py)
        NVIDIA_API_KEY="YOUR_NVIDIA_API_KEY_FROM_API_CATALOG_HERE" # ユーザーが置き換え済み
        ```
    *   **コマンド:**
        ```bash
        cd /home/cho/workspace/ACE/microservices/ace_agent/4.1 && docker compose -f deploy/docker/docker-compose.yml up model-utils-speech
        ```
    *   **結果:** 認証に成功し、ASR (parakeet-1.1b) および TTS (fastpitch_hifigan) モデルをダウンロードした。ServiceMaker を使用してモデルを処理し、TensorRT エンジンのビルドも含まれた。最後に、Riva Speech Server を起動し、モデルをロードした。`model-utils-speech` コンテナは正常に完了し、終了した。

## 5. ACE Agent コアサービス (`speech-event-bot`) のデプロイ

*   **目的:** Chat Engine, Chat Controller, Web UI などのコアコンポーネントを含むサービスを起動する。
*   **試行 1 (失敗):**
    *   コマンド: `cd /home/cho/workspace/ACE/microservices/ace_agent/4.1 && docker compose -f deploy/docker/docker-compose.yml up --build speech-event-bot -d`
    *   問題: `docker_init.sh` が同じセッションで実行されなかったため、環境変数 (例: `TAG`, `DOCKER_REGISTRY`) が失われ、`invalid reference format` エラーが発生し、正しいイメージ名が見つからなかった。
*   **試行 2 (成功):**
    *   **コマンド:**
        ```bash
        # up コマンドを実行する前に docker_init.sh を実行することを確認
        cd /home/cho/workspace/ACE/microservices/ace_agent/4.1 && source deploy/docker/docker_init.sh && echo 'Environment re-initialized.' && docker compose -f deploy/docker/docker-compose.yml up --build speech-event-bot -d
        ```
    *   **結果:** 関連イメージのビルドに成功し、必要なすべてのコンテナ (`chat-engine-event-speech`, `chat-controller`, `nlp-server`, `plugin-server`, `redis-server`, `bot-web-ui-client`, `bot-web-ui-speech-event`) を起動した。

## 6. テストとトラブルシューティング

*   **目的:** デプロイが成功したか検証し、遭遇した問題を解決する。
*   **問題 1 (Web UI エラー & 応答不能):**
    *   **症状:** Web UI (`http://10.204.222.147:7006/`) にアクセスした後、エラー `A fatal error occurred while running task GRPCSpeechTask. Message: "[canceled] This operation was aborted".` が発生し、ボットが質問に答えられなくなった。
    *   **ログ診断:**
        *   `chat-controller` ログは Riva ASR サービスへの接続失敗を示した: `Unable to establish connection to server localhost:50051`。
        *   `chat-engine-event-speech` ログは Bot のロード失敗を示した: `CRITICAL No bots available. Please register a bot by providing the path to its config folder.`
    *   **原因分析:**
        *   `chat-engine` エラーは `BOT_PATH` 環境変数がコンテナに正しく渡されなかったため。
        *   `chat-controller` エラーの原因は完全には明らかではないが、音声サービスチェーンに問題があることを示している。
    *   **解決策試行:** サービスを停止した後、起動コマンドの前に `BOT_PATH` を明示的に設定し、`docker_init.sh` を再実行してから、`speech-event-bot` を再度起動した。
        ```bash
        cd /home/cho/workspace/ACE/microservices/ace_agent/4.1 && \
        export BOT_PATH=./samples/llm_bot/ && \
        source deploy/docker/docker_init.sh && \
        echo "BOT_PATH is set to: $BOT_PATH" && \
        docker compose -f deploy/docker/docker-compose.yml up --build speech-event-bot -d
        ```
    *   **結果:** サービスは正常に起動し、**テキスト入力機能が回復した**。
*   **問題 2 (マイク有効化不可):**
    *   **症状:** Web UI で、マイクをクリックまたは有効にして音声入力を行うことができない。
    *   **原因分析:** ブラウザのセキュリティポリシーにより、安全でない HTTP 接続 (`http://10.204.222.147:7006`) を介したマイクへのアクセスがブロックされている。
    *   **解決策:** ブラウザ設定を変更し、このアドレスを安全なオリジンとして扱う。
        1.  `chrome://flags` または `edge://flags` にアクセスする。
        2.  `#unsafely-treat-insecure-origin-as-secure` フラグを検索して有効にする。
        3.  テキストボックスに `http://10.204.222.147:7006` を追加する。
        4.  ブラウザを再起動する。
    *   **結果:** **マイクが正常に有効化された**。

## 7. 現在の状態と次のステップ

*   **現在の状態:** すべてのサービスが正常に起動した。Web UI の**テキストおよび音声入力機能は両方とも有効**であり、テスト可能。
*   **次のステップ:**
    1.  Web UI のテキストおよび音声インタラクション機能を包括的にテストする。
    2.  (オプション) 他の ACE Agent 機能またはサンプルを探索する。
    3.  テスト完了後、すべてのサービスを停止する: `cd /home/cho/workspace/ACE/microservices/ace_agent/4.1 && docker compose -f deploy/docker/docker-compose.yml down`。
*   **サービスの再起動:** すべての ACE Agent サービスを再起動する必要がある場合は、次の手順を実行する：
    1.  **現在のサービスを停止する (実行中の場合):**
        ```bash
        cd /home/cho/workspace/ACE/microservices/ace_agent/4.1 && docker compose -f deploy/docker/docker-compose.yml down
        ```
    2.  **サービスを再起動する:**
        ```bash
        cd /home/cho/workspace/ACE/microservices/ace_agent/4.1 && \
        export BOT_PATH=./samples/llm_bot/ && \
        source deploy/docker/docker_init.sh && \
        docker compose -f deploy/docker/docker-compose.yml up --build speech-event-bot -d
        ```

## 8. モデルデプロイと API 利用状況

### LLM モデル
*   **デプロイ方式:** NVIDIA API Catalog ホストの NIM サービス
*   **使用モデル:** `meta/llama3-8b-instruct`
*   **設定情報:**
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
*   **認証方式:** `NVIDIA_API_KEY` 環境変数を使用
*   **特徴:**
    - ストリーミング出力対応、応答遅延を低減
    - 対話履歴管理対応
    - ユーザー クエリあたり平均 2～3 回の API 呼び出しが必要になる可能性あり（240ms の一時停止再トリガー メカニズムのため）

### 音声モデル（ローカルデプロイ）
*   **ASR (音声認識):**
    - モデル: Riva ASR `parakeet-ctc-1.1b` 英語モデル
    - 特徴:
        * 低遅延の ASR 2 パス End of Utterance (EOU) をサポート
        * 単語重み付け（word boosting）機能をサポート
        * 不適切表現フィルタリングをサポート
        * 英語のアクセントに対するロバスト性が高い
*   **TTS (音声合成):**
    - モデル: Riva TTS `fastpitch_hifigan` 英語モデル
    - gRPC API を介してサービスを提供
    - リアルタイム音声合成をサポート

### デプロイ アーキテクチャの特徴
1. **ハイブリッド デプロイ戦略:**
   - LLM：クラウド API を使用（計算リソース要件を削減）
   - 音声サービス：ローカル デプロイ（低遅延を確保）

2. **最適化措置:**
   - NVIDIA TensorRT を使用したモデル最適化
   - Triton Inference Server によるモデル デプロイ
   - Chat Controller の最適化により低遅延と高スループットを確保

3. **オプション設定:**
   - 完全なローカル デプロイへの切り替えをサポート（A100 または H100 GPU が必要）
   - NVIDIA NIM の代わりに OpenAI API の使用をサポート
   - サードパーティ TTS ソリューション（ElevenLabs など）の使用をサポート

4. **システム統合:**
   - gRPC API を使用したストリーミング低遅延音声サービスの実装
   - 純粋なテキスト チャットボット用の REST API をサポート
   - LangChain、LlamaIndex などのフレームワークとの統合をサポート

## 9. 日本語対応のための変更分析

本章では、現在の Docker Compose でデプロイされた `llm_bot` をベースに、日本語インタラクションをサポートするために必要な変更手順と注意点を分析します。

### コア戦略

ACE Agent は多言語デプロイをサポートしていますが、通常、1 つのインスタンスは 1 つの主要言語を処理するように設定されます。日本語をサポートするには、主に 2 つの戦略があります。

1.  **多言語または日本語 LLM を使用する:** 日本語を理解し生成できる LLM モデルを直接使用する。
2.  **ニューラル機械翻訳 (NMT) を使用する:** 日本語入力を LLM が理解できる言語（英語など）に翻訳し、LLM の応答を日本語に翻訳し直す。

### 具体的な変更手順 (Docker Compose ベース)

1.  **日本語 Riva モデルの確認とデプロイ:**
    *   **ASR (音声認識):**
        *   Riva ドキュメントまたは NGC カタログを確認し、利用可能な日本語 ASR モデル (例: `ja-JP` 関連モデル) があるか確認する。
        *   見つかった場合、`model-utils-speech` サービスの設定（例: `docker_init.sh` または `docker-compose.yml`）を変更し、英語 ASR モデルを日本語モデルに置き換える。
    *   **TTS (音声合成):**
        *   Riva が日本語 TTS モデルと音声名を提供しているか確認する。
        *   見つかった場合、`model-utils-speech` 設定を変更し、英語 TTS モデルを日本語モデルに置き換える。
        *   **代替案:** Riva が日本語 TTS をサポートしていない場合、サードパーティ TTS サービス (例: ElevenLabs) の統合を検討する。これには `nlp-server` の変更が必要。
    *   **NMT (ニューラル機械翻訳 - 戦略 2 を採用する場合):**
        *   Riva が日本語翻訳をサポートする NMT モデルを提供しているか確認する。
        *   NMT 戦略を選択した場合、NMT モデルを `model-utils-speech` のデプロイ リストに追加する必要がある。

2.  **Chat Controller 設定の変更:**
    *   **ファイル:** `ACE/microservices/ace_agent/4.1/deploy/docker/docker-compose.yml` 内の `chat-controller` サービスの環境変数またはそれが参照する設定ファイル。
    *   **ASR 言語:** 関連する設定項目 (例: `RIVA_ASR_LANGUAGE_CODE`) を `"en-US"` から `"ja-JP"` に変更する。
    *   **TTS 言語と音声:** 関連する設定項目 (例: `RIVA_TTS_LANGUAGE_CODE` と `RIVA_TTS_VOICE_NAME`) を日本語コードと利用可能な音声名に変更する。

3.  **LLM 設定または Bot ロジック (`actions.py`) の変更:**
    *   **戦略 1 (日本語/多言語 LLM を使用):**
        *   **ファイル:** `ACE/microservices/ace_agent/4.1/samples/llm_bot/actions.py`
        *   `MODEL` 変数を日本語をサポートする LLM 識別子に変更する。
        *   必要に応じて、`BASE_URL` と認証方式を更新する。
        *   `SYSTEM_PROMPT` を調整する。
    *   **戦略 2 (NMT を使用):**
        *   **ファイル:** `ACE/microservices/ace_agent/4.1/samples/llm_bot/actions.py`
        *   LLM 設定は変更しない。
        *   `call_nim_local_llm` アクションを変更する：
            *   LLM を呼び出す前に、Riva NMT を呼び出して日本語クエリを英語に翻訳する。
            *   翻訳された英語を使用して LLM を呼び出す。
            *   結果を返す前に、Riva NMT を呼び出して LLM の英語応答を日本語に翻訳し直す。
            *   これには、gRPC を介して Riva NMT サービスを呼び出す方法を理解する必要がある。

4.  **サービスの再ビルドと再起動:**
    *   設定とコードを変更した後、既存のサービスを停止する:
        ```bash
        cd /home/cho/workspace/ACE/microservices/ace_agent/4.1 && docker compose -f deploy/docker/docker-compose.yml down
        ```
    *   新しいモデル設定に従って `docker_init.sh` を更新し (必要な場合)、サービスを再起動する:
        ```bash
        cd /home/cho/workspace/ACE/microservices/ace_agent/4.1 && export BOT_PATH=./samples/llm_bot/ && source deploy/docker/docker_init.sh && docker compose -f deploy/docker/docker-compose.yml up --build speech-event-bot -d
        ```

### 重要注意事項
*   **モデルの可用性:** Riva が高品質な日本語 ASR および TTS モデルを提供しているか確認することが重要。
*   **LLM 能力:** 現在使用している Llama3 8B は日本語を十分に処理できない可能性があるため、適切な多言語または日本語 LLM を選択する必要がある。
*   **単一言語インスタンス:** 変更後、このデプロイ インスタンスは主に日本語を処理するようになる。 