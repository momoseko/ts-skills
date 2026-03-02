---
name: bank-report
description: JCB ASEANイシュアー向けの取引データ分析と銀行提案書PPTXスライドの自動生成
version: 1.0.0
triggers:
  - bank report
  - イシュアーレポート
  - 提案書を作成
  - スライド生成
---

# Bank Report Skill

このSkillが呼び出されたら、以下のPhaseを順番に実行してください。

---

## Phase 1：パラメータ確認

ユーザーに以下を確認する（すでに明示されていればスキップ）：

| 項目 | デフォルト |
|------|-----------|
| 対象イシュアー名 | （必須） |
| 対象国 | （必須、例: THA） |
| 対象期間（YYYY年M月） | （必須） |
| 出力セクション | 1.1, 1.2, 2.1, 2.2（デフォルト） |

---

## Phase 2：Agentを使ったデータ収集

`JCB ASEAN/Issuer Proposal Agent` に対して、各セクションを**個別のメッセージ**で依頼する。

### 依頼メッセージのテンプレート

```
対象イシュアー：{issuer_name}（iss_lcid: {iss_lcid}）
対象国：{country_code}
対象期間：{year_month}（例：2026-01）

セクション {section_id} の分析を実行し、return_analysis で結果を返してください。
```

### 各セクションのiss_lcidの特定方法

iss_lcidが不明な場合、最初にAgentへ以下を問い合わせる：
```
{country_code}の{issuer_name}のiss_lcidを教えてください（imark_transaction_data_aseanを参照）
```

### 収集するセクション（デフォルト）

- `1.1` — イシュアー競合比較（市場シェア）
- `1.2` — 成長トレンド（前月比・前年同月比）
- `2.1` — カードタイプ別パフォーマンス
- `2.2` — カードグレード別分析

各セクションの `return_analysis` レスポンスを変数に保存する：
```python
results = {
    "1.1": { "section_id": "1.1", "title": "...", "data": [...], "insights": [...], "chart_type": "pie" },
    "1.2": { ... },
    ...
}
```

---

## Phase 3：スライド生成

**目的**: `/tmp/bank_report_data.json` を使って PowerPoint プレゼンテーションを生成

**実行**:

**必ず `Skill` ツールで `document-skills:pptx` を呼び出すこと**（python-pptx や Bash での独自実装は禁止）。
`document-skills:pptx` スキルを起動し、`/tmp/bank_report_data.json` のデータをもとに PowerPoint プレゼンテーションを作成する。

**デザイン仕様**:
- **テーマ**: Modern Light — 全スライド白背景
- **ブランドカラー**:
  - Primary: JCB Blue `#0052A5`（タイトル、主要チャート）
  - Secondary: JCB Red `#E60012`（アクセント、補助チャート）
  - Text: `#333333`
  - Light Gray: `#F5F5F5`（KPIカード背景）

**スライド構成**:

1. **表紙スライド**
   - タイトル: `{issuer_name}` を44pt bold blue センタリング
   - サブタイトル: `{country_name} Market Report` を24pt gray センタリング
   - 日付: `{period_start} - {period_end}` を18pt gray センタリング

2. **スライド2: 1.1 市場競合比較**
   - タイトル: "1.1 市場競合比較" を28pt bold blue
   - 左: ランキングテーブル（列: Rank, Issuer, Sales Amount, Market Share）、ヘッダー行をblue背景
   - 対象イシュアー行をlight gray背景でハイライト
   - 右: 市場シェア分布の円グラフ

3. **スライド3: 1.2 月次トレンド分析**
   - タイトル: "1.2 月次トレンド分析" を28pt bold blue
   - 上段: 3つのKPIカード（Total Sales, Avg Monthly, Growth Rate）— light gray背景、blue枠線
   - 下段: 月次売上トレンドの折れ線グラフ（red、円マーカー付き）

4. **スライド4: 2.1 マーチャントカテゴリー分析**
   - タイトル: "2.1 マーチャントカテゴリー分析" を28pt bold blue
   - 左: カテゴリーシェアの横棒グラフ（blue）
   - 右: 平均金額上位5件テーブル（red ヘッダー）

5. **スライド5: 2.2 利用国別分析**
   - タイトル: "2.2 利用国別分析" を28pt bold blue
   - 左: 国内/海外比較の円グラフ（blue/red）
   - 右: 海外上位国の横棒グラフ（red）

**出力先**: `/tmp/{IssuerSlug}_Report_{period_start}_{period_end}.pptx`

IssuerSlugはイシュアー名のスペースを`_`に置換し、英大文字に統一する（例: `CARDX_COMPANY_LIMITED`）。

**エラーハンドリング**: データファイルが存在しない場合は Phase 2 に戻る

---

## Phase 4：プレビュー

生成したPPTXを `mcp__tdx-studio__preview_document` で表示する。

---

## エラーハンドリング

| 状況 | 対応 |
|------|------|
| Agentがデータを返せない（テーブルに該当データなし） | そのセクションをスキップし、スライドに「データなし」プレースホルダーを挿入 |
| iss_lcidが特定できない | ユーザーに確認を求める |
| PPTXが生成されない | `document-skills:pptx` のスキルを明示的にロードして再試行 |
