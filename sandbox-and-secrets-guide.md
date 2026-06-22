# AIエージェント セキュリティ 中級編
## 「誰が何を見られるか」完全ガイド ＋ 実践的な秘密情報管理

**対象**：AIエージェントを使って開発・業務自動化を行うエンジニア・経営者  
**レベル**：中級（基本的なClaude Code / Cursor の操作に慣れた方）

---

## この資料で学ぶこと

| 学習項目 | 重要度 | 概要 |
|---------|-------|------|
| 「誰が何を見られるか」の主語を理解する | ★★★ 最重要 | 全ての概念の土台。ここがズレると全部矛盾する |
| Credential Proxy の仕組み | ★★★ 高 | APIキー安全管理の核心 |
| マスキングの実装 | ★★★ 高 | 個人情報を扱う開発で必須 |
| Docker コンテナ vs. サンドボックス | ★★ 中 | 言葉を正しく使うために |
| AWS Secrets Manager の使い方 | ★★ 中 | 本番環境での秘密情報管理 |
| 1Password CLI の使い方 | ★ 参考 | チーム開発での秘密情報共有 |

---

## CHAPTER 1：最重要 —「誰が何を見られるか」

### 3人の登場人物

AIエージェントを使う時、実は**3つの異なる存在**が動いています。  
この3つを混同すると、セキュリティの説明が全て矛盾します。

| 名前 | 別名 | 役割 | 何を見られるか |
|------|------|------|--------------|
| **頭脳** | Claude Code 本体 | 指示を理解・ファイルを読む・コードを書く | PCの全ファイル（設定で制限可能） |
| **作業部屋** | サンドボックス | コードが実際に動く隔離された場所 | 作業フォルダ内の操作のみ |
| **受付係** | Credential Proxy | 外部API（OpenAI等）との橋渡し | 事前に預かったAPIキー |

### 視覚図：誰が何を見ているか

```
あなたのPC
┌──────────────────────────────────────────────────────┐
│                                                       │
│  【頭脳】 Claude Code 本体                             │
│  ─────────────────────────────                        │
│  ・ファイルを読む（Readツール）                          │
│  ・コマンドを実行する（Bashツール）                      │
│  ・プロジェクト外のファイルも届く ⚠️                    │
│    例：~/secrets/.env も読める                         │
│    例：~/.ssh/ も読める（設定しないと）                  │
│                          ↓                            │
│    ┌──────────────────────────────┐                  │
│    │ 【作業部屋】サンドボックス       │                  │
│    │ ──────────────────────────   │                  │
│    │ ・.py .js などのコードが動く    │                  │
│    │ ・コードで触れるのは作業フォルダ内│                  │
│    │ ・外部サービスへの直接接続は制限  │                  │
│    └──────────────────────────────┘                  │
│                                                       │
│  【受付係】 Credential Proxy                           │
│  ─────────────────────────────                        │
│  ・起動時にAPIキーを預かっている                         │
│  ・サンドボックスの代わりに外部APIを呼ぶ                  │
│  ・サンドボックスにはAPIキーを渡さない                    │
│                                                       │
└──────────────────────────────────────────────────────┘
                   ↕ APIコール（OpenAI / Anthropic等）
              外部サービス
```

### よくある誤解と正しい理解

| ❌ よくある誤解 | ✅ 正しい理解 |
|--------------|-------------|
| 「サンドボックスを使えば.envも安全」 | .envは**頭脳**（Claude Code本体）が読める。サンドボックスは別の仕組み |
| 「プロジェクト外に置けば安全」 | **頭脳**は全フォルダを読める（settings.jsonで制限が必要） |
| 「受付係がAPIキーを探しに行く」 | **受付係はあらかじめAPIキーを持っている**。探しには行かない |
| 「コンテナ＝サンドボックス」 | 異なる概念（CHAPTER 2で詳説） |
| 「外部APIは受付係しか呼べない」 | 設定次第。受付係を使うのはあくまで**推奨パターン** |

---

## CHAPTER 2：Docker コンテナ vs. サンドボックス

### 2つの「隔離」はどう違うか

```
通常の Docker コンテナ（開発用）
┌─────────────────────────┐
│     あなたのPC（建物）    │
│  ┌──────────────────┐  │
│  │  コンテナ（部屋）   │  │
│  │  ・鍵はかかってる   │  │
│  │  ・でも廊下（共有部） ─┼──→ 他の部屋に影響する可能性あり
│  │    は共有          │  │
│  └──────────────────┘  │
└─────────────────────────┘

Docker Sandbox（AI専用・より強い隔離）
┌─────────────────────────┐    ┌──────────────┐
│     あなたのPC（建物）    │    │ 別の小さな建物 │
│                         │    │ （Micro VM）  │
│                         │───→│  AIがここで   │
│                         │    │  動く         │
│                         │    │  何か壊れても  │
│                         │    │  元の建物に無関係│
└─────────────────────────┘    └──────────────┘
```

### 何を守って、何を守らないか

| 比較項目 | Docker コンテナ | Docker Sandbox（Micro VM） |
|---------|----------------|--------------------------|
| 使い道 | アプリ開発・実行環境の統一 | AIエージェントに作業させる |
| PCへの影響 | 共有部分あり（注意が必要） | Micro VMで完全隔離（影響小） |
| 壊れた時 | コンテナ再起動 | Sandbox削除して作り直すだけ |
| 守るもの | 他のアプリへの干渉 | PC全体への誤操作・影響 |
| 守らないもの ⚠️ | 共有フォルダ内のファイル変更 | 作業フォルダ内のファイル |
| AIの読み取り制限 | コード実行の範囲 | 頭脳は引き続き外部ファイルを読める |

### 重要：サンドボックスでも守れないこと

```
✅ サンドボックスが守ること
  └─ コードが誤って他のプロジェクトのファイルを書き換えること
  └─ AIが暴走してPC全体を壊すこと
  └─ 複数プロジェクト間の干渉

❌ サンドボックスでは守れないこと
  └─ 作業フォルダ内に置いたAPIキー（コードで読める）
  └─ 作業フォルダ内の個人情報（コードで読める）
  └─ Claude Code本体が全フォルダを読むこと（頭脳は別）
  └─ プロンプトに直接貼り付けた個人情報
```

---

## CHAPTER 3：Credential Proxy の仕組み

### 「受付係」の正体

Credential Proxy は「APIキーを事前に預かった受付係」です。

**重要なポイント：受付係はAPIキーを探しに行きません。**  
起動時にすでに手元に持っています。

### ホテルのたとえ

```
【通常のやり方（危険）】
  ゲスト（サンドボックス）
    → 「マスターキーを使ってこの部屋を開けて」
    → マスターキーをゲストに直接渡す  ← ❌ 危険
    → ゲストがマスターキーを知ってしまう

【Credential Proxyを使った安全なやり方】
  ゲスト（サンドボックス）
    → 「この部屋に入りたい」
    → フロント係（Credential Proxy）に伝える
    → フロント係が自分のマスターキーで開ける  ← ✅ 安全
    → ゲストはマスターキーを見ない・知らない
```

### 誰がAPIキーを「持つ」のか

```
起動前：
  管理者 → Credential Proxy に APIキーを設定（一度だけ）

動作中：
  サンドボックス →「OpenAIで画像を生成して」
                ↓
  Credential Proxy（受付係） → 手元のAPIキーでOpenAIに接続
                             ↓
  OpenAI → 画像を生成
                             ↓
  Credential Proxy → サンドボックスに結果を返す

  ※ この全過程でサンドボックスはAPIキーを見ない
```

### Credential Proxy の設定例（Node.js）

```javascript
// credential-proxy/server.js
const express = require('express');
const { OpenAI } = require('openai');

// APIキーはここで一度だけ設定（環境変数から）
const openai = new OpenAI({ apiKey: process.env.OPENAI_API_KEY });

const app = express();
app.use(express.json());

// サンドボックスからのリクエストを受け付ける
app.post('/generate-image', async (req, res) => {
  const { prompt } = req.body;
  
  // APIキーを使うのはプロキシだけ
  const image = await openai.images.generate({ prompt, n: 1, size: '512x512' });
  
  res.json({ url: image.data[0].url });
});

app.listen(3001, () => console.log('Credential Proxy 起動'));
```

```javascript
// sandbox/app.js（サンドボックス内のコード）
// ← APIキーはどこにも書いていない！

async function generateImage(prompt) {
  // プロキシに頼むだけ（APIキーを知る必要がない）
  const response = await fetch('http://localhost:3001/generate-image', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ prompt }),
  });
  return await response.json();
}
```

---

## CHAPTER 4：データの置き場所ガイド

### 動物病院の例で考える

動物病院が扱うデータを「置いていいもの」「置いてはいけないもの」に分けます。

#### ❌ 作業フォルダに絶対置いてはいけないもの

| データ種別 | 具体例 | 置く場所 |
|-----------|--------|---------|
| **カルテ原本** | 診断記録・処方履歴・ペット情報 | 病院の電子カルテシステム（EMR） |
| **医療記録** | 手術記録・検査結果 | EMR / 院内サーバー（外部接続なし） |
| **フル個人情報** | 飼い主の氏名＋住所＋電話のセット | 顧客管理DB（CRM） |
| **パスワード** | DBパスワード・管理画面パスワード | AWS Secrets Manager / 1Password |
| **APIキー** | OpenAI API Key / Stripe API Key | 同上 |
| **保険証・支払い情報** | クレジットカード情報 | 決済事業者（Stripe等）に委託 |

#### ✅ 作業フォルダに置いてよいもの（AIに渡せるデータ）

| データ種別 | 具体例 | 条件 |
|-----------|--------|------|
| **匿名化データ** | ペットID・年齢カテゴリ・症状カテゴリ | 個人を特定できない形に加工済み |
| **統計・集計データ** | 月別受診数・疾患ランキング | 個人情報を含まない |
| **マスキング済みデータ** | 住所の都道府県まで / 電話番号は非表示 | 必要最小限の情報のみ |
| **サンプル・テストデータ** | 架空の情報で作ったテスト用データ | 実際の個人情報を含まない |

### データを渡す前の「マスキングフロー」

```
カルテDB（EMR）
    ↓
  必要な属性だけ取り出す
  （氏名・電話は除外、ペットID・年齢・症状だけ）
    ↓
  マスキング処理（次章で実装）
    ↓
  AIエージェント（作業フォルダ内）
    ↓
  分析・自動化
    ↓
  結果を人間がレビュー
    ↓
  EMRに書き戻す（人間の手で）
```

---

## CHAPTER 5：マスキングの実装

### 基本原則：AIに渡す前に「必要最小限に絞る」

```javascript
// ❌ やってはいけない：個人情報を丸ごとAIに渡す
const patient = await db.query('SELECT * FROM patients WHERE id = ?', [id]);
const prompt = `この患者の対応方法を教えて: ${JSON.stringify(patient)}`;
// ↑ 氏名・住所・電話・生年月日が全部プロンプトに入る

// ✅ 正しい方法：必要な情報だけに絞ってAIに渡す
function maskPatient(patient) {
  return {
    petId: patient.pet_id,           // ペットID（匿名）
    species: patient.species,         // 犬・猫など
    ageGroup: patient.age_group,      // 子犬・成犬・老犬
    symptoms: patient.symptoms,       // 症状カテゴリ
    visitCount: patient.visit_count,  // 来院回数
    // ↓ 以下は渡さない
    // owner_name, phone, address, birthday は除外
  };
}

const safeData = maskPatient(patient);
const prompt = `この症例への対応を提案して: ${JSON.stringify(safeData)}`;
```

### 画面表示のマスキング（フロントエンド）

```javascript
// 管理画面での個人情報マスキング表示

function maskPhone(phone) {
  // 090-1234-5678 → 090-****-5678
  return phone.replace(/(\d{3})-(\d{4})-(\d{4})/, '$1-****-$3');
}

function maskAddress(address) {
  // 東京都新宿区西新宿1-2-3 → 東京都新宿区 ****
  return address.replace(/(.{5,})区(.*)/, '$1区 ****');
}

function maskName(name) {
  // 田中 花子 → 田中 ＊＊
  const parts = name.split(' ');
  return parts[0] + ' ' + '＊'.repeat(parts[1]?.length || 2);
}
```

### ログ出力のマスキング

```javascript
// ❌ 個人情報をログに出力してしまう例
console.log('処理完了:', { name: '田中花子', phone: '090-1234-5678', result: 'OK' });

// ✅ 個人情報をログから除外する
const { name, phone, ...safeLog } = userData;
console.log('処理完了:', { userId: userData.id, result: 'OK' });
// ↑ ログには ID と結果だけ残る
```

### AIへのプロンプトに個人情報を含めない仕組み（集中管理）

```javascript
// src/ai/prompt-builder.js
// ← ここだけでプロンプトを作る（他の場所でAIに渡さない）

function buildPetAnalysisPrompt(rawPatientData) {
  // このファイルでのみマスキング処理を行う
  const safe = {
    petId: rawPatientData.pet_id,
    species: rawPatientData.species,
    symptoms: rawPatientData.symptoms,
    visitCount: rawPatientData.visit_count,
  };
  
  return `
以下のペット情報に基づいて、次回の診察で確認すべき項目を3つ提案してください。

ペットID: ${safe.petId}
種類: ${safe.species}
主な症状: ${safe.symptoms.join(', ')}
来院回数: ${safe.visitCount}回

※ 個人を特定する情報は含まれていません。
  `.trim();
}

module.exports = { buildPetAnalysisPrompt };
```

---

## CHAPTER 6：APIキー管理の進化

### 安全性レベルの比較

```
レベル1（最低限）：.env ファイル
  ├─ ファイルとして存在する → AI に読まれる可能性あり
  ├─ .gitignore 設定で Git には入らない
  ├─ 設定しないと GitHub に漏れる ⚠️
  └─ ローカル開発の出発点として許容

レベル2（推奨）：OS の環境変数
  ├─ ファイルとして存在しない → ディレクトリスキャンに出ない
  ├─ メモリ上のみに存在
  ├─ ただし AI がコードを実行すれば process.env で読める
  └─ .env より攻撃表面が小さい（完全安全ではない）

レベル3（本番向け）：AWS Secrets Manager / HashiCorp Vault
  ├─ コードにキーを書かない
  ├─ IAM ロール（身分証明）で認証 → キーを知らなくても使える
  ├─ アクセスログが残る（誰がいつ取得したか）
  └─ キーのローテーション（定期更新）が容易

レベル4（チーム向け）：1Password CLI
  ├─ チームで秘密情報を安全に共有できる
  ├─ .env ファイルに実際の値を書かない
  └─ `op run` でプロセス起動時に環境変数として注入
```

---

## CHAPTER 7：AWS Secrets Manager 詳解

### 認証の仕組み —「毎回ログインが必要？」

**必要ありません。** AWS は「セッション認証」を使います。

```
開発環境（ローカルPC）:
  $ aws configure  ← 一度だけ実行
  Access Key ID: AKIA...
  Secret Access Key: ****
         ↓
  ~/.aws/credentials に保存される（一時トークンを自動更新）
  以降は自動で認証される

本番環境（EC2 / Lambda / ECS）:
  IAMロールをサービスに付与  ← インフラ設定で一度だけ
         ↓
  コードから認証コード不要で自動認証
  AWS SDK が自動的に IAM ロールを使う
```

### シークレット取得のコード例（キャッシュあり）

```javascript
// src/config/secrets.js
const { SecretsManagerClient, GetSecretValueCommand } = require('@aws-sdk/client-secrets-manager');

const client = new SecretsManagerClient({ region: 'ap-northeast-1' });

// キャッシュ（アプリ起動時に一度だけ取得）
let cachedSecrets = null;

async function getSecrets() {
  if (cachedSecrets) return cachedSecrets;  // キャッシュがあれば再取得しない
  
  const response = await client.send(
    new GetSecretValueCommand({ SecretId: 'myapp/production' })
  );
  
  cachedSecrets = JSON.parse(response.SecretString);
  return cachedSecrets;
}

// 使い方
async function main() {
  const secrets = await getSecrets();  // 1回だけ Secrets Manager に問い合わせる
  
  const openai = new OpenAI({ apiKey: secrets.OPENAI_API_KEY });
  const db = new Database({ password: secrets.DB_PASSWORD });
  // 以降は cachedSecrets を使うため Secrets Manager への問い合わせ不要
}
```

```python
# Python 版
import boto3, json

client = boto3.client('secretsmanager', region_name='ap-northeast-1')
_cached_secrets = None  # キャッシュ

def get_secrets():
    global _cached_secrets
    if _cached_secrets:
        return _cached_secrets
    
    response = client.get_secret_value(SecretId='myapp/production')
    _cached_secrets = json.loads(response['SecretString'])
    return _cached_secrets
```

### AWS コンソールでの設定手順

```
1. AWS コンソール → Secrets Manager → 「シークレットを保存」
2. シークレットの種類: 「その他のシークレット」
3. キーと値を入力:
   OPENAI_API_KEY  : sk-proj-xxxxxxxx
   DB_PASSWORD     : supersecretpass
   STRIPE_API_KEY  : sk_live_xxxxxxxx
4. シークレット名: myapp/production
5. 保存
```

### コスト感

- 1シークレット = **月額 $0.40**（約60円）
- 10,000回の取得 = **月額 $0.05**（約7円）
- 月5シークレット、100回取得 = **約300円**

---

## CHAPTER 8：1Password CLI

### 1Password とは

個人・チーム向けの**パスワードマネージャー**です。  
CLIを使うと、ターミナルからVault（金庫）にアクセスできます。

### 通常の .env との違い

```
通常の .env:
  OPENAI_API_KEY=sk-proj-xxxxxxxx  ← 実際の値がファイルに書かれている
  DB_PASSWORD=supersecretpass       ← .gitignore していても AI に読まれる可能性

1Password CLI を使った .env.template:
  OPENAI_API_KEY=op://MyVault/OpenAI/api-key    ← 1Password の参照パス
  DB_PASSWORD=op://MyVault/Database/password      ← 実際の値はここにない
         ↓ op run コマンドで起動
  op run --env-file=.env.template -- node server.js
         ↓ 起動時に自動で値が注入される
  process.env.OPENAI_API_KEY  = 'sk-proj-xxxxxxxx'  （メモリ内のみ）
```

### インストールと基本操作

```bash
# macOS
brew install 1password-cli

# ログイン（一度だけ）
op signin

# シークレットを読む
op read "op://MyVault/OpenAI/api-key"

# .env.template を使ってアプリを起動
op run --env-file=.env.template -- node server.js
op run --env-file=.env.template -- python app.py
```

### .env.template（git にコミットしてよい）

```bash
# .env.template
# このファイルは git にコミットしてチームで共有できる
# 実際の値は 1Password に保存されている

OPENAI_API_KEY=op://MyVault/OpenAI/api-key
ANTHROPIC_API_KEY=op://MyVault/Anthropic/api-key
DB_PASSWORD=op://MyVault/Database/production-password
STRIPE_API_KEY=op://MyVault/Stripe/secret-key
```

### チームでの使い方

```
1. 1Password でチーム用 Vault（金庫）を作成
2. チームメンバーを Vault に招待
3. .env.template を git にコミット
4. 各自が自分のアカウントで op signin してから op run で起動
5. 実際のキーはVaultで一元管理
   → 誰かが退職したら Vault のアクセスを削除するだけ
```

---

## CHAPTER 9：次回ミーティング準備チェックリスト

### 必ず説明すること（優先度：高）

- [ ] **3人の登場人物の図を見せる**：頭脳・作業部屋・受付係の違いを最初に確認する
- [ ] **「頭脳はプロジェクト外も読める」を実演**：settings.json の deny ルールを設定して見せる
- [ ] **Credential Proxy の正しいたとえ**：「探しに行く」ではなく「最初から持っている」

### 動物病院向けに用意すること（優先度：高）

- [ ] **データ仕分け表を1枚作る**：カルテのどの項目をAIに渡せて、どれは渡せないか
- [ ] **マスキングのデモ**：架空のカルテデータでマスキング処理を実演する
- [ ] **「AIに渡せる最小限データ」の定義**：ペットID・症状カテゴリ・年齢区分など

### 環境設定として実演すること（優先度：中）

```json
// .claude/settings.json（実際に設定して見せる）
{
  "permissions": {
    "deny": [
      "Read(./.env*)",
      "Read(./secrets/*)",
      "Read(~/.ssh/*)",
      "Read(~/.aws/credentials)",
      "Bash(curl * | bash)",
      "Bash(wget * -O- | sh)"
    ]
  }
}
```

### 将来的に導入を提案すること（優先度：低〜中）

- [ ] AWS Secrets Manager または 1Password CLI への移行計画
- [ ] 本番環境でのアクセスログ取得設定
- [ ] APIキーのローテーション（定期更新）スケジュール

### 一言で伝えるメッセージ

> **「AIエージェントを使う時は、AIに渡すデータを絞ること。AIは賢いが、渡したものは見る。渡さなければ見えない。」**

---

## まとめ：今すぐできる対策

| 対策 | 難易度 | 効果 | いつやるか |
|------|-------|------|---------|
| settings.json に deny ルール追加 | ★ 簡単 | .env 読み取り防止 | **今すぐ** |
| .claudeignore 設定 | ★ 簡単 | AI の参照範囲制限 | **今すぐ** |
| APIキーを作業フォルダ外の .env に移動 | ★★ 普通 | 誤読リスク低減 | **今週中** |
| OS 環境変数への移行 | ★★ 普通 | ファイルレス化 | **今月中** |
| マスキング関数の実装 | ★★ 普通 | 個人情報漏洩防止 | **開発前に** |
| Credential Proxy の導入 | ★★★ 難しい | APIキー完全隔離 | **本番前に** |
| AWS Secrets Manager 導入 | ★★★ 難しい | エンタープライズ級管理 | **本番前に** |

---

*参考：OWASP LLM Top 10 / Anthropic Security Best Practices / AWS Secrets Manager Documentation*
