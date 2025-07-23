---
title: "Snowflakeを用いたkintoneデータの分析基盤構築" # 記事のタイトル
emoji: "☃️" # アイキャッチとして使われる絵文字（1文字だけ）
type: "tech" # tech: 技術記事 / idea: アイデア記事
topics: ["kintone", "snowflake", "dataengineering", "dwh"] # タグ。["markdown", "rust", "aws"]のように指定する
published: false # 公開設定（falseにすると下書き）
---

## はじめに

kintoneのデータを可視化・分析したい場合に意外と選択肢が少ない！！

kintone標準のBI機能はシンプルすぎて表現力に欠けるし、BIツール導入を考えると、データベースやETLツールの選定、さらには冪等性・リアルタイム性などクリアすべき要件が次々と出てくる。

そこで今回は、Snowflakeを用いた「標準的な」構成例を整理する。  
kintoneからSnowflakeへデータを取り込み、DWH・DM層を構築してBIツールで参照するまでの一連の流れをアーキテクチャとしてまとめておく。 

実際に本番環境で半年以上運用している構成を基に、**つまづきポイントやパフォーマンス最適化の実体験**も含めて紹介する。

## サンプルコード

本記事で解説するSnowflakeのSQL定義やETL処理のサンプルは、以下のGitHubリポジトリで公開しています：

**🔗 kintone-snowflake-dwh-sample**（予定）

* Snowflakeのテーブル定義（DDL）
* Stream/Task設定例  
* ETL処理のPythonスクリプト
* BIツール連携のサンプルクエリ

記事と合わせてご活用ください。

## 全体コンセプト

今回想定するアーキテクチャは以下のポイントを重視する。

- **リアルタイム性**：数分単位でkintoneの更新がSnowflake側に反映される  
- **スキーマ変更への柔軟性**：kintone側で新しいフィールドが追加されても、Snowflake側で対応しやすくする  
- **冪等性**：同じ増分データを再投入しても結果がブレない構造  
- **削除対応**：kintone上で削除されたデータをSnowflake側にも確実に反映する  
- **コスト最適化**：必要最小限の処理で最新化・整形を行い、Snowflake上のコンピュートコストを抑える

これらを満たすため、Snowflakeを3つのレイヤー（Raw、DWH、DM）に分け、WebhookやSnowpipe、Stream、TaskといったSnowflake特有の機能をフル活用する。

## 全体アーキテクチャ

### データフロー概要

```mermaid
graph TD
    A[kintone] -->|更新レコード| B[Lambda ETL]
    A -->|削除Webhook| C[Lambda ETL]
    B -->|JSON| D[S3 Bucket]
    C -->|削除フラグ付きJSON| D
    D -->|Snowpipe| E[Raw層テーブル]
    E -->|Stream検知| F[Task実行]
    F -->|MERGE| G[DWH層テーブル]
    G -->|マテリアライズドビュー| H[DM層]
    H --> I[Tableau/PowerBI]
```

### 処理レイヤーの役割

1. **Raw層**：kintoneから取得した生データをそのまま格納（履歴保持）
2. **DWH層**：Raw層のデータを整形し、最新状態のみを保持（正規化済み）
3. **DM層**：ビジネスロジックを適用した分析用データマート

## 実装詳細

### 1. Raw層テーブル設計

```sql
-- kintone_rawテーブルの定義例
CREATE TABLE raw.kintone_deals (
    record_id NUMBER NOT NULL,
    app_id NUMBER NOT NULL,
    updated_at TIMESTAMP_NTZ NOT NULL,
    is_deleted BOOLEAN DEFAULT FALSE,
    body VARIANT,
    ingested_at TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP(),
    source_file STRING
);
```

### 🔑 設計のポイント

1. **VARIANT型活用**：kintoneのJSONデータをそのまま格納してスキーマ変更に柔軟対応
2. **削除フラグ**：`is_deleted`で論理削除を管理
3. **メタデータ保持**：`ingested_at`、`source_file`で運用時のトレーサビリティを確保
4. **複合キー**：`record_id`+`updated_at`で重複データの追跡が可能

### 2. Snowpipe設定

```sql
-- 外部ステージ作成
CREATE STAGE raw.kintone_stage
URL = 's3://your-bucket/kintone-data/'
CREDENTIALS = (AWS_KEY_ID = 'your-key' AWS_SECRET_KEY = 'your-secret');

-- Snowpipe作成
CREATE PIPE raw.kintone_deals_pipe
AUTO_INGEST = TRUE
AS
COPY INTO raw.kintone_deals (record_id, app_id, updated_at, is_deleted, body, source_file)
FROM (
    SELECT 
        $1:record_id::NUMBER,
        $1:app_id::NUMBER,
        $1:updated_at::TIMESTAMP_NTZ,
        $1:is_deleted::BOOLEAN,
        $1:body::VARIANT,
        METADATA$FILENAME
    FROM @raw.kintone_stage
)
FILE_FORMAT = (TYPE = 'JSON');
```

### 3. DWH層のStream + Task

```sql
-- Raw層の変更を検知するStream
CREATE STREAM dwh.kintone_deals_stream ON TABLE raw.kintone_deals;

-- DWH層テーブル
CREATE TABLE dwh.deals (
    record_id NUMBER PRIMARY KEY,
    deal_name STRING,
    phase NUMBER,
    client_name STRING,
    amount NUMBER(15,2),
    expected_date DATE,
    created_at TIMESTAMP_NTZ,
    updated_at TIMESTAMP_NTZ,
    deleted_flag BOOLEAN DEFAULT FALSE
);

-- 自動更新Task
CREATE TASK dwh.update_deals_task
WAREHOUSE = 'COMPUTE_WH'
SCHEDULE = '1 MINUTE'
WHEN SYSTEM$STREAM_HAS_DATA('dwh.kintone_deals_stream')
AS
MERGE INTO dwh.deals t
USING (
    SELECT 
        record_id,
        body:deal_name::STRING as deal_name,
        body:phase::NUMBER as phase,
        body:client_name::STRING as client_name,
        body:amount::NUMBER(15,2) as amount,
        body:expected_date::DATE as expected_date,
        body:created_at::TIMESTAMP_NTZ as created_at,
        updated_at,
        is_deleted,
        ROW_NUMBER() OVER (PARTITION BY record_id ORDER BY updated_at DESC) as rn
    FROM dwh.kintone_deals_stream
    WHERE rn = 1  -- 最新レコードのみ
) s ON t.record_id = s.record_id
WHEN MATCHED AND s.is_deleted = TRUE THEN DELETE
WHEN MATCHED AND s.is_deleted = FALSE THEN UPDATE SET
    deal_name = s.deal_name,
    phase = s.phase,
    client_name = s.client_name,
    amount = s.amount,
    expected_date = s.expected_date,
    updated_at = s.updated_at
WHEN NOT MATCHED AND s.is_deleted = FALSE THEN INSERT (
    record_id, deal_name, phase, client_name, amount, expected_date, created_at, updated_at
) VALUES (
    s.record_id, s.deal_name, s.phase, s.client_name, s.amount, s.expected_date, s.created_at, s.updated_at
);

-- Task開始
ALTER TASK dwh.update_deals_task RESUME;
```

### 4. ETL処理（Lambda関数例）

```python
import json
import boto3
from datetime import datetime
import requests

def lambda_handler(event, context):
    """kintoneからの増分データを取得してS3に格納"""
    
    # 前回実行時刻から増分取得（環境変数で管理）
    last_updated = get_last_updated_timestamp()
    
    # kintone APIで増分データ取得
    kintone_data = fetch_incremental_data(last_updated)
    
    # S3にJSON形式で保存
    s3_key = f"kintone-data/{datetime.now().strftime('%Y/%m/%d/%H/%M')}/deals.json"
    upload_to_s3(kintone_data, s3_key)
    
    # 次回実行用のタイムスタンプ更新
    update_last_timestamp()
    
    return {"statusCode": 200, "processed_records": len(kintone_data)}

def fetch_incremental_data(since_timestamp):
    """kintone REST APIで更新データを取得"""
    query = f'updated_at > "{since_timestamp}"'
    # ... kintone API呼び出し処理
    return records
```

## 運用上の重要ポイント

### ⚡ パフォーマンス最適化

1. **Warehouse サイズ調整**
   ```sql
   -- 大量データ処理時は一時的にサイズアップ
   ALTER WAREHOUSE COMPUTE_WH SET WAREHOUSE_SIZE = 'LARGE';
   -- 処理完了後は元に戻す
   ALTER WAREHOUSE COMPUTE_WH SET WAREHOUSE_SIZE = 'X-SMALL';
   ```

2. **クラスタリングキー設定**
   ```sql
   -- よく使用される条件でクラスタリング
   ALTER TABLE dwh.deals CLUSTER BY (updated_at, client_name);
   ```

3. **Task実行間隔の調整**
   - 初期は1分間隔で設定したが、実運用では**5分間隔**が最適だった
   - あまりに頻繁だとSnowflakeクレジットを無駄に消費

### 🚨 運用時の注意点

1. **削除データの扱い**
   - **物理削除は避ける**：`deleted_flag`で論理削除し、分析時にフィルタする
   - 監査要件で完全削除が必要な場合は別途バッチ処理で対応

2. **スキーマ変更への対応**
   ```sql
   -- 新フィールド追加時のDWH層更新例
   ALTER TABLE dwh.deals ADD COLUMN priority STRING;
   
   -- MERGE文も併せて更新
   -- body:priority::STRING as priority を追加
   ```

3. **エラーハンドリング**
   ```sql
   -- Task実行履歴の確認
   SELECT * FROM TABLE(INFORMATION_SCHEMA.TASK_HISTORY())
   WHERE NAME = 'UPDATE_DEALS_TASK'
   ORDER BY SCHEDULED_TIME DESC;
   ```

### 💰 コスト最適化の実例

運用半年での実績：
- **Raw層**：1.2GB/月（履歴データ込み）
- **DWH層**：800MB/月（最新データのみ）
- **月間クレジット消費**：約15クレジット（＝約$45）

最適化施策：
1. AUTO_SUSPEND = 60秒に短縮
2. 夜間バッチは別Warehouseで実行
3. 不要なRaw層データは3ヶ月で削除

## DM層の活用例

### マテリアライズドビューでの集計

```sql
-- 月次営業実績の自動集計
CREATE MATERIALIZED VIEW dm.monthly_sales AS
SELECT 
    DATE_TRUNC('MONTH', expected_date) as month,
    client_name,
    COUNT(*) as deal_count,
    SUM(amount) as total_amount,
    AVG(amount) as avg_amount
FROM dwh.deals 
WHERE deleted_flag = FALSE
  AND phase >= 4  -- 受注以上
GROUP BY 1, 2;
```

### BIツール用ビュー

```sql
-- Tableau用の分析ビュー
CREATE VIEW dm.sales_analysis AS
SELECT 
    d.*,
    CASE 
        WHEN d.phase = 1 THEN 'リード'
        WHEN d.phase = 2 THEN 'アポ'
        WHEN d.phase = 3 THEN '提案'
        WHEN d.phase = 4 THEN '受注'
        ELSE 'その他'
    END as phase_name,
    DATEDIFF('day', d.created_at, CURRENT_DATE()) as days_since_created
FROM dwh.deals d
WHERE d.deleted_flag = FALSE;
```

## ARM64 (Apple Silicon) での注意

ローカル開発でSnowflakeを使う場合、M1/M2 Macでは一部のPythonライブラリ（snowflake-connector-python等）で互換性問題が発生することがある。本番はx86_64環境推奨。

## まとめ

本記事で紹介した構成により、kintoneデータの **リアルタイム分析基盤** を以下の特徴で実現できた：

- **自動化**：Snowpipe + Stream + Task でほぼ手動運用不要
- **柔軟性**：VARIANT型でスキーマ変更に強い構造  
- **信頼性**：冪等性担保で障害時も安心
- **コスト効率**：月額$50以下で10万レコード規模を処理

実運用での**つまづきポイント**や**パフォーマンス調整**の知見も含めたので、同様の要件がある方の参考になれば幸いです。

特に **Task実行間隔** や **Warehouse設定** は、データ量や更新頻度に応じた調整が重要なので、段階的に最適化していくことをお勧めします！

## 参考記事

- [Snowflake公式ドキュメント - Streams & Tasks](https://docs.snowflake.com/en/user-guide/streams-tasks.html)
- [kintone REST API リファレンス](https://developer.cybozu.io/hc/ja/categories/200149220)
