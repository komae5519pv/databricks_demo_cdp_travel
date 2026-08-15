# databricks_demo_cdp_travel

CustomerLake デモ環境用のデータ準備キットです。  
架空の航空会社「**DBX Airlines**」の顧客行動データを Unity Catalog 上にデモテーブルとして作成します。

---

## 前提条件

| 項目 | 要件 |
| --- | --- |
| コンピュート | サーバーレスコンピュート（Python + SQL） |
| Unity Catalog | カタログ作成権限、または既存カタログへのスキーマ作成権限 |

---

## ディレクトリ構成

```
databricks_demo_cdp_travel/
├── README.md               ← 本ファイル
├── 00_config               ← 【唯一の編集対象】カタログ名・スキーマ名を設定（直接実行不要）
├── 01_create_tables        ← DDL + データ投入（これだけ Run All すれば完了）
└── _data/
    ├── customers.csv.gz
    ├── flight_activity.csv.gz
    ├── bookings.csv.gz
    ├── ancillary_purchases.csv.gz
    ├── app_sessions.csv.gz
    └── customer_support_tickets.csv.gz
```

> **ℹ️ CSV は gzip 圧縮済み**（合計 266MB → 60MB）。Spark が透過的に読み込むため、解凍作業は不要です。

---

## セットアップ手順

### 1. `00_config` を編集

使用するカタログ名とスキーマ名を設定してください（この2行だけ）。

```python
catalog_name = "your_catalog"    # ← 変更
schema_name  = "cdp_travel"      # ← 必要に応じて変更
```

### 2. `01_create_tables` を Run All

`01_create_tables` を開いて全セル実行するだけで完了です。  
内部で `%run ./00_config` を呼び出すため、`00_config` を個別に実行する必要はありません。

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

---

## データの特徴

### 言語

| 区分 | フィールド例 | 理由 |
| --- | --- | --- |
| 日本語 | cabin、booking_channel、product_name、サポートチケット各種 | 業務画面・顧客対応で使われるデータ |
| 英語 | flight_number、空港コード、fare_class、loyalty_tier | 国際標準・システムコード |

### 顧客分布

約80%が国内顧客（日本語氏名・+81電話番号・JP国コード）、約20%が海外顧客。

### ティア別傾向

| 指標 | Platinum | Gold | Silver | Member |
| --- | --- | --- | --- | --- |
| 平均運賃 | ¥3,432 | ¥2,532 | ¥1,983 | ¥1,551 |
| ビジネスクラス率 | 60% | 30% | 11% | 2% |
| 離反リスクスコア | 0.11 | 0.16 | 0.25 | 0.50 |

### 主要ディメンションの分布

**travel_purpose**（9カテゴリ）

| カテゴリ | 割合 |
| --- | --- |
| Business | ~30% |
| Leisure | ~19% |
| Mixed | ~11% |
| Family Vacation | ~10% |
| Weekend Getaway | ~10% |
| Visiting Friends & Relatives | ~8% |
| Honeymoon | ~5% |
| Adventure | ~4% |
| Special Event | ~3% |

**home_airport**（NYC圏）

| 空港 | 人数 |
| --- | --- |
| JFK | ~2,170 |
| EWR | ~1,300 |
| LGA | ~870 |

**region_focus**: 欧州路線予約実績のあるユーザーは「国際」に偏りあり。  
**marketing_consent_status**: 同意済み / 未同意 / 保留 の3値。  
**欧州目的地**: LHR・CDG が主要。LIS（リスボン）便は存在しない（新路線想定）。

### その他

- サポートチケットの約40%が解決済み
- FK整合性 100%（孤立レコードなし）

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
