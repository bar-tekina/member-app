# Barてきな？ 会員管理アプリ

網走のダーツ＆ゲームバー「Barてきな？」用の会員管理・会計（レジ）アプリ。
もとは Claude Artifact（`window.claude.db` を使った内蔵ストレージ）上で動いていたものを、
GitHub Pages ＋ 実際の Firebase（Firestore）に移行したもの。見た目・機能は移行前と同一で、
バックエンドの保存先だけを置き換えている。

## できること

- 会員一覧（VIP / GOLD / REGULAR ランク表示）
- 会計（レジ）: コース料金・単品・延長・追加メニュー・その他金額の入力と記録
- 新規登録（本人確認書類の撮影必須、紹介者設定）
- 誕生日タブ（今月誕生日の会員一覧）
- 売上タブ、分析タブ（月別・年別・曜日別・前年比・ランク別・紹介・休眠会員など）
- 会員ごとの入店QR（`#m<会員ID>` のハッシュで開くと自動で会計対象にセット）
- 公開自己登録ページ（`#join` のハッシュで開くと、PIN・スタッフコードなしで
  新規登録フォームのみを表示。お客様自身のスマホで登録してもらうためのQR用リンク）
- オーナーPIN・スタッフコードによるタブ・機能の制限（クライアント側JSチェック）
- オーナーがアプリ内から料金・追加メニューを編集可能

## 技術構成

- 素のHTML/CSS/JS（フレームワークなし）の1ファイル（`index.html`）
- Firebase JS SDK v10（compat版）を CDN から読み込み、Firestore に直接接続
  - `https://www.gstatic.com/firebasejs/10.14.1/firebase-app-compat.js`
  - `https://www.gstatic.com/firebasejs/10.14.1/firebase-firestore-compat.js`
- QRコード生成: qrcodejs（cdnjs）、QRコード読み取り: jsQR（cdnjs）
- フォント: Google Fonts（Bebas Neue, Noto Sans JP）

### Firestore のコレクション構成

- `members`: 会員データ（名前・連絡先・ランク計算用ポイント・直近15件の来店履歴・
  本人確認書類の画像（圧縮済みJPEGのdata URL）など）
- `guestSales`: 非会員（一般客）の会計記録
- `visitLog`: 会員の来店履歴の全件保存（`members.*.visits` は直近15件しか保持しないため、
  売上・分析タブの集計はこちらを正として使う）
- `settings/staffAccess`: スタッフ専用コード（`code` フィールド。初回起動時に自動発行）
- `settings/pricing`: コース・メニューの料金設定、追加メニュー項目
- `settings/app`: オーナーPIN（`ownerPin` フィールド。初回起動時に初期値 `6188` を自動発行）

## デプロイ方法（GitHub Pages）

1. このリポジトリの `main` ブランチにプッシュする。
2. GitHub の当該リポジトリ → Settings → Pages で、Source を
   「Deploy from a branch」、Branch を `main` / `/ (root)` に設定する
   （`.nojekyll` が置いてあるので Jekyll 処理によるファイルの欠落は起きない）。
3. 数分後、`https://bar-tekina.github.io/member-app/` のようなURLで公開される
   （実際のURLはリポジトリの Pages 設定画面に表示される）。
4. スタッフ用には、そのURLに `#<スタッフコード>` を付けたQRコードを配布する。
   会員証には `#m<会員ID>` 付きのQR（アプリ内「入店QRを表示」から発行）、
   会員登録用には `#join` 付きのQR（お客様自身のスマホで開く）をそれぞれ使う。

## Firestore セキュリティルールのトレードオフ（重要）

`firestore.rules` にある通り、**現状はすべてのコレクション・ドキュメントで
読み書きを無条件に許可している**（Firebase Authenticationを導入していないため）。

これはつまり、このアプリのURL・Firebase設定（`firebaseConfig`、本READMEにも
記載の値）を知っていて、かつブラウザの開発者ツールなどで直接 Firestore に
アクセスしようとする悪意あるユーザーがいれば、理論上はデータを直接読んだり
書き換えたりできてしまう、ということを意味する。

ただし、

- `apiKey` を含む Firebase の設定値は、そもそも「秘密鍵」ではなく
  （クライアントサイドのWebアプリでは必ず誰でも読める値であり、Googleもそれを
  前提にセキュリティルール側で保護する設計を取っている）、
- このアプリは1店舗のみで使う小規模な会員管理アプリであり、
- `#join` の公開登録ページ自体が「ログインなしの誰でも書き込み」を前提にした
  機能であること

を踏まえると、今回は「実質的なリスクは低く、Firebase Authenticationを追加で
導入する複雑さに見合わない」と判断し、オープンなルールのまま運用することにした。
これは Claude Artifact 版（Claude組織のメンバーなら誰でも読み書き可能だった）と
同程度のアクセス制御レベルであり、実質的なリスクプロファイルは変わっていない。

将来、セキュリティを強化したくなった場合は、Firebase Authentication
（匿名認証や、スタッフ用のメール/パスワード認証など）を導入したうえで、
`firestore.rules` の `allow read, write: if true;` を `request.auth != null` 等の
条件に置き換えることを推奨する。

## Firebase設定を変更する場合

`index.html` 内、`<script>(function () { ... })()</script>` の先頭付近にある
以下の定数を書き換える（Firebaseコンソールの「プロジェクトの設定」から
再取得できる）。

```js
const firebaseConfig = {
  apiKey: "...",
  authDomain: "...",
  projectId: "...",
  storageBucket: "...",
  messagingSenderId: "...",
  appId: "...",
};
```

変更後は `git add -A && git commit -m "..." && git push origin main` で
GitHub Pages に反映される（反映までに数分かかる場合がある）。

## 本人確認書類の画像について

本人確認書類の画像は、撮影時にブラウザ上で最大辺1000px・JPEG品質0.7程度まで
圧縮してから `members.{id}.idDocument.image` に data URL（base64）として
保存している。これは Firestore の1ドキュメントあたり1MBというサイズ上限を
超えないようにするための対策で、移行前の実装から変更していない
（実運用では数百KB程度に収まる想定）。非常に大きな写真を複数枚保存するような
用途には向かないため、もし将来的に高解像度の画像保存が必要になった場合は
Firebase Storage（別サービス）へ画像本体を置き、Firestoreにはその参照URLだけを
持たせる構成への変更を検討すること。
