---

# 概要設計：S3からSalesforceへのデータ取り込みバッチ (AppFlow版)

## 1. システム構成図（論理構成）

1. **Amazon S3**: ソースデータ（CSV/JSONなど）の配置場所
2. **Amazon EventBridge (Scheduler)**: AppFlowジョブの定期実行トリガー
3. **AWS AppFlow**: データ転送・マッピング・Salesforce API実行
4. **AWS Secrets Manager / AWS KMS**: Salesforce接続情報とデータの暗号化管理
5. **Salesforce**: ターゲット（データのインポート先）
6. **Amazon CloudWatch**: 実行結果の監視とエラー通知

---

## 2. 詳細設計方針

### 2.1. 実行トリガー (EventBridge / AppFlow Schedule)

* **実行方式:** AppFlow自体の「スケジュール実行」機能を利用。
* **増分転送:** 前回実行時からの差分ファイルのみを処理するように「増分転送設定」を有効化。

### 2.2. データソース設計 (S3)

* **ファイル形式:** CSV または JSON 形式。
* **配置ルール:** * `s3://[BucketName]/inbound/salesforce/[ObjectName]/` 配下に配置。
* 処理済みファイルを識別するため、AppFlowの機能で処理後にファイルをアーカイブフォルダへ移動させる運用を推奨。



### 2.3. データ転送・マッピング (AppFlow)

* **書き込みモード:** * **Upsert (更新または挿入):** 既存レコードの更新と新規作成を同時に行う。Salesforce側の「外部ID (External ID)」フィールドをキーとして指定。
* **エラーハンドリング:** * Salesforce側のバリデーションエラーが発生した場合、エラーレコードのみを特定のS3バケットに出力するように設定。
* **APIの選択:**
* 大量データ（数千件以上）の場合は、AppFlowの設定で **Salesforce Bulk API 2.0** を選択し、ガバナ制限を回避。



### 2.4. セキュリティ設計

* **認証:** Salesforceの「接続アプリケーション (Connected App)」と連携し、OAuth 認証を使用。
* **暗号化:** S3上のデータおよび転送経路を AWS KMS で暗号化。
* **プライベート接続:** AWS PrivateLink (AppFlow用) を利用することで、インターネットを経由せずにSalesforceと通信する構成も検討。

---

## 3. ベストプラクティスに基づく運用ポイント

### ① データの整合性と「外部ID」

* Salesforce側に取り込む際、`Id` (Salesforce ID) を直接指定するのではなく、基幹システムのIDなどを **External ID** としてSalesforceに定義し、それをマッチングキーに使うことで、重複データの発生を防ぎます。

### ② エラー監視 (Monitoring)

* **EventBridge 通知:** AppFlowの実行ステータス（Success/Failure）を EventBridge でキャッチし、失敗時に SNS 経由で Slack やメールに通知。
* **実行ログ:** CloudWatch Logs を有効化し、どのレコードがなぜ失敗したか（型不一致、入力規則違反など）を追跡可能にする。

### ③ 流量制御

* 短時間に大量のファイルをS3に置くと、AppFlowのジョブが並列で走りすぎてSalesforce側のロック競合（Row Lock）が発生する可能性があります。実行頻度とデータ量のバランスを調整します。

---

## 4. ワークフロー（処理の流れ）

| ステップ | アクション | 内容 |
| --- | --- | --- |
| 1 | **ファイル配置** | 外部システムが S3 の指定バケットにデータをアップロード。 |
| 2 | **ジョブ起動** | EventBridge Scheduler が設定された時間に AppFlow ジョブを起動。 |
| 3 | **データ変換** | AppFlow 内で S3 のカラムと Salesforce のフィールドをマッピング。 |
| 4 | **データ転送** | Bulk API 2.0 を経由して Salesforce へ Upsert。 |
| 5 | **結果確認** | 成功/失敗の結果を CloudWatch に記録。失敗レコードは S3 のエラー専用フォルダへ。 |

---
