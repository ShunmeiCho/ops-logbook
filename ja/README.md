# 日本語運用記録

このディレクトリには、様々なシステムやアプリケーションの日本語運用記録が含まれています。これらの記録は、環境構成からデプロイメント、トラブルシューティングまでの完全な運用ワークフローを文書化しています。

## ファイル一覧

以下は現在利用可能な日本語運用記録のリストです：

1. [ACE Agent LLM ボットデプロイメント概要](ace_agent_llm_bot_deployment_summary.md) - NVIDIA ACE Agent LLM ボットのデプロイメント手順、構成変更、トラブルシューティングの解決策の詳細な記録。

## 文書形式

各運用記録は統一された形式で構成されており、以下の主要セクションが含まれています：

1. **環境確認** - システム環境の評価と要件検証
2. **事前準備** - コード取得、構成変更などの準備作業
3. **モデルデプロイメント** - モデルのダウンロードと構成手順
4. **コアサービスデプロイメント** - サービス起動と構成プロセス
5. **テストとトラブルシューティング** - デプロイメント後の検証と問題解決
6. **現状と次のステップ** - システム状態の要約と推奨されるフォローアップアクション
7. **モデルデプロイメントとAPI使用** - モデルとAPIの詳細な構成情報
8. **多言語サポート分析** - (該当する場合) 多言語サポートの実装分析

## 貢献ガイドライン

新しい日本語運用記録を貢献したい場合は、以下の命名規則に従ってください：

- ファイル名は簡潔で、操作内容を反映したものにする
- 単語をつなぐにはアンダースコアを使用する
- ファイル拡張子は `.md` にする

例：`system_name_operation_type.md`

プロジェクトのルートディレクトリにあるREADMEに記載されている文書形式標準に従っていることを確認してください。 