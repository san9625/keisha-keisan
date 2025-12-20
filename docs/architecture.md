# Architecture & Requirements - Keisha（傾斜割り勘）

## 1. Goal
飲み会の割り勘を「役職/ランクに応じて傾斜」させて計算し、スマホでその場で結果を見てスクリーンショットで保存できるようにする。

## 2. Users / Use Case
### Primary User
- 飲み会の幹事（スマホで入力する）
- その場で素早く計算して、参加者に提示する

### Usage Flow
1. 合計金額（円）を入力
2. 傾斜率（急/普通/なだらか）を選択
3. ランクごとの人数を入力（初期5段、必要なら最大10段まで追加）
4. 「計算する」を押下
5. 各ランク1人あたりの支払額が表示される（スクショで保存）

## 3. Functional Requirements
### Inputs
- totalAmount: number（円）
- slopeType: enum = { steep, normal, gentle }
- ranks: 1..10（rank=1が偉い・支払いが最も安い）
- peopleCount[rank]: number（各ランク人数、0は除外）

### Outputs
- payPerPerson[rank]: number（円、整数）
- subtotal[rank] = payPerPerson[rank] * peopleCount[rank]
- totalCalculated = Σ subtotal[rank]
- totalCalculated must equal totalAmount（必須）

### UX/UI
- スマホの入力を優先（大きめフォーム、スクロール少なめ）
- 結果がスクショしやすい（余白、見やすい表）
- 結果の保存/送信/エクスポートは不要

## 4. Non-Functional Requirements
- Low cost & low ops（サーバレス/静的）
- High availability（CloudFront配信）
- Security: S3は非公開（Block Public Access ON）、CloudFront OAC経由のみアクセス可
- SEO: title/meta/description/見出しをHTMLに含める（SPAで空DOMにしない）

## 5. Calculation Specification
### 5.1 Weight Model（傾斜率の表現）
各ランクに重み weight を割り当てる。
- rank=1: weight=1.0
- rankが大きいほどweightが増える（=支払いが増える）

傾斜率ごとの base 値（例）
- super steep:  base = 1.60
- steep:  base = 1.40
- normal: base = 1.20
- gentle: base = 1.15

weight(rank) = base^(-(rank-1))

### 5.2 Core Formula
対象ランク集合 A = { rank | peopleCount[rank] > 0 }

W = Σ (peopleCount[r] * weight[r]) for r in A
unit = totalAmount / W

rawPay[r] = unit * weight[r]
pay[r] = floor(rawPay[r])  // 仮値

subtotal = Σ (pay[r] * peopleCount[r])
diff = totalAmount - subtotal

### 5.3 Rounding & Adjustment（合計一致のための1円調整）
diff を 0 にするまで、pay[r] を 1円単位で調整する。

基本方針（実用優先）：
- diff > 0（不足）: 高いランク側を優先して +1 円する（または小数部が大きい順）
- diff < 0（超過）: 低いランク側を優先して -1 円する（payが0未満にはしない）
- いずれも「pay[r] を±1すると合計が peopleCount[r] 分だけ変わる」点に注意する

制約：
- pay[r] >= 0
- Σ(pay[r] * peopleCount[r]) == totalAmount（最終的に必ず一致）

※将来的に「最適化（誤差最小）」を厳密にやりたくなったら、
「diff を peopleCount の組合せで埋める」問題になるため、別途アルゴリズム検討する。
現段階は「速い・分かりやすい・確実に合計一致」を優先する。

## 6. System Architecture
### 6.1 Overview（静的配信）
- This is a static website (no backend required).
- Hosting: S3 + CloudFront
- Custom domain: keisha.sakuraproduct.com
- TLS: ACM certificate in us-east-1 (required by CloudFront)

### 6.2 AWS Components
- S3
  - Store static assets (`index.html`, optional `/assets/*`)
  - Block Public Access: ON
- CloudFront
  - Origin: S3 (REST endpoint)
  - Origin Access Control (OAC): enabled
  - Default root object: `index.html`
  - Alternate domain name: `keisha.sakuraproduct.com`
  - Custom SSL certificate: ACM (us-east-1, Issued)
- DNS（external）
  - CNAME: `keisha.sakuraproduct.com` -> `<distribution>.cloudfront.net`

### 6.3 Security
- S3 bucket must not be public.
- Access is allowed only from CloudFront distribution via OAC policy.

## 7. Deployment & Operations
### 7.1 Deployment
- Upload `index.html` (and assets) to S3.
- (Optional) CloudFront invalidation:
  - `/*` when immediate update is needed
- Otherwise rely on TTL.

### 7.2 Monitoring (Optional)
- CloudFront access logs (optional)
- No server-side runtime monitoring required.

## 8. Future Extensions (Out of Scope)
- URLでパラメータ共有
- 履歴保存（DynamoDB等）
- 多言語対応（EN/JP）
- 広告表示（AdSense）
