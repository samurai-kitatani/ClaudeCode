# パスワード・認証セキュリティガイド
## 開発者のための「正しい認証」完全解説

**対象**：Webアプリ・AIツールを開発・利用するエンジニア・経営者  
**目的**：パスワード保存・2FA・JWT・OAuthの正しい実装と利用を理解する

---

# PART 1｜パスワードの正しい保管方法

---

## ❌ よくある間違い：パスワードをそのまま保存する

```
ユーザーが入力：  password123
DBに保存される：  password123   ← 平文保存（絶対NG）
```

**なぜ危険か：**

```
DBが漏洩した場合：
  → 攻撃者がすべてのパスワードをそのまま読める
  → ユーザーは他サービスでも同じパスワードを使っていることが多い
  → 1箇所の漏洩で、複数サービスが乗っ取られる（パスワードの使い回し被害）
```

---

## ❌ MD5・SHA-1 でのハッシュ化も危険

「暗号化しています」と言いながらMD5を使っているケースが多く見られます。

```
MD5の問題点：

① 計算が速すぎる
  → 現代のGPUは1秒間に1,000億個以上のMD5ハッシュを計算できる
  → "password123"のMD5は数ミリ秒で解読できる

② レインボーテーブル攻撃に弱い
  → よく使われるパスワードのMD5ハッシュ一覧（レインボーテーブル）が
    ネットに大量に存在する
  → MD5ハッシュを検索サイトに貼るだけで元のパスワードが判明する
    例）482c811da5d5b4bc6d497ffa98491e38 → "password123"（即座に判明）

③ 衝突脆弱性
  → 異なる入力が同じMD5ハッシュになる組み合わせが存在する（数学的に証明済み）
```

---

## ✅ 正しいパスワードハッシュ：bcrypt / Argon2

```
パスワード保存の正しい方法：
  「意図的に遅いハッシュ関数」を使う

bcrypt / Argon2 の特徴：
  → 計算に時間がかかるよう設計されている（0.1〜1秒程度）
  → ハッシュのたびに「ソルト（ランダムな文字列）」が追加される
  → 同じパスワードでも毎回違うハッシュ値になる（レインボーテーブル無効化）
  → 将来的に「コスト係数」を上げるだけで強度を高められる
```

### bcrypt の実装例（Node.js）

```javascript
const bcrypt = require('bcrypt');

// パスワード保存時（新規登録・パスワード変更）
async function hashPassword(plainPassword) {
  const saltRounds = 12;  // コスト係数：高いほど安全・高いほど遅い
  return await bcrypt.hash(plainPassword, saltRounds);
}

// パスワード照合時（ログイン）
async function verifyPassword(plainPassword, hashedPassword) {
  return await bcrypt.compare(plainPassword, hashedPassword);
  // 正しければ true、間違いなら false
}

// 使い方
const hash = await hashPassword('password123');
// → "$2b$12$K8GpCsP7Uq7..." （毎回異なる値）

const isValid = await verifyPassword('password123', hash);
// → true
```

### ハッシュ関数の比較

| 方式 | 推奨度 | 解読難易度 | 用途 |
|------|--------|-----------|------|
| **Argon2id** | ✅ 最推奨 | 非常に高い | 新規開発はこれ |
| **bcrypt** | ✅ 推奨 | 高い | 多くのフレームワークで標準 |
| **scrypt** | ✅ 推奨 | 高い | メモリ負荷も高い |
| SHA-256 | ⚠️ NG | 低い（速すぎる） | パスワード保存には使わない |
| MD5 | ❌ 絶対NG | 非常に低い | 使ってはいけない |
| 平文保存 | ❌ 論外 | ゼロ | 論外 |

---

# PART 2｜二要素認証（2FA / MFA）

---

## 2FA とは何か

```
通常のログイン：「知っているもの」だけで認証
  → パスワード1つ

二要素認証（2FA）：「知っているもの」＋「持っているもの」で認証
  → パスワード ＋ スマートフォンのワンタイムコード

パスワードが漏洩しても、スマートフォンがなければログインできない
```

## 2FA の種類（安全順）

| 方式 | 安全性 | 使いやすさ | 説明 |
|------|--------|-----------|------|
| **ハードウェアキー（YubiKey等）** | 🔴 最高 | 低め | 物理デバイス。フィッシングに完全耐性 |
| **認証アプリ（TOTP）** | 🟡 高い | 中 | Google Authenticator、Authy等。30秒ごとに変わるコード |
| **パスキー（Passkey）** | 🟡 高い | 高い | 指紋・顔認証でログイン。パスワード不要 |
| **メールOTP** | 🟢 中 | 高い | メールにコードが届く。メール乗っ取りリスクあり |
| **SMS OTP** | 🟢 低め | 高い | SMSでコード。SIMスワップ攻撃に弱い |

> **開発者として覚えておくこと：**  
> SMS認証（電話番号へのコード送信）は便利ですが最も脆弱です。  
> 重要なサービスには**認証アプリ（TOTP）以上**を使ってください。

---

## TOTP の仕組み（認証アプリの仕組み）

```
TOTP = Time-based One-Time Password（時刻ベースのワンタイムパスワード）

仕組み：
  ① サービスとスマートフォンで「シークレットキー」を共有（QRコードでセットアップ）
  ② 現在時刻 ＋ シークレットキー → 6桁のコードを生成（30秒ごとに変わる）
  ③ サーバー側でも同じ計算をして照合

コードが30秒ごとに変わる = 仮にコードを盗まれても30秒後には無効
```

### 開発者向け TOTP 実装（Node.js）

```javascript
const speakeasy = require('speakeasy');
const QRCode = require('qrcode');

// 2FAセットアップ（ユーザー登録時）
async function setup2FA(userId, userEmail) {
  const secret = speakeasy.generateSecret({
    name: `MyApp (${userEmail})`,
    length: 20
  });

  // QRコードを生成（ユーザーに表示してスキャンしてもらう）
  const qrCode = await QRCode.toDataURL(secret.otpauth_url);

  // シークレットキーをDBに保存
  await db.users.update({ id: userId }, { twofa_secret: secret.base32 });

  return { qrCode, secret: secret.base32 };
}

// ログイン時の2FA検証
function verify2FA(userId, userInputCode) {
  const user = await db.users.findById(userId);

  return speakeasy.totp.verify({
    secret: user.twofa_secret,
    encoding: 'base32',
    token: userInputCode,
    window: 1  // 前後30秒のズレを許容
  });
}
```

---

## SIMスワップ攻撃（SMS認証の弱点）

```
SMS認証が危険な理由：

① 攻撃者がキャリアショップに行く
② 「電話を無くしました。同じ番号のSIMを再発行してほしい」と偽装
③ あなたの電話番号が攻撃者のSIMに移される（SIMスワップ）
④ 以降、あなたのSMSはすべて攻撃者に届く
⑤ 「パスワードを忘れた」でSMS認証 → アカウント乗っ取り

実際の被害：
  → アメリカで年間1万件以上のSIMスワップ被害
  → 仮想通貨口座の大規模被害が多数報告
```

---

# PART 3｜JWT（JSON Web Token）とセッション管理

---

## JWT とは何か

```
ログイン後、「あなたは誰か」を証明するための「証明書」

従来のセッション方式：
  ① ログイン → サーバーがランダムなID（セッションID）を発行
  ② セッションIDをCookieで持つ
  ③ リクエストのたびにサーバーのDBでセッションIDを確認
  → サーバーの負荷が高い・スケールしにくい

JWT方式：
  ① ログイン → サーバーが署名入りの「トークン」を発行
  ② トークンをLocalStorageかCookieで持つ
  ③ リクエストのたびにトークンを送信 → サーバーが署名を検証（DBアクセス不要）
  → 高速・スケールしやすい
```

### JWT の構造

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VySWQiOjEyMywiZXhwIjoxNzE5OTk5OTk5fQ.abc123

↑ 3つの部分が「.」で区切られている

Header（ヘッダー）：   アルゴリズム情報（Base64エンコード）
Payload（ペイロード）： ユーザーID・権限・有効期限（Base64エンコード）
Signature（署名）：    改ざん検知用の署名（サーバーの秘密鍵で生成）
```

> **重要：JWTのPayloadは暗号化されていません。Base64デコードするだけで中身が読めます。**  
> パスワード・秘密情報をJWTに入れてはいけません。

### JWT の安全な実装

```javascript
const jwt = require('jsonwebtoken');

// トークン生成（ログイン時）
function generateToken(userId, role) {
  return jwt.sign(
    { userId, role },          // Payloadに入れる情報（最小限に）
    process.env.JWT_SECRET,    // 署名用の秘密鍵（.envやSecrets Managerで管理）
    { expiresIn: '15m' }       // 有効期限：短く設定（15分〜1時間）
  );
}

// リフレッシュトークン（有効期限を延長するため）
function generateRefreshToken(userId) {
  return jwt.sign(
    { userId, type: 'refresh' },
    process.env.JWT_REFRESH_SECRET,
    { expiresIn: '7d' }        // リフレッシュは長め（7日）
  );
}

// トークン検証（APIリクエスト時）
function verifyToken(token) {
  try {
    return jwt.verify(token, process.env.JWT_SECRET);
  } catch (err) {
    throw new Error('Invalid or expired token');
  }
}
```

### JWTのよくある間違い

```
❌ 間違い①：alg を "none" に設定できてしまう実装
   → 署名なしのJWTを受け入れてしまう脆弱性
   → 必ず使用アルゴリズムをサーバー側で固定する

❌ 間違い②：有効期限を長くしすぎる（30日など）
   → トークンが盗まれた場合、30日間悪用され続ける
   → アクセストークンは15分〜1時間、リフレッシュトークンで延長する

❌ 間違い③：LocalStorageに保存する
   → XSS攻撃でJavaScriptからLocalStorageが読める
   → HttpOnly Cookieに保存する（JavaScriptからアクセスできない）

❌ 間違い④：秘密鍵（JWT_SECRET）をGitにコミットする
   → 秘密鍵が漏洩すると任意のトークンを偽造できる
   → 必ず環境変数やSecrets Managerで管理する
```

---

# PART 4｜OAuth 2.0 / SSO（Googleログイン等）

---

## OAuth 2.0 とは何か

```
「Googleでログイン」「GitHubでログイン」の仕組み

ユーザーのパスワードを自社サービスが受け取らずに認証を行う

流れ：
  ① ユーザーが「Googleでログイン」をクリック
  ② Googleのログイン画面にリダイレクト
  ③ ユーザーがGoogleのパスワードを入力（自社サービスは見えない）
  ④ Googleが「この人は本物の山田太郎さんです」という証明書を発行
  ⑤ 自社サービスがその証明書を検証してログイン完了

メリット：
  → パスワードを自社DBに保管しなくていい
  → パスワード漏洩リスクがなくなる
  → ユーザーが新しいパスワードを覚えなくていい
```

### OAuth 2.0 の実装（Node.js / Express）

```javascript
const passport = require('passport');
const GoogleStrategy = require('passport-google-oauth20').Strategy;

passport.use(new GoogleStrategy({
  clientID:     process.env.GOOGLE_CLIENT_ID,      // Google Cloud Consoleで取得
  clientSecret: process.env.GOOGLE_CLIENT_SECRET,  // 絶対にコードに書かない
  callbackURL:  '/auth/google/callback'
}, async (accessToken, refreshToken, profile, done) => {
  // profileにGoogleアカウント情報が入っている
  const user = await db.users.findOrCreate({
    googleId: profile.id,
    email:    profile.emails[0].value,
    name:     profile.displayName,
  });
  return done(null, user);
}));
```

---

# PART 5｜AIツールアカウントの認証管理

---

## Claude / ChatGPT / Cursor を使うチームの注意点

### ❌ やってはいけないこと

```
❌ アカウントを複数人で共有する
   → 「誰が何をしたか」が追えない
   → 退職者のアクセスを即座に止められない
   → 1人の認証情報漏洩でチーム全員が危険に

❌ APIキーを1本だけ作って全員で使う
   → 漏洩時にどこから漏れたか分からない
   → 使用量・コストの追跡ができない

❌ APIキーをSlack・チャットに貼り付けて共有する
   → チャット履歴が残り、退職後も見られる
   → チャットサービスの漏洩でAPIキーが流出する
```

### ✅ 正しいチーム運用

```
✅ 個人アカウントを各人に発行する
   → 退職時は即座にそのアカウントだけ無効化できる

✅ APIキーはメンバーごとに別々に発行する
   → 用途・人別に分けることで、漏洩時の追跡と被害を最小化

✅ APIキーの共有は 1Password Teams や AWS Secrets Manager 経由にする
   → チャットやメールで生キーを送らない

✅ 定期的にAPIキーをローテーション（更新）する
   → 古いキーを無効化して新しいキーに切り替える習慣
   → AWS Secrets Manager は自動ローテーションをサポート
```

### Claude Code / Cursor の認証設定

```
企業でAIコーディングツールを使う場合の推奨設定：

1. APIキー管理
   → 個人のAPIキーを使うか、会社のAPIキーを1Passwordで共有
   → 必ずキー名に「用途-担当者名」を記載
     例）claudecode-yamada-prod, cursor-suzuki-dev

2. アクセス制限
   → IPアドレス制限が使えるサービスでは会社のIPに限定
   → 個人端末でのAIツール使用ポリシーを明文化する

3. 使用量監視
   → Claude / OpenAI のダッシュボードでAPIキー別の使用量を定期確認
   → 異常なアクセスを早期に発見できる
```

---

# PART 6｜パスワードマネージャーの選び方と使い方

---

## なぜパスワードマネージャーが必要か

```
人間が記憶できる安全なパスワードの数：現実的には5〜10個

必要なアカウント数：50〜200個以上

→ 結果として起きること：パスワードの使い回し
→ 1箇所で漏洩したパスワードが他サービスで使える（リスト型攻撃）

パスワードマネージャーを使えば：
→ サービスごとに異なる20文字以上のランダムパスワードを自動生成・記憶
→ 人間が覚えるのはマスターパスワード1つだけ
```

## 推奨パスワードマネージャー

| サービス | 対象 | 特徴 | 料金 |
|---------|------|------|------|
| **1Password** | 個人・チーム | UI最高・チーム共有機能・CLI連携 | 約¥450/月〜 |
| **Bitwarden** | 個人・企業 | オープンソース・無料プランあり | 無料〜¥100/月 |
| **Dashlane** | 個人 | ダークウェブ監視機能付き | 約¥500/月 |
| **1Password Teams** | チーム | 共有ボルト・管理機能充実 | $7.99/人/月 |

---

## 📌 このタブの結論

> **認証は「パスワードを保存する」だけでは終わりません。**  
> 保存方法・2FA・トークン管理・チーム運用のすべてがセキュリティに直結します。

| やること | 優先度 |
|---------|--------|
| パスワードをbcryptでハッシュ化する | 🔴 リリース前必須 |
| MD5・SHA-1によるパスワード保存をやめる | 🔴 今すぐ |
| 重要サービスに認証アプリ（2FA）を設定する | 🔴 今すぐ |
| JWTのアクセストークンを15〜60分に制限する | 🟡 今週中 |
| APIキーをメンバーごとに別々に発行する | 🟡 今週中 |
| チームでパスワードマネージャーを導入する | 🟡 今月中 |
| Googleログイン等のOAuth連携を検討する | 🟢 中期計画 |

> **一言まとめ：** 「MD5をやめてbcryptにする」と「全サービスに2FAを入れる」、  
> この2つが今すぐできる最も費用対効果の高いセキュリティ改善です。

---

*参考：OWASP Password Storage Cheat Sheet / NIST SP 800-63B / RFC 7519 (JWT) | 2026年版*
