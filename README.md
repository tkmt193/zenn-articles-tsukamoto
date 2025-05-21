# zenn-articles-tsukamoto
個人用のZenn記事管理用リポジトリ


## 参考
https://zenn.dev/zenn/articles/install-zenn-cli


## Zenn CLIのインストール
```bash
$ npm init --yes # プロジェクトをデフォルト設定で初期化
$ npm install zenn-cli # zenn-cliを導入
$ npm install zenn-cli@latest # zenn-cliをアップデート
```

## Zenn用のセットアップ
```bash
$ npx zenn init # zenn-cliの初期化
```

## 記事を作成する
```bash
$ npx zenn new:article --slug github-zenn-linkage-20250521 # 記事を作成
```

## 記事をプレビューする
```bash
$ npx zenn preview # プレビュー開始
$ 👀 Preview: http://localhost:8000
```