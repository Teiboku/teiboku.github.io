---
title: "A model release pipeline"
date: 2025-01-15
draft: false
description: "a description"
tags: ["example", "tag"]
summary: "A model release pipeline for our semantic search"
---
## 参考リンク
[PR by me](https://github.com/opensearch-project/opensearch-py-ml/pull/394)

[リリースされたモデル](https://huggingface.co/opensearch-project)
## はじめに

AWS OpenSearchでは、ユーザーがダウンロードできる事前トレーニング済みモデルを提供しています。これらの事前トレーニング済みモデルは、専用のパイプラインを通じて管理され、トレーニングから推論、デプロイメントまでのさまざまな検証が行われます。モデルは指定されたサーバーにアップロードされ、関連するドキュメントが適宜更新されます。

新しいセマンティックモデルをリリースした後、元のパイプラインでは効果的に検証およびデプロイできないことがわかりました。これは、従来の密なモデルではないためです。その結果、私たちはニューラルスパースモデルを管理および維持するための新しいパイプラインを実装することに決めました。

## 私が行った作業	
スパースモデルのアップロードプロセスを自動化しました：
私は（または修正した）完全なGitHub Actionsワークフローと、特定の条件（model_type = Sparseのとき）で以下のアクションを実行するために必要なPythonコードを書きました：
	•	入力パラメータの準備（モデルソース、ID、バージョン、トラッキングフォーマット、埋め込み次元、プーリングモード、説明など）
	•	自動トレース：元のモデルをTorchScriptまたはONNX形式に変換するか、両方の形式を同時に生成します。
	•	埋め込み検証：変換されたモデルの埋め込み次元と機能が正しいことを確認します。
	•	モデルがすでに存在するか確認：存在し、上書きできない場合は、アップロードを拒否します。
	•	手動承認：重要な担当者の承認後にのみ、正式にS3にアップロードします。
	•	リポジトリのドキュメントと記録を更新：アップロードアクションをMODEL_UPLOAD_HISTORY.md、supported_models.jsonに追加し、CHANGELOG.mdを更新します。
	•	下流のJenkinsプロセスをトリガーして、モデルのリリースまたはml-modelsへの統合をさらに完了します。

## 使用技術
- Python
- GitHub Actions
- Huggingface
- Jenkins
- AWS S3
- AWS OpenSearch
- AWS IAM
- AWS Secrets Manager