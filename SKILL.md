---
name: x-timeline-curator
description: XのタイムラインをAIエージェントで多軸評価・キュレーションし、ノイズを排除した信頼性の高い情報だけを抽出するバッチ処理スキル。「Xのタイムラインを絞りたい」「情報収集を効率化したい」「タイムラインをAIで分類・評価したい」「Xの滞留時間を減らして情報だけ取りたい」といった要望があった場合に使用する。滞留型UIを避けて必要な情報だけを効率的に取得したいエンジニア・リサーチャー向け。
---

# X Timeline Curator

Xの投稿を外部から取得し、AIエージェントで多軸評価して、ノイズを排除した信頼性の高い情報を抽出するバッチ処理システム。

## 設計思想

### 解きたい課題

Xは「最新情報」「多様な人の考え」「議論」を得る場として価値がある一方、滞留設計(無限スクロール・通知バッジ・For Youの時間減衰)がユーザーの目的と衝突する。**情報取得だけを切り出し、消費は自分のペースで行う**ための仕組みが必要。

### データソースの設計方針

- **Following タイムラインは使わない**: Following と List はともに「あらかじめ設定されたアカウント範囲の投稿」であり役割が重複する。List に一本化する。
- **List を信頼アカウントの管理台帳として育てる**: キュレーション結果の蓄積先。手動で追加・削除を管理する。
- **検索・トレンドで未知を補う**: List だけでは情報が偏り新規発見ができない。キーワード検索・トレンド起点の検索で未フォロー圏からの発見を担保する。
- **For You は使わない**: 公式 API が存在しない。Grok による UI カスタマイズに任せる。

### 制御の2レベル

個々の投稿の評価と、全体の分布・質の評価は別レイヤで行う。

```
【メタ制御ループ】
ソース設定(sources.json)
      ↓
投稿取得 → 個別評価 → 全体評価エージェント
                             ↓
                       分布・質レポート出力
                       (人間がソース設定を見直す判断材料)
      ↑_______________________________________________
```

全体評価は「何が足りないか・偏っているか」を可視化するためのもの。改善アクション(検索 KW の変更・List 追加削除)は人間が判断して `sources.json` を編集する。

---

## 全体アーキテクチャ

```
[手動実行 CLI]
      ↓
[1. sources.json 読み込み]
      ↓
[2. 取得: List + 検索 + トレンド]  ← xmcp 経由
      ↓
[3. 前処理: RT展開・スレッド結合・重複排除・URL展開]
      ↓
[4. 個別評価 Stage1: ノイズフィルタ(Haiku)]
      ↓ 3〜5割削減
[5. 個別評価 Stage2: トピック分類 + 信頼性スコア(Sonnet)]
      ↓
[6. 全体評価エージェント: 分布・質・偏りの集計と診断(Sonnet)]
      ↓
[7. 出力: 個別レポート + 全体評価レポート]
```

---

## ソース設定 (`config/sources.json`)

システムの「現在の設定状態」を持つ唯一のファイル。ここを編集することで次回実行の動作が変わる。

```json
{
  "lists": [
    {
      "id": "LIST_ID_HERE",
      "name": "AI core",
      "note": "主力リスト",
      "members": [
        {
          "username": "simonw",
          "user_id": "xxx",
          "status": "active",
          "added_at": "2026-04-18",
          "avg_trust_score_recent": 4.2,
          "watch_since": null,
          "archive_since": null
        }
      ]
    }
  ],
  "search_keywords": {
    "en": ["LLM agent", "Claude Anthropic", "google cloud"],
    "ja": ["AI エージェント", "LLM"]
  },
  "author_seeds": ["karpathy", "simonw", "GergelyOrosz"],
  "trend": {
    "woeid": 23424856,
    "enabled": false,
    "relevance_topics": ["AI", "tech", "cloud", "LLM"],
    "note": "xmcp の OAuth2.0 対応後に enabled: true にする"
  },
  "thresholds": {
    "hours": 24,
    "noise_rate_warn": 0.2,
    "primary_source_rate_target": 0.2,
    "min_topic_coverage_rate": 0.15,
    "trust_score_highlight": 4,
    "engagement_score_weights": {
      "like": 1,
      "retweet": 3,
      "reply": 2
    }
  }
}
```

| フィールド               | 役割                                         | 変更タイミング                               |
| ------------------------ | -------------------------------------------- | -------------------------------------------- |
| `lists[].members`        | アカウントの状態管理台帳(status/スコア履歴)  | エージェントが自動更新。物理削除のみ人間確認 |
| `search_keywords.en/ja`  | 言語別キーワード検索                         | 特定カテゴリが薄いと全体評価で判明したとき   |
| `author_seeds`           | 著名人起点層のターゲット                     | 信頼できる著者を追加・削除するとき           |
| `trend.enabled`          | トレンド層の有効/無効                        | xmcp の OAuth2.0 対応後に `true` にする      |
| `trend.relevance_topics` | トレンドのフィルタ条件(enabled時)            | トレンド由来のノイズが多いとき               |
| `thresholds`             | 全体評価の警告・目標値、エンゲージメント重み | 基準を調整したいとき                         |

---

## データ取得レイヤ (`fetch.py`)

### xmcp 経由で使うツール

| ソース            | xmcp ツール                         | 役割                       | 状態          |
| ----------------- | ----------------------------------- | -------------------------- | ------------- |
| List タイムライン | `getListsPosts`                     | 既知アカウント群の動向     | ✅ 利用可     |
| キーワード検索    | `searchPostsRecent`                 | 未フォロー圏からの新規発見 | ✅ 利用可     |
| 著名人起点        | `getUsersPosts`                     | 信頼著者の投稿を直接取得   | ✅ 利用可     |
| トレンド取得      | `getTrendsByWoeid`                  | 今日話題のトピック把握     | ⚠️ 保留(後述) |
| トレンド→検索     | `searchPostsRecent`(トレンド KW で) | トレンドを起点に深掘り     | ⚠️ 保留(後述) |

### 著名人起点層の取得方法

`searchPostsRecent` の `from:xxx` クエリは24hウィンドウで空振りが発生しうる(投稿頻度依存)。`getUsersPosts` で直接取得する方が安定する。

```
sources.json の author_seeds に対して
getUsersPosts(user_id, max_results=20) を並列実行
      ↓
個別評価パイプラインへ
```

`author_seeds` のユーザー名 → user_id 変換は `getUsersByUsernames` で事前に解決し `cache/author_ids.json` にキャッシュする。

### トレンド→検索の連携フロー(⚠️ 現在保留)

`getTrendsByWoeid` は OAuth 2.0 App-Only 認証が必要。xmcp は現時点で OAuth 2.0 未対応(PR 対応待ち)のため、このフローは一時的に無効化している。

xmcp が OAuth 2.0 に対応次第、以下のフローを有効化する:

```
getTrendsByWoeid(woeid=23424856)
      ↓
トレンド一覧取得
      ↓
relevance_topics に関連するトレンドだけ AI が選別
      ↓
searchPostsRecent で各トレンド KW を検索
      ↓
個別評価パイプラインへ
```

それまでの代替として `sources.json` の `search_keywords` に手動でトレンド系 KW を追加することで部分的にカバーする。

### 取得パラメータ

```python
tweet_fields = "created_at,author_id,conversation_id,referenced_tweets,entities,public_metrics"
expansions   = "author_id,referenced_tweets.id,attachments.media_keys"
user_fields  = "name,username,verified,description,public_metrics"
start_time   = now() - timedelta(hours=sources["thresholds"]["hours"])
```

### レート制限対策

- 429 に対する指数バックオフ
- 取得済み `tweet_id` を `cache/fetched_ids.json` にキャッシュして重複取得を回避
- pagination は `next_token` がなくなるまで追う

---

## 前処理レイヤ (`preprocess.py`)

評価コスト削減のための正規化。2〜3割削減が目安。

| 処理           | 内容                                                                               |
| -------------- | ---------------------------------------------------------------------------------- |
| RT 正規化      | 素の RT は `referenced_tweets` を展開し元投稿に置換。同一投稿の複数 RT を1件に統合 |
| 引用 RT 処理   | コメント部と元投稿を両方保持(コンテキスト評価に必要)                               |
| スレッド結合   | `conversation_id` + 同一著者の連投を1件に結合                                      |
| URL 展開       | `entities.urls[*].expanded_url` で短縮 URL を展開(信頼性判定で使用)                |
| 重複排除       | 同一 `tweet_id`・完全一致テキストを除去                                            |
| ソースタグ付与 | `source: "list" / "search" / "trend"` を各投稿に付与(全体評価で使用)               |

---

## 個別評価: Stage 1 ノイズフィルタ (`evaluate.py`)

### モデル

`claude-haiku-4-5-20251001` — 低コスト・高速

### 方針

**保守的に判定する**。明らかなノイズだけを弾く。迷ったら通過させる。ここで削りすぎると多様性が死ぬ。

### プロンプト

```
あなたはXタイムラインのノイズフィルタです。
以下の投稿を読み、後段の詳細評価に通すべきか判定してください。

【除外すべき投稿】
- エンゲージメント釣り(「RTで○○プレゼント」「いいねで△△」)
- 炎上・レスバ(感情的応酬、個人攻撃が主体)
- 宣伝・広告(商品/サービスの一方的告知、アフィリエイト)
- 中身のない感想・定型ポエム(「今日も頑張ろう」系)

【通過させる投稿】
- 技術的な知見、解説、ニュース、議論の提起
- 判断に迷うもの → 通過させる

【出力】JSON:
{ "pass": true/false, "reason": "一言で判定理由" }

投稿: {post_text}
```

期待削減率: 30〜50%

---

## 個別評価: Stage 2 多軸評価 (`evaluate.py`)

### モデル

`claude-sonnet-4-6` — 構造化出力の精度と一貫性を重視

### プロンプト

```
以下のXの投稿を3つの観点で評価してください。

【1. トピック分類】1〜3個選択:
- ai_llm       : AI/LLM/エージェント技術
- cloud_infra  : クラウド/インフラ/開発ツール
- jp_tech_biz  : 日本のビジネス/技術ニュース
- software_eng : 一般的ソフトウェアエンジニアリング
- research     : 研究・論文
- industry_news: 業界動向・M&A・採用
- opinion      : 意見・考察
- meta         : 業界メタ話・ポエム寄り
- other        : その他

【2. 信頼性・一次情報度】1〜5スコア:
5: 当事者/専門家による一次情報(自社発表・自分の実験結果・実体験)
4: 引用・リンクつきで検証可能な二次情報
3: 引用なしだが発信者の専門性から裏付けられる解説
2: 伝聞ベース・引用元不明の解説
1: 憶測・個人の感想のみ

【3. 一言サマリ】30〜50字の日本語要約

【出力】JSON:
{
  "topics": ["ai_llm"],
  "trust_score": 4,
  "trust_reason": "論文リンクつき著者による解説",
  "summary": "...",
  "is_primary_source": true
}

投稿本文: {post_text}
著者名: {author_name} (@{username})
Bio: {author_bio}
Verified: {verified}
フォロワー数: {follower_count}
展開済みURL: {expanded_urls}
```

発信者情報・URL を含めることが信頼性判定の精度に直結する。

---

## 全体評価エージェント (`aggregate.py` + `meta_evaluate.py`)

個別評価の結果を集計し、今回の収集全体の分布・質・偏りを診断する独立したエージェント。**改善アクションの提案は行わない。現状把握に徹する。**

### 集計指標 (`aggregate.py`)

```python
{
  "period": "2026-04-18",
  "window_hours": 24,
  "counts": {
    "fetched": 1432,
    "after_preprocess": 1100,
    "stage1_passed": 680,
    "stage2_evaluated": 680
  },
  "topic_distribution": {
    "ai_llm": 0.42,
    "cloud_infra": 0.11,
    "jp_tech_biz": 0.08,
    "other": 0.39
  },
  "quality": {
    "avg_trust_score": 3.1,
    "primary_source_rate": 0.13,
    "noise_rate": 0.19
  },
  "by_source": {
    "list":   { "count": 380, "avg_trust": 3.8, "engagement_top20_threshold": 1200 },
    "search": { "count": 220, "avg_trust": 2.5, "engagement_top20_threshold": 45 },
    "author": { "count":  32, "avg_trust": 4.1, "engagement_top20_threshold": 380 },
    "trend":  { "count":   0, "avg_trust": null, "note": "OAuth2.0対応待ち" }
  },
  "engagement_sampling": {
    "note": "エンゲージメントスコアはソース別に独立して計算する。List層に大規模アカウントが混入しても他層の閾値が歪まない",
    "formula": "like*1 + retweet*3 + reply*2",
    "per_source_top20_pct": true
  },
  "high_trust_authors": [
    { "username": "xxx", "count": 3, "avg_trust": 4.7, "in_list": false }
  ]
}
```

### 全体評価エージェントプロンプト (`meta_evaluate.py`)

```
あなたは情報収集システムのアナリストです。
以下の収集結果サマリを読み、今回の収集の全体的な状況を診断してください。

ユーザーの関心領域: AI/LLM、クラウド/インフラ、日本のビジネス技術
目標値:
  - 関心カテゴリ各 20% 以上のカバレッジ
  - ノイズ率 20% 以下
  - 一次情報率 20% 以上

【収集結果サマリ】
{aggregate_json}

【出力】JSON:
{
  "overall_score": "good | acceptable | needs_attention",
  "diagnosis": "全体状況を3〜5文で。数値を引用しながら具体的に",
  "strengths": ["良かった点を列挙"],
  "concerns": [
    {
      "area": "問題領域(例: cloud_infra カバレッジ)",
      "detail": "具体的な数値と何が問題かの説明",
      "hint": "考えられる原因(改善アクションは提案しない)"
    }
  ],
  "notable_authors": [
    {
      "username": "xxx",
      "reason": "List未登録だが今週3件 trust_score=5"
    }
  ]
}
```

---

## 出力レイヤ (`report.py`)

### ファイル構成

```
out/
├── {run_id}.md          # 個別投稿レポート(人間が読む)
├── {run_id}.json        # 個別投稿データ(プログラマブル)
├── {run_id}_meta.json   # 全体評価結果
└── {run_id}_meta.md     # 全体評価サマリ(人間が読む)
```

### 個別レポート (`{run_id}.md`)

```markdown
# Timeline Digest 2026-04-18

取得: 1432件 → Stage1通過: 680件

## 🌟 一次情報ハイライト (trust_score=5, 8件)

- [@karpathy] 新アーキテクチャの実験結果を公開
  https://x.com/karpathy/status/1234567890

## AI/LLM — 285件

### trust_score >= 4 (52件)

- [@simonw] LangChain 新エージェント API の解説
  https://x.com/simonw/status/1234567891

### trust_score 2-3 (180件)

...

## クラウド/インフラ — 75件

...
```

投稿 URL は `https://x.com/{username}/status/{tweet_id}` の固定フォーマットで生成する。取得時に author の `username` と `tweet_id` が揃っていれば API 追加呼び出しは不要。

### 全体評価レポート (`{run_id}_meta.md`)

```markdown
# 全体評価 2026-04-18

**総合評価**: needs_attention

## 診断

cloud_infra が11%と目標の20%を大きく下回っている。
一次情報率も13%と目標未達。ノイズ率は19%で許容範囲内。

## 良かった点

- ai_llm は42%と十分なボリューム
- List 由来の平均 trust_score が3.8と高品質

## 懸念点

### cloud_infra カバレッジ不足 (11%)

現在の検索 KW に cloud_infra 系が少ない可能性がある。

## 注目著者 → List 自動追加済み

- @xxx: 直近5件 avg=4.6、List に追加しました

## ⚠️ 削除候補(要確認)

以下のアカウントは archive 状態が30日経過しました。
削除する場合は `removeListsMemberByUserId` を実行してください。

- @yyy: archive 開始 2026-03-18、直近10件 avg=1.2
- @zzz: archive 開始 2026-03-15、直近10件 avg=1.5

## List 状態サマリ

| status  | 件数 |
| ------- | ---- |
| active  | 42   |
| watch   | 3    |
| archive | 2    |
```

---

## ディレクトリ構成

```
x-curator/
├── pyproject.toml
├── .env
├── config/
│   └── sources.json          # ソース設定(編集することで動作が変わる)
├── src/x_curator/
│   ├── __init__.py
│   ├── cli.py                # python -m x_curator run --hours 24
│   ├── fetch.py              # xmcp 経由で List/検索/トレンド取得
│   ├── preprocess.py         # 正規化・重複排除
│   ├── evaluate.py           # 個別評価 Stage1(Haiku) + Stage2(Sonnet)
│   ├── aggregate.py          # 全体集計・分布算出
│   ├── meta_evaluate.py      # 全体評価エージェント(Sonnet)
│   ├── report.py             # Markdown + JSON 出力
│   ├── list_manager.py       # List 追加(自動) / 削除候補提示
│   ├── list_lifecycle.py     # アカウント状態遷移(active/watch/archive)
│   ├── models.py             # Pydantic モデル
│   └── config_loader.py      # sources.json の読み書き
├── cache/
│   └── fetched_ids.json      # 取得済み tweet_id キャッシュ
└── out/
```

### 依存関係

```toml
[project]
dependencies = [
  "httpx",
  "anthropic",
  "pydantic",
  "typer",
  "python-dotenv",
]
```

xmcp はローカルで別プロセスとして起動。`fetch.py` は MCP クライアントとして `http://127.0.0.1:8000/mcp` を叩く。

---

## X API 認証

xmcp の OAuth1 フローを使う。起動時にブラウザ認証が走りトークンはメモリに保持。長時間作業・headless 環境では `.env` に OAuth2 アクセストークンを直書きする方が安定する。

```
X_OAUTH_CONSUMER_KEY=...
X_OAUTH_CONSUMER_SECRET=...
X_BEARER_TOKEN=...
X_OAUTH_ACCESS_TOKEN=...
X_OAUTH_ACCESS_TOKEN_SECRET=...
ANTHROPIC_API_KEY=...
```

X API 料金: Basic プラン ($100/月) が目安。実装前に [developer.x.com](https://developer.x.com) で最新料金・レート制限を確認すること。

---

## コスト概算(1日1回・24h分)

| レイヤ                                       | 月額目安      |
| -------------------------------------------- | ------------- |
| X API Basic                                  | $100          |
| Claude Haiku (Stage1, ~1200件/日)            | $2〜3         |
| Claude Sonnet (Stage2 + 全体評価, ~700件/日) | $20〜30       |
| **合計**                                     | **〜$125/月** |

Anthropic Batch API 使用で Claude 部分は半額化可能(24h 以内納期で問題なければ有効)。

---

## 発展パス

本スキルは MVP(手動 CLI 実行)のスコープ。以下は将来の拡張として別途設計する。

1. **MVP** ← 本スキルのスコープ
2. Cloud Run Job 化(cron 実行・結果永続化)
3. 配信レイヤ追加(Slack DM / メール)
4. フィードバック記録 → 長期的なアカウント信頼性スコアの蓄積
5. ベクトル検索層(過去いいね投稿との類似度スコア追加)

---

## 制約・注意

### API でできないこと

- For You ランキングへの直接介入(公開 API なし)
- 「Not interested」相当のネガティブフィードバック送信(公開 API なし)
- 非公式 GraphQL 直叩きは X 規約違反

### 既知の制限(検証済み)

| 制限                               | 内容                                                                                    | 対応策                                                                     |
| ---------------------------------- | --------------------------------------------------------------------------------------- | -------------------------------------------------------------------------- |
| トレンド API 使用不可              | `getTrendsByWoeid` は OAuth 2.0 App-Only 認証が必要。xmcp は現時点で未対応(PR 対応待ち) | `trend.enabled: false` のまま維持。xmcp が OAuth2.0 対応次第 `true` に変更 |
| 著名人層の空振り                   | `searchPostsRecent(from:xxx)` は24hウィンドウで投稿ゼロになりうる                       | `getUsersPosts` で直接取得に変更済み                                       |
| List汚染によるエンゲージメント歪み | List に大規模フォロワーアカウントが混入するとスコア分布が崩壊する                       | エンゲージメント計算をソース別に分離。`per_source_top20_pct: true`         |

### List 管理のルール

List の追加・新陳代謝はエージェントが自動で行う。**物理削除の直前のみ人間確認**を安全弁として残す。

#### アカウントの状態遷移

```
[active] ──低スコア継続──▶ [watch] ──低スコア継続──▶ [archive] ──30日経過──▶ 削除候補(人間確認)
            ◀──自動復帰──            ◀──自動復帰──
```

各状態の意味:

| status    | 取得対象           | 説明                             |
| --------- | ------------------ | -------------------------------- |
| `active`  | ✅ 通常取得        | 信頼アカウントとして稼働中       |
| `watch`   | ✅ 通常取得        | スコア低下を観察中               |
| `archive` | ⬜ 週1チェックのみ | 取得対象から外れているが復帰可能 |

#### 遷移ルール

**active → watch(自動)**

- 直近20件の平均 trust_score < 2.0
- かつ noise_flag 率 > 40%

**watch → archive(自動)**

- watch 状態が14日以上継続
- かつ直近5件の全てが trust_score <= 2

**archive → active(自動復帰)**

- archive 中も `getUsersPosts` で週1チェック
- 直近3件の平均 trust_score >= 3.5 なら active に戻す

**archive → 削除候補(人間確認)**

- archive 状態が30日継続した場合にレポートに列挙
- 人間が確認後 `removeListsMemberByUserId` を実行

#### 追加ルール(自動)

全体評価の `notable_authors` に挙がったアカウントのうち:

- List 未登録
- 直近N件の平均 trust_score >= 4.0
- 該当件数 >= 3

を満たす場合、`addListsMember` で自動追加し `sources.json` の `members` に記録する。

### データの取り扱い

- 取得データの保存・用途は個人利用の範囲に留める
- 第三者への再配布・サービス化には X Developer Agreement の確認が必要
