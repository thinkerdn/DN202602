---

# 概要設計：Salesforceデータ定期抽出バッチ

## 1. システム構成図（論理構成）

1. **Amazon EventBridge (Scheduler)**: 定期実行のトリガー
2. **AWS Lambda**: メインロジック（認証・データ取得・S3保存）
3. **AWS Secrets Manager**: Salesforce接続情報の安全な管理
4. **Salesforce API**: データソース（REST または Bulk API 2.0）
5. **Amazon S3**: データ保存先ストレージ
6. **Amazon CloudWatch**: 監視・ログ管理

---

## 2. 詳細設計方針

### 2.1. 実行制御 (EventBridge)

* **EventBridge Scheduler** を使用。
* **リトライポリシー:** APIの一時的なエラーに備え、最大リトライ回数と再試行間隔を設定。
* **タイムゾーン:** `Asia/Tokyo` 明示指定による運用ミス防止。

### 2.2. 認証処理 (Lambda & Secrets Manager)

* **認証方式:** **OAuth 2.0 JWT Bearer Flow** を推奨（パスワード更新が不要で、バッチ処理に最適）。
* **秘密情報の管理:** * `Consumer Key`、`Private Key`、`Username` を **AWS Secrets Manager** で管理。
* Lambda実行時に環境変数から取得するのではなく、SDK経由で直接読み込む。



### 2.3. データ取得ロジック (Lambda)

* **APIの使い分け:**
* 小規模データ（数千件以下）: **REST API (SOQL)**。
* 大規模データ: **Bulk API 2.0**。Lambdaの実行時間制限（15分）を考慮し、非同期でジョブを作成・監視する。


* **言語:** パフォーマンスとライブラリ（`simple-salesforce` 等）の充実度から **Python 3.x** を推奨。

### 2.4. データ保存 (S3)

* **ディレクトリ構造:** パーティショニング（Athena等での利用）を考慮。
* `s3://[BucketName]/raw/[ObjectName]/year=YYYY/month=MM/day=DD/[ObjectName]_YYYYMMDD_HHMM.csv`


* **暗号化:** **Amazon S3 Managed Keys (SSE-S3)** または **AWS KMS** によるサーバーサイド暗号化を有効化。

---

## 3. ベストプラクティスに基づく実装ポイント

### ① セキュリティ (Security)

* **IAM 最小権限:** Lambda ロールには「S3への書き込み」「Secrets Managerの特定シークレットの読み取り」「CloudWatch Logsへの出力」権限のみを付与。
* **VPC Endpoint (オプション):** セキュリティ要件が高い場合、VPC内から S3 や Secrets Manager に PrivateLink 経由で接続。

### ② 信頼性と監視 (Reliability & Monitoring)

* **Dead Letter Queue (DLQ):** Lambdaの失敗を検知するため、SQSまたはSNSをDLQとして設定。
* **CloudWatch Metrics & Alarms:** 「エラー率」や「Lambdaの実行時間」にアラートを設定し、異常時に通知。

### ③ パフォーマンスとコスト (Performance & Cost)

* **メモリ最適化:** Lambdaのメモリを適切に設定（メモリを増やすとCPU性能も上がり、実行時間が短縮されてコストが下がる場合がある）。
* **ライフサイクルルール:** S3上の古いデータを一定期間後に Glacier へ移行、または削除するルールを設定。

---

## 4. 異常系処理

| 事象 | 対策 |
| --- | --- |
| **Salesforce API 制限超過** | CloudWatch Logs にエラーを記録し、管理者に通知。必要に応じて抽出条件を絞り込み。 |
| **Lambda タイムアウト** | 大量データの場合は Step Functions を導入し、Bulk API のステータス確認をループ処理化する。 |
| **S3 書き込み失敗** | IAM 権限の確認と、EventBridge 側のリトライ機能による再試行。 |

---
