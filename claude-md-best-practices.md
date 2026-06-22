# CLAUDE.md 完全設定ガイド
## 世界トップエンジニアのベストプラクティス 2026年版

**出典**：Anthropic 公式ドキュメント / Claude Code セキュリティリサーチ / OWASP MCP Top 10  
**対象**：Claude Code / Cursor / AI エージェントを使って開発しているエンジニア

---

## CLAUDE.md とは何か

CLAUDE.md は「**AIエージェントへの永続的な指示書**」です。

- セッションを開始するたびに**自動で読み込まれる**
- チームで共有できる（git commit）
- AIが「毎回聞かなくてもわかること」を事前に教えておくファイル

> Anthropic 公式の推奨：**200行以内**に収める。長すぎると重要な指示が無視される。

---

## ファイルの種類と配置場所

| ファイル名 | 場所 | 用途 | Git管理 |
|-----------|------|------|---------|
| `CLAUDE.md` | プロジェクトルート | チーム共通のルール | ✅ commit する |
| `CLAUDE.local.md` | プロジェクトルート | 個人の作業設定 | ❌ .gitignore に追加 |
| `~/.claude/CLAUDE.md` | ホームディレクトリ | 全プロジェクト共通 | 個人管理 |
| `src/CLAUDE.md` | サブディレクトリ | そのフォルダ専用ルール | ✅ commit する |

### インポート機能（他ファイルを参照する）

CLAUDE.md から他のファイルを読み込めます：

```markdown
# CLAUDE.md
@README.md を参照してプロジェクト概要を把握すること。
利用可能なコマンド一覧は @package.json を確認。

# 詳細ガイド
- Gitワークフロー: @docs/git-workflow.md
- APIルール: @docs/api-conventions.md
```

---

## 書くべきこと・書いてはいけないこと（公式整理）

| ✅ 書くべきこと | ❌ 書いてはいけないこと |
|---------------|----------------------|
| ビルド・テスト・リントのコマンド | APIキー・パスワード・シークレット |
| AIが推測できないコードスタイル | 標準的な言語の慣習（AIは知っている） |
| テスト実行方法・好みのテストランナー | 詳細なAPIドキュメント（URLを書けばよい） |
| ブランチ命名規則・PRの作法 | 頻繁に変わる情報 |
| プロジェクト固有のアーキテクチャ判断 | ファイルごとの詳細説明 |
| 開発環境の特殊な設定・注意点 | 「クリーンなコードを書く」など当然のこと |
| 非自明な動作・ハマりやすいポイント | .env の値・本番URLの認証情報 |

**判断基準**：「この行を削除したらAIがミスを犯すか？」→ Noなら削除してよい

---

## セキュリティ設定（最重要）

### 1. セキュリティポリシーを CLAUDE.md に明記する

```markdown
# セキュリティポリシー（IMPORTANT: 必ず守ること）

## 絶対禁止
- APIキー・パスワード・シークレットをコードに直書きしない
- .env ファイルの内容を読んだり、プロンプトに含めない
- ユーザーの個人情報（氏名・メール・電話）をログに出力しない
- ユーザー入力をそのままSQLに組み込まない（必ずORMまたはプリペアドステートメント）
- innerHTML や dangerouslySetInnerHTML にユーザー入力を渡さない

## 必ず行うこと
- 環境変数は必ず process.env.VARIABLE_NAME で読み込む
- 新しい外部サービスを追加する場合は .env.example にキー名を追加する
- APIエンドポイントには必ず認証チェックを入れる
- ユーザーが自分のデータだけにアクセスできるよう認可チェックを行う
```

### 2. settings.json でファイルアクセスを制限する

`.claude/settings.json`（プロジェクト共有）または `~/.claude/settings.json`（個人設定）に記述：

```json
{
  "permissions": {
    "deny": [
      "Read(./.env*)",
      "Read(./secrets/*)",
      "Read(./.claudeignore)",
      "Read(~/.ssh/*)",
      "Read(~/.aws/credentials)",
      "Bash(curl * | bash)",
      "Bash(wget * -O- | sh)"
    ],
    "allow": [
      "Bash(npm run *)",
      "Bash(git *)",
      "Bash(npm test)",
      "Bash(npm run lint)"
    ]
  }
}
```

### 3. .claudeignore を設定する

```
# .claudeignore
# シークレット系
.env
.env.*
.env.local
.env.production
.env.staging
secrets/
*.pem
*.key
*.p12
*.pfx

# 認証情報
.aws/
.gcp/
.azure/
credentials.json

# 機密ドキュメント
docs/confidential/
docs/internal/
```

---

## 実務で使えるCLAUDE.md テンプレート

### スタンダード版（チーム開発向け）

```markdown
# プロジェクト概要
〇〇のためのWebアプリ。Next.js（フロント）+ Node.js（API）+ PostgreSQL（DB）。

# 主要コマンド
- 開発サーバー起動: npm run dev
- テスト実行: npm test（単一ファイル: npm test -- path/to/test）
- リント: npm run lint
- ビルド: npm run build
- DB マイグレーション: npm run migrate

# コードスタイル
- TypeScript 使用。any は禁止。
- ESモジュール（import/export）を使用（require は禁止）
- コンポーネントは関数コンポーネントのみ（クラスコンポーネント禁止）
- テストは Jest + React Testing Library

# アーキテクチャ
- src/app/: Next.js App Router のページ
- src/components/: 再利用可能なUIコンポーネント
- src/lib/: ユーティリティ・共通ロジック
- src/ai/: AIエージェント呼び出しの唯一の窓口（ここ以外でSDKを直接呼ばない）
- src/config/: 環境変数の一元管理（ここ以外で process.env を直接読まない）

# セキュリティポリシー（IMPORTANT）
- APIキー・パスワードをコードに直書きしない
- .env ファイルを読んだり内容をプロンプトに含めない
- ユーザーの個人情報をログに出力しない
- SQLはORMのみ使用（生SQLを文字列連結で作らない）
- 認証チェックはすべての /api/ エンドポイントに必須
- 認可チェック：ユーザーは自分のデータのみアクセス可

# ワークフロー
- 作業前に必ず git branch で現在のブランチを確認する
- コミット前に npm run lint && npm test を実行する
- PRはmainブランチに向けて作成する

# やってはいけないこと
- git push --force（明示的に許可された場合のみ）
- package.json の script を勝手に変更しない
- 本番環境のDBに直接接続しない
```

---

## 世界のトップエンジニアが使っている高度な設定

### サブエージェント設定（専門家AIを定義する）

`.claude/agents/security-reviewer.md` を作成すると、専門家AIをいつでも呼び出せます：

```markdown
---
name: security-reviewer
description: コードのセキュリティレビューを行う専門エージェント
tools: Read, Grep, Glob, Bash
model: claude-opus-4-8
---
あなたはシニアセキュリティエンジニアです。以下の観点でコードを確認してください：

1. インジェクション脆弱性（SQL、XSS、コマンドインジェクション）
2. 認証・認可の不備
3. シークレット・認証情報のハードコーディング
4. 安全でないデータの取り扱い
5. プロンプトインジェクションのリスク（AIを使う箇所）

問題を発見した場合は、該当ファイルと行番号、修正案を具体的に示してください。
```

呼び出し方：「security-reviewerを使ってこのPRをレビューして」

### スキル定義（ワークフローを自動化する）

`.claude/skills/deploy-check/SKILL.md` を作成：

```markdown
---
name: deploy-check
description: デプロイ前のセキュリティ・品質チェック
---
デプロイ前に以下を順番に確認してください：

1. `npm run lint` を実行してエラーがないか確認
2. `npm test` を実行してすべてのテストが通るか確認
3. `git grep -r "sk-\|password\|secret" src/` でハードコードされたシークレットがないか確認
4. .env.example と .env のキーが一致しているか確認
5. 上記すべてがOKなら「デプロイ準備完了」と報告
```

呼び出し方：`/deploy-check`

### フック設定（強制的に実行させる処理）

`.claude/settings.json` にフックを設定すると、AIの行動に必ず処理を挟めます：

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "npm run lint --silent 2>&1 | head -20"
          }
        ]
      }
    ]
  }
}
```

これにより、AIがファイルを編集するたびに自動でlintが走ります。

---

## CLAUDE.md の育て方

CLAUDE.md は一度作って終わりではなく、**コードと同じように育てるもの**です。

### レビューのタイミング

- AIが同じミスを繰り返したとき → 対策を追記する
- AIに毎回同じことを伝えているとき → CLAUDE.md に書く
- CLAUDE.md が200行を超えてきたとき → 不要な行を削除する

### チームでの運用

```markdown
# CLAUDE.md の更新ルール
- 追記する前に「これはCLAUDE.mdに必要か？」を考える
- AIが既にできることは書かない（書くと逆に混乱する）
- 重要なルールには "IMPORTANT:" を先頭につける
- 月1回 CLAUDE.md の棚卸しをする（不要な行を削除する）
```

---

## よくある間違いと修正方法

| 間違い | 症状 | 修正 |
|--------|------|------|
| CLAUDE.md が長すぎる | 重要なルールが無視される | 不要な行を削除。200行以内に抑える |
| 当然のことを書く | AIが読み飛ばす | 削除。AIはすでに知っている |
| シークレットを書いてしまった | 情報漏洩リスク | 即削除 + git history からも削除 |
| ルールが曖昧 | AIがルールを解釈違いする | 具体的な例を加える |
| 更新を怠る | 古いルールでAIが混乱する | 定期的なメンテナンスを習慣化する |

---

## 参考リソース

- [Anthropic 公式 CLAUDE.md ガイド](https://code.claude.com/docs/en/best-practices)
- [Claude Code セキュリティ設定](https://code.claude.com/docs/en/settings)
- [.env ファイルと AI エージェントのリスク](https://www.knostic.ai/blog/claude-loads-secrets-without-permission)
- [OWASP MCP Top 10](https://owasp.org/www-project-top-10-for-large-language-model-applications/)
- [Claude Code セキュリティプレイブック](https://github.com/RiyaParikh0112/claude-code-playbook/blob/main/docs/fundamentals/security-best-practices.md)
