---
name: whatsnew
description: "Fetch Claude Code release notes and generate a personalized feature report (MD + HTML) based on your usage patterns. Use when asked about new features, latest updates, release notes, or feature recommendations. Invokable via /whatsnew."
allowed-tools: Bash, Read, Write, Skill, AskUserQuestion
---

# What's New — Claude Code パーソナライズドレポート

Claude Code の最新リリースノートを取得し、あなたの使用パターンを分析して「あなたに特に刺さる機能」を優先的に紹介する。MD + HTML レポートとして生成する。

## 出力ファイル

- `cc-whatsnew-<YYYY-MM-DD>.md` — Markdown レポート（カレントディレクトリ）
- `cc-whatsnew-<YYYY-MM-DD>.html` — スタンドアローン HTML レポート（カレントディレクトリ）

---

## 禁止事項

- **WebFetch・Fetch ツールを使用してはならない** — CHANGELOG の取得は Bash curl のみ使用する
- **Glob・Search・Grep ツールを使用してはならない** — ファイル確認は Bash `test -f`、内容確認は Read のみ使用する
- **ls・既存ファイル確認を自己判断で行ってはならない** — 上書き確認は AskUserQuestion で行う
- **HTML ファイルを Write ツールで直接書き出してはならない** — HTML は Step 4 の Bash コマンドで JSON + テンプレートから自動生成する

## 確認が必要なケース

- **report.html が存在する場合**: 既存を使うか `/insights` を実行するかをユーザーに確認する
- **report.html が存在しない場合**: `/insights` を実行するか `settings.json で代替` するかをユーザーに確認する（自動実行禁止）
- **出力ファイルが既に存在する場合**: 上書きするかキャンセルするかをユーザーに確認する（MD・HTML それぞれ）

## 処理方針

- 各 Step はサブステップ単位で依存関係に従い、可能な限り並列実行する
- Bash コマンドの出力を信頼し、追加確認のための Search・Grep・ls は行わない
- Read ツールで読み込んだ内容から必要な情報を直接把握し、再度検索しない

---

## 処理手順

以下の手順を依存関係に従って実行してください。

### Step 1: 初期データ収集

以下のサブステップを並列で実行する。

#### 1a: 出力言語を決定

`Read` ツールで `~/.claude/settings.json` を読み込み、`language` フィールドの値から出力言語を決定する。以降の全ステップ（レポート本文・HTMLラベル・フッターテキスト）をその言語で統一する。

| `language` の値 | 出力言語 | `lang` コード |
|----------------|---------|--------------|
| `"Japanese"` | 日本語 | `ja` |
| `"English"` または未設定 | 英語 | `en` |
| その他 | 対応する言語 | BCP47 コード |

#### 1b: ユーザープロファイルを収集（Insight）

`Bash` で `test -f ~/.claude/usage-data/report.html && echo "EXISTS" || echo "NOT_FOUND"` を実行して存在確認する。

**レポートが存在する場合**: `Read` ツールでファイルの先頭50行を読み込み、`<p class="subtitle">` 内の日付範囲を確認する。その後 **必ず** `AskUserQuestion` で確認する（自動選択禁止・省略禁止）:

> 「〇〇〜〇〇のInsightレポートが `~/.claude/usage-data/report.html` に存在します。どのように進めますか？」
> 選択肢: `["既存レポートを使う（推奨）", "/insights を新たに実行する"]`

- 「既存レポートを使う」を選んだ場合: そのまま次の Read へ進む
- 「/insights を新たに実行する」を選んだ場合: `Skill` ツールで `insights` スキルを呼び出す

**レポートが存在しない場合**: **必ず** `AskUserQuestion` ツールで確認する（自動選択禁止・省略禁止）:

> 「`report.html` が存在しないため、`/insights` の実行が必要です。どのように進めますか？」
> 選択肢: `["/insights を実行する（推奨）", "settings.json で代替（精度が下がります）"]`

- `/insights を実行する` を選んだ場合: `Skill` ツールで `insights` スキルを呼び出す
- `settings.json で代替` を選んだ場合: `~/.claude/settings.json`・スキル一覧・フック設定からプロファイルを推定する

**いずれの場合も**: `Read` ツールで `~/.claude/usage-data/report.html` を一度だけ読み込み、以下の情報を把握する:

| 把握する情報 | HTML 内の手がかり |
|---|---|
| セッション数・メッセージ数・稼働日数 | `stat-value` クラスの数値 |
| 主なゴール・タスクタイプ | `What You Wanted` セクション |
| よく使うツール | `Top Tools Used` セクション |
| 主な開発言語 | `Languages` セクション |
| 作業プロジェクト | `project-area` セクション |
| 摩擦ポイント | `friction-category` セクション |
| 未活用機能の提案 | `feature-card` セクション |
| 成功パターン | `big-win` セクション |
| 総合サマリー | `at-a-glance` セクション |

収集した情報からプロファイルを箇条書きで内部的にメモしておく。例:

- バックエンド系の作業が多い
- セキュリティ意識が高い（複数のセキュリティフック）
- 自動化・ワークフロー開発をしている
- カスタムスキルを積極的に作成している
- 摩擦ポイント: ファイル編集後のテスト実行を忘れがち

#### 1c: リリースノートを取得

> ⚠️ **WebFetch・Fetch ツール使用禁止。必ず `Bash` + `curl` を使うこと。**

<!-- NOTE: /release-notes は Claude Code の組み込みコマンドだが、SKILL.md ベースではない別実装のため
     Skill ツールから呼び出すと "not a prompt-based skill" エラーになり使用不可。
     WebFetch では 129KB 全体がコンテキストに入り遅くなるため、Bash curl + awk で先頭3バージョンのみ抽出する。 -->

`Bash` で以下のコマンドを実行し、CHANGELOG の先頭3バージョン分のみ取得する:

```bash
# NOTE: -k フラグ（TLS検証無効化）は絶対に使用しないこと
curl -s https://raw.githubusercontent.com/anthropics/claude-code/refs/heads/main/CHANGELOG.md \
  | awk '/^## /{count++} count>3{exit} {print}'
```

出力から以下の情報を把握する:

- 各バージョン/日付のリリース内容
- 機能カテゴリ（新機能 / 改善 / バグ修正 / 実験的機能）
- 機能の対象ユーザー（開発者向け / CI/CD向け / エンタープライズ向け など）

#### 1d: テンプレートの読み込みと出力ファイルの確認

> 1a（言語決定）の完了後に実行する。1b・1c とは並列実行可。

**テンプレート読み込み:**

1a で決定した言語コードに応じてテンプレートを選択する:

| 言語 | MD テンプレート | HTML テンプレート |
|---|---|---|
| `ja` | `${CLAUDE_SKILL_DIR}/assets/template.ja.md` | `${CLAUDE_SKILL_DIR}/assets/template.ja.html` |
| `en` | `${CLAUDE_SKILL_DIR}/assets/template.en.md` | `${CLAUDE_SKILL_DIR}/assets/template.en.html` |
| その他 | `${CLAUDE_SKILL_DIR}/assets/template.en.md`（フォールバック） | `${CLAUDE_SKILL_DIR}/assets/template.en.html`（フォールバック） |

ユーザーカスタム HTML テンプレートが `~/.claude/whatsnew-template.html` に存在する場合はそちらを優先する（`Bash` で `test -f` 確認）。

選択したテンプレートを `Read` で並列読み込みする。

**出力ファイル確認:**

- `Bash` で `test -f cc-whatsnew-<DATE>.md && echo "EXISTS" || echo "NOT_FOUND"` を実行
- `Bash` で `test -f cc-whatsnew-<DATE>.html && echo "EXISTS" || echo "NOT_FOUND"` を実行

どちらか一方でも既存ファイルが存在する場合は `AskUserQuestion` で確認する:

> 「以下のファイルが既に存在します: `cc-whatsnew-<DATE>.md` / `cc-whatsnew-<DATE>.html`。上書きしますか？」
> 選択肢: `["上書きする", "キャンセル"]`

キャンセルを選んだ場合はスキルを終了する。

上書きを選んだ場合は、既存ファイルを `Bash` で削除しておく（Step 4 の Write を新規作成にして高速化するため）:

```bash
rm -f cc-whatsnew-<DATE>.md cc-whatsnew-<DATE>.html
```

---

### Step 2: コンテンツ分析

Step 1 完了後、以下のサブステップを並列で実行する。

#### 2a: パーソナライズドレコメンデーションを生成

1b のプロファイルと 1c の機能リストを照合して、「あなたへのおすすめ」を決定する。

**マッチングロジック（優先度高い順）:**

1. **直接マッチ**: ユーザーが使っているスキル・ツールに直接関係する機能
   - 例: 外部サービス連携が多い → MCP/ツール統合の新機能を優先
   - 例: commit/securityスキルが多い → Git連携・セキュリティ強化を優先

2. **ワークフロー補完**: 現在のワークフローを強化できる機能
   - 例: 多くのカスタムスキルがある → スキル作成・管理の新機能を優先
   - 例: フックを多用 → フック・自動化の新機能を優先

3. **未活用機能**: ユーザーがまだ使っていなさそうだが有用な機能
   - プロファイルに対応するツールが見当たらない場合に提案

各おすすめ機能には以下を添える:

- なぜこの機能があなたに刺さるか（1〜2文で具体的に）
- 使い始めるための最初のステップ（コマンド例や設定例）

#### 2b: 最新バージョン詳細解説を生成

1c で取得したリリースノートのうち、**最新バージョン（一番上）の新機能（Added）を1件ずつ詳しく解説する**。

各機能について以下をまとめる:

- **概要**（1文）: 何ができるようになったか
- **メリット**（2〜3文）: なぜ使うべきか、具体的にどう役立つか
- **使い方のヒント**（あれば）: コマンド例・設定例

解説の粒度の目安:

- 単純なUI改善（スピナー表示など）は1〜2文で簡潔に
- フック・スキル・設定など開発ワークフローに影響する機能は3〜4文でしっかり説明
- セキュリティ修正は「何が塞がれたか」「ユーザーへの影響」を明示

---

### Step 3: 出力ファイルを生成

Step 2 完了後、以下の 3a と 3b を **同一ターンで並列に実行する**。

#### 3a: MD ファイルを Write

1d で読み込んだ MD テンプレートのプレースホルダーを置換して `Write` する:

| プレースホルダー | 内容 | 書式 |
|---|---|---|
| `__DATE__` | 今日の日付 | YYYY-MM-DD |
| `__LATEST_VERSION__` | 最新バージョン番号 | そのまま |
| `__SECTION_PICKS_TITLE__` | `TOP N`（N はおすすめ件数） | そのまま |
| `__TOP_PICKS_MD__` | おすすめ機能（2a の結果） | `### 1. タイトル` + 理由 + 始め方 |
| `__LATEST_DETAIL_MD__` | 最新バージョン詳細（2b の結果） | 各機能を見出し + 本文 |
| `__ALL_FEATURES_MD__` | 全リリースノート（1c のデータ） | バージョンごとに `### vX.Y.Z` |
| `__PROFILE_MD__` | プロファイル要約 + タグ | 箇条書き |

#### 3b: JSON データファイルを Write

Step 1〜2 で収集したデータを以下の JSON スキーマに従って `/tmp/whatsnew-data.json` に `Write` する。HTML テンプレート内の JavaScript がこの JSON を読み込んでページを描画する。

```json
{
  "date": "YYYY-MM-DD",
  "latest_version": "vX.Y.Z",
  "picks_title": "あなたへのおすすめ TOP N",
  "profile_summary": "プロファイル要約（1〜2文）",
  "profile_tags": ["タグ1", "タグ2"],
  "picks": [
    {"rank": 1, "title": "機能名", "reason": "理由", "howto": "コマンド例"}
  ],
  "details": [
    {"title": "機能名", "overview": "概要", "merit": "メリット", "tip": "ヒント（省略可）"}
  ],
  "releases": [
    {
      "version": "vX.Y.Z",
      "date": "YYYY-MM-DD",
      "items": [{"badge": "feat", "text": "説明"}]
    }
  ]
}
```

**badge の値**: `feat`（新機能）、`fix`（修正）、`improve`（改善）、`exp`（実験的）

**テキスト内の強調**: `**テキスト**` で太字、`` `コード` `` でインラインコードを指定可能（HTML レンダラーが変換する）

---

### Step 4: HTML を生成

Step 3b の JSON ファイルを、1d で読み込んだ HTML テンプレートに注入して HTML ファイルを生成する。`Bash` で以下のコマンドを実行する:

```bash
awk '/__JSON_DATA__/{system("cat /tmp/whatsnew-data.json");next}1' \
  "${CLAUDE_SKILL_DIR}/assets/template.${LANG}.html" > "cc-whatsnew-${DATE}.html"
```

> HTML テンプレート内の JavaScript が JSON データからページコンテンツを描画する。LLM が HTML タグを直接生成する必要はない。

---

### Step 5: 完了報告

一時ファイルを削除する:

```bash
rm -f /tmp/whatsnew-data.json
```

```
✓ パーソナライズドレポートを生成しました

  あなたのプロファイル: <主な特徴2〜3個>
  おすすめ機能数: <N>件
  リリースノート件数: <N>件

  MD : cc-whatsnew-<DATE>.md
  HTML: cc-whatsnew-<DATE>.html
       open cc-whatsnew-<DATE>.html
```

---

## ソース帰属（著作権コンプライアンス）

このスキルが生成するすべての出力（MD および HTML）に、必ずソース帰属を含めること。省略は不可。

### Markdown レポートのフッター（必須）

```markdown
---
*Sources: [GitHub CHANGELOG](https://github.com/anthropics/claude-code/blob/main/CHANGELOG.md) · [Release Notes](https://docs.anthropic.com/en/docs/claude-code/release-notes)*
*Generated by /whatsnew (https://github.com/ni4ta9/cc-whatsnew) — Unofficial community plugin, not affiliated with Anthropic.*
```

### HTML レポートのフッター

フッター（公式リリースノートリンク・クレジット・出典表記）は HTML テンプレート内にハードコードされており、JS が自動描画する。LLM が JSON データにフッター関連フィールドを含める必要はない。

---

## 引数

`$ARGUMENTS`（オプション）— 指定なしで最新のリリースノートを取得。バージョンを指定した場合（例: `v0.2.0`）はそのバージョン周辺の情報にフォーカスする。
