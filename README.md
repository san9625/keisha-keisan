# Keisha - 飲み会の傾斜割り勘計算ツール

スマホでサクッと「傾斜割り勘」を計算し、結果をスクリーンショットで保存できるシンプルな静的Webサイトです。

- 入力：合計金額 / 傾斜率（急・普通・なだらか）/ ランク別人数
- 出力：各ランク1人あたりの支払額（合計金額と必ず一致）
- 配信：S3 + CloudFront（OACでS3は非公開）
- ドメイン：`https://keisha.sakuraproduct.com`

## Features
- 傾斜割り勘計算（1円単位の端数調整で合計一致）
- 傾斜率：急 / 普通 / なだらか を選択
- ランク：1（偉い=多く支払う）〜最大10（新人=支払い金額が少ない）、初期表示は5段、必要なら追加
- スマホ利用前提のUI（スクショ保存が主目的）

## Non-Goals（やらないこと）
- ログイン/ユーザ管理
- 計算結果の保存（DB）・メール送信・エクスポート
- 管理画面、課金、履歴機能

## Repository Structure（想定）
.
├── index.html # 1ファイル完結（HTML/CSS/JS）
├── assets/ # 画像などを置く場合
├── docs/
│ └── architecture.md # アーキテクチャ・要件詳細
└── README.md

## Local Development
静的ファイルなので、ローカルは簡単です。

- macOS:
  - `python3 -m http.server 8080`
  - `http://localhost:8080` にアクセス

## Deploy（概要）
S3に静的ファイルをアップロードし、CloudFrontで配信します。
詳細は `docs/architecture.md` を参照してください。
