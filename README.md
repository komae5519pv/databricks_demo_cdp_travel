# databricks_demo_cdp_travel

Customer Data Platform（CDP）デモ環境を Unity Catalog 上に構築するノートブック＋データセットです。  
架空の航空会社「**DBX Airlines**」の顧客行動データを題材に、カスタマーレイクの分析基盤を再現します。

---

## 前提条件

| 項目 | 要件 |
| --- | --- |
| Databricks ワークスペース | AWS / Azure / GCP いずれも可 |
| コンピュート | サーバーレス SQL または汎用クラスタ（DBR 14.0+） |
| Unity Catalog | カタログ作成権限、または既存カタログへのスキーマ作成権限 |
| 言語 | Python / SQL |

---

## ディレクトリ構成

```
databricks_demo_cdp_travel/
├── README.md               ← 本ファイル
├── 00_config               ← 【唯一の編集対象】カタログ名・スキーマ名を設定
├── 01_create_tables        ← DDL（空テーブル作成 + PK/FK + PIIタグ）
├── 02_insert_data          ← _data/ の CSV をテーブルに投入
└── _data/
    ├── customers.csv
    ├── flight_activity.csv
    ├── bookings.csv
    ├── ancillary_purchases.csv
    ├── app_sessions.csv
    └── customer_support_tickets.csv
```

---

## セットアップ手順

### 1. `00_config` を編集

使用するカタログ名とスキーマ名を設定してください（この2行だけ）。

```python
catalog_name = "your_catalog"    # ← 変更
schema_name  = "cdp_travel"      # ← 必要に応じて変更
```

### 2. ノートブックを順番に実行

```
00_config         → カタログ・スキーマの USE 設定
01_create_tables  → 6テーブルの DDL 実行 + PII タグ付与
02_insert_data    → CSV データをテーブルに INSERT OVERWRITE
```

> **冪等性あり**: 何度実行しても同じ結果になります（INSERT OVERWRITE）。

---

## 作成されるリソース

### スキーマ

`<catalog_name>.<schema_name>`（デフォルト: `cdp_travel`）

### テーブル一覧

| # | テーブル名 | 行数 | PK | FK | 概要 |
| --- | --- | ---: | --- | --- | --- |
| 1 | customers | 62,446 | user_id | — | 旅行者プロファイル（ロイヤルティ・離反スコア含む） |
| 2 | flight_activity | 2,258,844 | flight_id | — | フライト運航記録（搭乗率・遅延情報） |
| 3 | bookings | 493,827 | booking_id | user_id → customers, flight_id → flight_activity | 予約トランザクション |
| 4 | ancillary_purchases | 119,452 | purchase_id | user_id → customers, booking_id → bookings | 付帯購入（手荷物・座席指定等） |
| 5 | app_sessions | 312,230 | event_id | user_id → customers | アプリ/Web/Push イベントストリーム |
| 6 | customer_support_tickets | 12,490 | ticket_id | user_id → customers | 顧客サポートケース |

### Unity Catalog タグ

| タグキー | 値 | 対象カラム |
| --- | --- | --- |
| is_pii | yes | customers.first_name, last_name, email, phone |

---

## データの特徴

### 言語

| 区分 | フィールド例 | 理由 |
| --- | --- | --- |
| 日本語 | cabin、booking_channel、product_name、サポートチケット各種 | 業務画面・顧客対応で使われるデータ |
| 英語 | flight_number、空港コード、fare_class、loyalty_tier | 国際標準・システムコード |

### 顧客分布

約80%が国内顧客（日本語氏名・+81電話番号・JP国コード）、約20%が海外顧客。

### 分析的な傾向

| 指標 | Platinum | Gold | Silver | Member |
| --- | --- | --- | --- | --- |
| 平均運賃 | ¥3,432 | ¥2,532 | ¥1,983 | ¥1,551 |
| ビジネスクラス率 | 60% | 30% | 11% | 2% |
| 離反リスクスコア | 0.11 | 0.16 | 0.25 | 0.50 |

サポートチケットの約40%が解決済み。FK整合性 100%（孤立レコードなし）。

---

## 想定ユースケース

- CDP / カスタマーレイクのデモ・PoC
- AI/BI ダッシュボードのプロトタイピング
- Unity Catalog ガバナンス（PII タグ・ABAC）のデモ
- Genie Space / AI Query によるセルフサービス分析
- セグメンテーション・離反予測モデルの学習データ

---

## 注意事項

- PK/FK は Unity Catalog の **informational constraint**（非強制）として定義されています
- `01_create_tables` は `CREATE TABLE IF NOT EXISTS` のため、既存テーブルを上書きしません。再作成する場合は事前に DROP してください
- CSV データはデモ用の合成データです。実データは含まれていません
