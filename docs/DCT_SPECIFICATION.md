# Digital Customer Twin (DCT) — システム仕様書

> **Version 1.0 / 2026-04-17**  
> このドキュメントは設計の一次情報です。コードより先にこの仕様を読み、実装の根拠としてください。

---

## 1. システム概要

### 1.1 目的

保険パンフレット（PDF）に対して、AIが生成した消費者エージェントを使って「理解度・加入意向」をシミュレートし、課題箇所を特定・改善提案を生成することで、実際の営業前に資料品質を定量評価するPoC。

### 1.2 コアコンセプト

```
パンフレット（PDF）
    ↓ ページテキスト抽出 / 読み原稿
    ↓ AIエージェント（消費者）が閲覧・評価
    ↓ 不満・懸念を収集・分類
    ↓ AI専門家が改善提案
    ↓ エージェントが改善版を再評価
    → 加入率向上を測定
```

**「読み原稿」とは何か**

- **標準モード**: PDFから抽出したページテキストそのもの（= 担当者が読み上げ・説明する素材）
- **ノックモード**: 別途用意した `talk_script_v1.json`（= 担当者が使うセールストーク原稿）

### 1.3 2つのモード

| 項目 | Standard Mode | Knock Mode |
|------|--------------|-----------|
| 対象 | PDF資料の各ページ | セールストーク原稿（セクション単位） |
| 評価対象 | ページテキスト | 説明原稿（talk_script） |
| 改善の適用 | **しない**（提案のみ） | **する**（次バージョンに反映） |
| 繰り返し | なし（初期→再評価の1往復） | 最大Nラウンド（ver1→ver2→ver3…） |
| 用途 | 資料の課題発見・優先度付け | 説明原稿の品質向上サイクル |

---

## 2. Standard Mode パイプライン

### 2.1 処理フロー

```
Step 0: 入力データ読み込み
  ├── PDF → ページテキスト（読み原稿 ver1）
  ├── config.json → ページ別メタデータ（anxiety_modifier, complaint_weights）
  └── PDF → サムネイル画像（base64）

Step 1-1: 初期評価
  入力: ページテキスト × n人の消費者エージェント
  処理: 各エージェントが全ページを閲覧 → 理解度・加入意向を判定
  出力: 加入率（goal_rate）、4段階分布、不満コメント

Step 1-2: 不満分析
  入力: 全エージェントの不満コメント
  処理: カテゴリ分類・集計 → ワーストページ特定（上位3ページ）
  出力: 不満件数/ページ、カテゴリランキング、ワーストページリスト

Step 2: 専門家レビュー（ワースト3ページのみ）
  入力: ワーストページのテキスト + 不満コメント
  処理: 3専門家（保険営業 / UI/UX / コピーライター）が改善提案を生成
  出力: 専門家ごとに5提案 × 3専門家 × 3ページ = 最大45提案
  ※ この提案はStep3の評価に反映されるが、原稿は書き換えない

Step 3: 再評価
  入力: 同じページテキスト（改善は加味した文脈で評価）
  処理: 同一エージェントが再評価
  出力: 改善後の加入率、改善幅（+Xpt）
```

### 2.2 結果JSONスキーマ（standard）

```json
{
  "run_id": "run_YYYYMMDD_HHMMSS",
  "mode": "standard",
  "config": {
    "n": 30,
    "seed": 42,
    "adapter_type": "NullAdapter",
    "num_pages": 22
  },
  "brochure_pages": [
    {
      "page_num": 1,
      "title": "ページタイトル",
      "body": "ページ本文（先頭600文字）",
      "body_full": "ページ本文（先頭2000文字）",
      "anxiety_modifier": 0.05,
      "complaint_weights": {
        "商品構造に対する複雑性": 0.5,
        "専門用語に対する説明不足": 0.3
      }
    }
  ],
  "step1_1_initial": {
    "decision_distribution": { "goal_rate": 0.133, "distribution": {...} },
    "agent_summaries": [...]
  },
  "step1_2_complaints": {
    "total_complaints": 388,
    "classified_complaints": [
      { "page": 6, "agent_id": "A001", "category": "商品構造に対する複雑性", "comment": "..." }
    ],
    "category_ranking": [...],
    "page_complaint_counts": [...]
  },
  "step2_expert_improvements": [
    {
      "page_num": 19,
      "expert_name": "保険営業エキスパート",
      "suggestions": ["提案1", "提案2", "提案3", "提案4", "提案5"]
    }
  ],
  "step3_re_evaluation": {
    "decision_distribution": { "goal_rate": 0.433 }
  },
  "summary": {
    "initial_goal_rate": 0.133,
    "re_eval_goal_rate": 0.433,
    "improvement_pt": 0.300,
    "worst_pages": [19, 6, 20],
    "total_initial_complaints": 388
  },
  "page_thumbnails": { "1": "data:image/png;base64,...", "2": "..." }
}
```

---

## 3. Knock Mode パイプライン

### 3.1 処理フロー

```
前提: talk_script_v1.json（説明原稿）が必要

Round 1:
  ver1の説明原稿 → エージェント評価 → 不満収集 → 専門家改善提案 → ver2を生成・保存

Round 2:
  ver2の説明原稿 → エージェント評価 → 不満収集 → 専門家改善提案 → ver3を生成・保存

Round N（最終）:
  verNの説明原稿 → 最終評価 → サマリー出力

※ 閾値到達（全員加入）で早期終了
```

### 3.2 Knock Mode 結果JSONスキーマ（追加キー）

```json
{
  "mode": "knock",
  "script_versions": [
    {
      "version": 1,
      "sections": [
        {
          "section_id": 1,
          "title": "セクションタイトル",
          "content": "原稿テキスト",
          "complexity": 0.8,
          "jargon_density": 0.6,
          "risk_salience": 0.3,
          "source_pages": [1, 2],
          "improvements_applied": ["改善1", "改善2"]
        }
      ]
    }
  ],
  "knock_rounds": [...],
  "knock_summary": { "total_rounds": 3, "final_subscription_rate": 0.10 }
}
```

---

## 4. 資料マップ（flowmap）タブ仕様

### 4.1 概念

**「資料マップ」は何を見せるか**

資料（パンフレット）の流れに沿って、「エージェントが何に詰まり、専門家がどう改善を提案したか」を可視化する。標準モードとノックモードで表示データが異なるが、**3カラム構造は共通**。

### 4.2 共通レイアウト

```
┌─────────────────────────────────────────────────────────────────┐
│ ヘッダー: [P番号 or Section番号] タイトル  [不満X件] [改善済] [Nver] │
├──────────────┬──────────────────────┬───────────────────────────┤
│ 📄 資料ページ │ 📝 読み原稿           │ 🔴不満 / 🤖AI改善提案     │
│ (240px固定)  │ (1fr)                │ (1fr)                     │
│              │                      │                           │
│ PDFサムネイル │ ページ/原稿テキスト   │ 不満コメント（全件）       │
│ ページ番号   │ 展開/折りたたみ       │ カテゴリタグ付き           │
│ リスク指標   │ 複雑度・専門語・リスク │ ─────────────────────    │
│ カテゴリ重み │ メトリクスバッジ      │ AI改善提案（専門家別全件） │
└──────────────┴──────────────────────┴───────────────────────────┘
```

### 4.3 Standard Mode 表示仕様

**単位**: ページ（1 PDF page = 1 カード）  
**対象**: 全ページ（22ページなら22カード）

| カラム | データソース | 表示内容 |
|-------|------------|---------|
| 📄 資料ページ | `brochure_pages[n].page_num/title` + `page_thumbnails` | PDFサムネイル、P番号、タイトル |
| | `brochure_pages[n].anxiety_modifier` | リスク指標バッジ（高/中/低） |
| | `brochure_pages[n].complaint_weights` | カテゴリ別重みのプログレスバー |
| 📝 読み原稿 | `brochure_pages[n].body / body_full` | ページ本文（折りたたみ）= ver1のみ |
| | configから算出 | 複雑度・専門用語・リスク度バッジ |
| | `step1_2_complaints.classified_complaints` | 不満カテゴリサマリー |
| 🔴/🤖 右カラム | `step1_2_complaints.classified_complaints` | 全不満コメント（カテゴリタグ付き）|
| | `step2_expert_improvements`（ワースト3のみ） | 専門家3人 × 5提案（全件）|
| | ワースト3以外のページ | 「今回の改善対象外ページです」表示 |

**ヘッダーバッジ**:
- 不満件数バッジ（赤）: 常時表示
- 「改善提案あり」バッジ（緑）: ワースト3ページのみ
- verバッジ: **表示しない**（standard modeは1バージョン固定）

### 4.4 Knock Mode 表示仕様

**単位**: セクション（talk_scriptのsection単位）  
**対象**: 全セクション

| カラム | データソース | 表示内容 |
|-------|------------|---------|
| 📄 資料ページ | `script_versions[0].sections[n].source_pages` + `page_thumbnails` | 該当PDF全ページのサムネイル |
| | `script_versions[0].sections[n].page_role` | ページの役割ラベル |
| 📝 読み原稿 | `script_versions[*].sections[n].content` | バージョンタブ（ver1/ver2/ver3） |
| | | ver2以降は差分ハイライト（追加:緑/削除:赤） |
| | `.complexity/.jargon_density/.risk_salience` | メトリクスバッジ + デルタ表示 |
| | `.improvements_applied` | 適用された改善タグ |
| 🤖 右カラム | `knock_rounds[n].expert_improvements` | 専門家3人の提案 |
| | latestSec との差分 | 最終改善結果（数値変化） |

**ヘッダーバッジ**:
- 不満件数バッジ（赤）
- 「改善済」バッジ（緑）: メトリクスが変化したセクション
- verバッジ（黄）: バージョン数

---

## 5. ダッシュボードタブ一覧

| タブID | 名称 | 内容 |
|--------|------|------|
| `summary` | サマリー | KPIカード（加入率・改善幅・不満件数）、ビフォーアフターグラフ |
| `flowmap` | 資料マップ | **本仕様書4章参照** |
| `heatmap` | ヒートマップ | 不満カテゴリ × ページ/セクション の頻度ヒートマップ |
| `funnel` | ファネル | 「理解できない→理解したが不安→契約する」の4段階分布 |
| `agents` | エージェント | 個別エージェントのペルソナ・評価詳細 |

---

## 6. 商品オンボーディング手順

新しい保険商品をDCTで評価するための手順書。

### 6.1 必要ファイル

```
product/
└── {商品コード}/            例: FWD-2ndFloor/
    ├── {商品コード}.pdf                               ← PDF本体（必須）
    ├── {商品コード}_pages.json                        ← ページテキスト（自動生成）
    ├── {商品コード}_config.json                       ← ページ設定（手動作成・必須）
    └── talk_script_{商品コード}_v1.json               ← AI生成読み原稿（generate_talk_script.pyで自動生成）
```

### 6.2 ステップ1: config.jsonを先に作成

**⚠️ 重要**: PDFを触る前にconfig.jsonを作成する。ページタイトルと不安度設定は読み原稿の品質に直結する。

→ **§6.4「config.jsonを作成」を先に実施すること**

### 6.3 ステップ2: AI読み原稿を生成（必須）

保険PDFは縦書きレイアウトのため、生テキスト抽出は崩れる。**必ずGemini APIで読み原稿を生成する**。

```bash
# 品質チェックのみ（APIを使わない）
python tools/generate_talk_script.py \
  --product-dir product/FWD-2ndFloor \
  --product-code fwd2nd \
  --dry-run

# 本番生成（22ページで約1.5分）
python tools/generate_talk_script.py \
  --product-dir product/FWD-2ndFloor \
  --product-code fwd2nd \
  --delay 3.0

# 特定ページのみ再生成
python tools/generate_talk_script.py \
  --product-dir product/FWD-2ndFloor \
  --product-code fwd2nd \
  --pages 7,15-18 \
  --resume
```

**生成物**:
- `{product_code}_pages.json` — `script` フィールドに読み原稿を追加（元テキストも `text` で保持）
- `talk_script_{product_code}_v1.json` — Knock Mode用トークスクリプト

**品質チェック基準**（自動適用）:
| 条件 | 閾値 | 対処 |
|------|------|------|
| テキスト長 | 100文字以上 | NG: 再生成 |
| 日本語率 | 50%以上 | NG: 再生成 |
| ですます調 | 含む | NG: 再生成 |
| テキスト長上限 | 2000文字以下 | 警告のみ |

### 6.4 ステップ2（実際の作業順）: config.jsonを作成

各ページに以下を設定：

| フィールド | 説明 | 設定の目安 |
|-----------|------|-----------|
| `page_num` | ページ番号 | 1始まり |
| `title` | ページタイトル | PDFの見出しから |
| `anxiety_modifier` | 不安喚起度 | 0.0〜0.15（高いほど不満が出やすい）|
| `complaint_weights` | 不満カテゴリの重み | 合計1.0になるように設定 |

**anxiety_modifierの目安**:
- `0.00〜0.03`: 表紙・アフターサービス等、低不安ページ
- `0.04〜0.07`: 商品説明・税務等、中程度
- `0.08〜0.12`: Q&A・特別勘定・リスク説明、高不安
- `0.13〜0.15`: 解約費用・元本割れリスク、最高不安

**complaint_weightsに使える不満カテゴリ**:

| カテゴリ名 | 意味 |
|-----------|------|
| `商品構造に対する複雑性` | 商品の仕組みが複雑で理解しにくい |
| `専門用語に対する説明不足` | 金融・保険用語の解説がない |
| `途中解約に対する元本割れリスク` | 解約すると損をする可能性への不安 |
| `ターゲット層に対する配慮不足` | 高齢者など対象者への配慮が足りない |
| `リスク特性に対する説明の欠如` | リスクの説明が不十分 |
| `運用実態に対する不透明感` | どう運用されるかわからない |
| `実績データに対する信頼性不足` | シミュレーションや実績の信憑性 |
| `資産拘束に対する流動性不安` | お金が使えない期間への不安 |
| `図表表示に対する視認性不足` | グラフ・図表が見づらい |

**config.json サンプル**:

```json
{
  "product_name": "商品正式名称",
  "product_display_name": "表示名",
  "product_code": "fwd2nd",
  "issuer": "発行会社名",
  "product_type": "商品種別",
  "page_config": [
    {
      "page_num": 1,
      "title": "ご契約後のアフターサービス",
      "anxiety_modifier": 0.03,
      "complaint_weights": {
        "ターゲット層に対する配慮不足": 0.4,
        "商品構造に対する複雑性": 0.3,
        "図表表示に対する視認性不足": 0.3
      }
    }
  ]
}
```

### 6.5 ステップ4: Standard Mode で実行

```bash
# 基本実行（30エージェント、ビルトインペルソナ使用）
python -m src.main run \
  --mode standard \
  --n 30 \
  --seed 42 \
  --adapter null \
  --persona __builtin__ \
  --product-dir "product/FWD-2ndFloor" \
  --product-code fwd2nd

# 出力先: ./outputs/run_YYYYMMDD_HHMMSS/
#   dashboard.html  ← メインダッシュボード
#   results.json    ← 全データ
#   summary.json    ← サマリーのみ
#   report.md       ← Markdownレポート
#   responses.csv   ← エージェント応答CSV
```

### 6.5 ステップ4: GitHub Pagesにデプロイ

```bash
# 1. dashboardをdocsフォルダにコピー
cp outputs/run_YYYYMMDD_HHMMSS/dashboard.html \
   /path/to/dct-dashboard/docs/{product_code}_standard.html

# 2. index.htmlに新カードを追加（手動）

# 3. git push
cd /path/to/dct-dashboard
git add docs/
git commit -m "feat: add {商品名} standard mode dashboard"
git push origin main
```

### 6.6 ステップ5（任意）: Knock Mode で原稿改善

Knock Modeには別途 `talk_script_v1.json` が必要。

```bash
# 実行前に talk_script_{product_code}_v1.json を配置
python -m src.main run \
  --mode knock \
  --n 10 \
  --seed 42 \
  --adapter null \
  --persona __builtin__ \
  --product-dir "product/FWD-2ndFloor" \
  --product-code fwd2nd \
  --max-rounds 3
```

---

## 7. ペルソナ仕様

### 7.1 ビルトインアーキタイプ（PA001〜PA030）

30種のペルソナアーキタイプが組み込み済み。`--persona __builtin__` で使用。

| 属性 | 説明 |
|------|------|
| `persona_id` | PA001〜PA030 |
| `name` | ペルソナ名（例: 慎重派の定年退職者） |
| `typical_concerns` | よく持つ懸念（不満コメント生成に使用） |
| `risk_tolerance` | リスク許容度（低:0.0〜高:1.0） |
| `financial_literacy` | 金融知識（低:0.0〜高:1.0） |
| `liquidity_need` | 流動性ニーズ（低:0.0〜高:1.0） |

### 7.2 SNSコーパス（97エントリ）

実際の保険相談SNSコメントをベースにした不満サンプル集。NullAdapterが不満コメントを生成する際に参照。

---

## 8. CLIリファレンス

```bash
python -m src.main run [OPTIONS]

# モード
--mode standard        # 標準モード（デフォルト）
--mode knock           # ノックモード

# エージェント設定
--n 30                 # エージェント数（推奨: standard=30, knock=10〜100）
--seed 42              # 乱数シード（再現性確保のため固定）

# LLMアダプタ
--adapter null         # ルールベース（高速・無料、推奨）
--adapter gemini       # Gemini API（低速・有料、高品質）

# ペルソナ
--persona __builtin__  # ビルトイン30アーキタイプ（推奨）
--persona path/to/personas.json  # カスタムペルソナファイル

# 商品設定
--product-dir "product/FWD-2ndFloor"   # 商品ディレクトリ
--product-code fwd2nd                   # 商品コード（ファイル名のプレフィックス）

# その他
--max-rounds 3         # Knock Modeの最大ラウンド数
--data-dir ./data      # デフォルトデータディレクトリ（商品ディレクトリ未指定時）
--output-dir ./outputs # 出力先ディレクトリ
```

---

## 9. ダッシュボード index.html カード追加ガイド

`docs/index.html` に新商品を追加する際のカードテンプレート：

```html
<!-- セクションを追加 -->
<h2 class="section-title">{商品表示名} - {商品正式名}</h2>
<div class="cards">
  <div class="card">
    <a href="{product_code}_standard.html">
      <span class="card-product product-{css_class}">{商品略称}</span>
      <span class="badge badge-standard">Standard</span>
      <div class="card-title">Standard Mode - {n} Agents × {pages} Pages</div>
      <div class="card-meta">
        <strong>Agent:</strong> {n} / <strong>Seed:</strong> {seed}<br>
        <strong>Persona:</strong> {ペルソナ説明}<br>
        <strong>Result:</strong> {初期率}% → {改善後率}% ({改善幅}pt)<br>
        <strong>不満件数:</strong> {総不満件数}件 / {pages}ページ全カバー<br>
        <strong>Top Issue:</strong> {トップカテゴリ} {割合}%<br>
        <strong>Worst Pages:</strong> P{X}・P{Y}・P{Z}<br>
        <strong>Date:</strong> {YYYY-MM-DD}
      </div>
      <div class="card-arrow">→</div>
    </a>
  </div>
</div>
```

**CSS クラス追加（商品ごとに1色）**:
```css
.product-fwd  { background: #f59e0b33; color: #fbbf24; }  /* FWD: 金色 */
.product-ppn  { background: #7c3aed33; color: #a78bfa; }  /* PPN: 紫 */
.product-pj   { background: #0ea5e933; color: #67e8f9; }  /* PJ: 水色 */
/* 新商品: 追加 */
```

---

## 10. 既知の制約

| 制約 | 詳細 |
|------|------|
| Standard Modeの改善適用 | 専門家提案はStep3に「ヒント」として渡されるが、原稿テキスト自体は変わらない |
| ワースト3ページのみ改善提案 | 全ページに改善提案は生成されない（コスト・速度のトレードオフ） |
| NullAdapterの限界 | ルールベースのため、コメントのバリエーションは限定的 |
| Gemini APIの速度 | delay=13秒/call × エージェント数 × ページ数 × calls = 数時間 |
| HTMLファイルサイズ | サムネイル22枚 × base64 ≈ 1.5MB。22ページ超の商品は要注意 |

---

## 11. ファイル構成

```
src/
├── main.py                    # CLI エントリーポイント
├── scoring/
│   └── journey.py             # JourneyPipeline（標準・ノック両モード）
├── reporting/
│   └── report_html.py         # HTMLダッシュボード生成
├── agents/
│   ├── consumer.py            # 消費者エージェント
│   └── expert.py              # 専門家エージェント
├── persona/
│   ├── builtin_archetypes.py  # ビルトイン30アーキタイプ定義
│   └── persona.py             # ペルソナ読み込み
├── llm/
│   └── adapter.py             # NullAdapter / GeminiAdapter
├── script/
│   └── talk_script.py         # TalkScript（Knock Mode用）
└── utils/
    └── pdf_thumbnails.py      # PDF→base64サムネイル生成

product/
├── FWD-2ndFloor/              # FWD 円建一時払変額年金
│   ├── FWD-2ndFloor.pdf
│   ├── fwd2nd_pages.json
│   ├── fwd2nd_config.json
│   └── talk_script_fwd2nd_v1.json  （未作成）
├── PPN/                       # プレミアパートナー
└── PJ/                        # プレミアジャーニー

docs/                          # GitHub Pages
├── index.html                 # ポータル
├── fwd2nd_standard.html
├── ppn_100agents.html
├── ppn_10agents.html
├── pj_knock.html
└── pj_standard.html

docs/
└── DCT_SPECIFICATION.md       # 本ドキュメント
```
