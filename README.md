# x-timeline-curator

**X(旧Twitter)の情報収集を、滞留せずに行うための Claude Code スキル**

---

## これは何か

Xは最新の技術情報や議論を得る場として優れていますが、アプリの設計がユーザーを滞留させることを目的としています。このスキルは「Xの情報価値だけを取り出し、消費は自分のペースで行う」ためのパイプラインを Claude Code 上で構築します。

実行すると、設定したソース(List・キーワード検索・著名人の投稿)から投稿を収集し、AIが自動で以下を行います:

- **ノイズ除去**: エンゲージメント釣り・炎上・宣伝を除外
- **トピック分類**: AI/LLM・クラウド・日本のテックビジネスなど
- **信頼性スコアリング**: 一次情報か・発信者の専門性はあるかを1〜5で評価
- **全体診断**: 今日の収集は偏りがないか・何が足りないかをレポート
- **List の自動管理**: 良質な著者を自動追加、低品質アカウントを段階的に降格・削除候補化

最終的に Markdown レポートとして出力されます。Xアプリを開かなくても必要な情報が手に入る状態を作ることがゴールです。

---

## 前提環境

| 必要なもの | 備考 |
|---|---|
| Claude Code | このスキルの実行環境 |
| [xmcp](https://github.com/xdevplatform/xmcp) | X API を MCP ツールとして公開するローカルサーバー。別プロセスで起動しておく |
| X Developer アカウント。[developer.x.com](https://developer.x.com) で申請 |
| Anthropic API キー | Claude Haiku(ノイズフィルタ) + Sonnet(評価・診断)を使用 |

---

## セットアップ

### 1. xmcp を起動する

```bash
git clone https://github.com/xdevplatform/xmcp
cd xmcp
cp env.example .env
# .env に X_OAUTH_CONSUMER_KEY / X_OAUTH_CONSUMER_SECRET / X_BEARER_TOKEN を記入
python server.py
# ブラウザで OAuth 認証 → http://127.0.0.1:8000/mcp で待機状態になる
```

### 2. このスキルをプロジェクトに配置する

```
your-project/
├── .claude/
│   └── settings.json   # MCP ツールの自動承認設定
├── CLAUDE.md
├── SKILL.md
└── config/
    └── sources.json    # ソース設定(あなたの関心に合わせて編集)
```

### 3. `.claude/settings.json` を作成する

MCP ツールの読み取り系を自動承認する設定です。

```json
{
  "mcpServers": {
    "xmcp": {
      "autoApprove": [
        "searchPostsRecent",
        "getUsersPosts",
        "getUsersByIds",
        "getUsersByUsernames",
        "getUsersMe",
        "getListsPosts",
        "getUsersOwnedLists",
        "getListsMembers",
        "getPostsByIds",
        "getPostsQuotedPosts",
        "getPostsRepostedBy",
        "getPostsLikingUsers"
      ]
    }
  }
}
```

### 4. `config/sources.json` を自分の関心に合わせて編集する

最低限 `search_keywords` と `author_seeds` を自分用に書き換えてください。

```json
{
  "lists": [],
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
    "noise_rate_warn": 0.20,
    "primary_source_rate_target": 0.20,
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

### 5. Claude Code から実行する

```bash
claude  # プロジェクトディレクトリで起動
```

```
直近24時間のタイムラインを収集・評価してレポートを out/ に出力してください
```

---

## 出力されるもの

| ファイル | 内容 |
|---|---|
| `out/{date}.md` | 個別投稿レポート。トピック別・信頼性別に整理され、各投稿に元ポストへのリンクつき |
| `out/{date}.json` | 同上のJSON形式(プログラマブルな活用向け) |
| `out/{date}_meta.md` | 全体評価レポート。分布・品質の診断、注目著者、削除候補アカウントの一覧 |
| `out/{date}_meta.json` | 同上のJSON形式 |

---

## List の自動管理について

このスキルは X の List をアカウントの信頼性台帳として育てていきます。

- **自動追加**: キーワード検索や著名人起点で発見した著者のうち、trust_score が継続的に高いアカウントを自動で List に追加します
- **段階的降格**: 低品質な投稿が続くアカウントは `active → watch → archive` と段階的に降格し、取得対象から外します
- **削除候補の提示**: archive 状態が30日続いたアカウントはレポートに列挙します。物理削除だけは人間が確認して実行する設計です

---

## カスタマイズ

`SKILL.md` に設計の全詳細が集約されています。以下を変更したい場合は `SKILL.md` の該当セクションを指定して Claude Code に編集を依頼してください。

| 変えたいこと | 参照セクション |
|---|---|
| 追いたいトピック・キーワード | `ソース設定 > sources.json` |
| ノイズ判定の基準 | `個別評価: Stage 1` |
| トピック分類のカテゴリ | `個別評価: Stage 2` |
| 信頼性スコアの定義 | `個別評価: Stage 2` |
| List の昇格・降格の閾値 | `制約・注意 > List 管理のルール` |
| 全体評価の目標値 | `全体評価エージェント` |

---

## 既知の制限

- **トレンド API**: xmcp が OAuth 2.0 に対応次第、自動で有効化できます(`trend.enabled: true` にするだけ)。現時点では `sources.json` に手動でキーワードを追加することで代替してください
- **API コスト**: X API は Pay-Per-Use($0.005/件)で月$75程度、Claude 部分は月$20〜30程度。合計 **〜$95〜105/月** が目安。Anthropic Batch API を使うと Claude 部分が半額になります。(毎日利用した場合の概算)
- **For You の完全再現は不可能**: X の全投稿プールへのアクセスは Enterprise 契約($42,000/月〜)が必要です。このスキルは「多層クエリ設計によって擬似的に多様性を担保する」アプローチを取っています