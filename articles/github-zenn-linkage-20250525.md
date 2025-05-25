---
title: "HomebrewをWSL2に導入し、aws, aws-vaultをインストールする"
emoji: "🔥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["homebrew", "wsl2", "wsl", "aws", "windows"]
published: false
---


最終更新日：2025/05/25

## はじめに
WSL2でAWS系のコマンドをインストールする際に、brewを使った方法が手軽だったので、その方法を備忘録的に残しておきます。


## 手順

まず、Homebrewをインストールするために必要なパッケージをインストールします。

WSL上で下記を実行します。

```bash
# パッケージリストを更新
$ sudo apt update

# 必要な依存パッケージをインストール
$ sudo apt-get install build-essential curl file git
```

次に、Homebrewをインストールします。

[Homebrew公式サイト](https://brew.sh/ja/)に従ってWSL上で下記を実行します。

```bash
 /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install
/HEAD/install.sh)"
```

すると、下記のようにWarningが表示されます。

```jsx
Warning: /home/linuxbrew/.linuxbrew/bin is not in your PATH.
  Instructions on how to configure your shell for Homebrew
  can be found in the 'Next steps' section below.
==> Installation successful!

==> Homebrew has enabled anonymous aggregate formulae and cask analytics.
Read the analytics documentation (and how to opt-out) here:
  https://docs.brew.sh/Analytics
No analytics data has been sent yet (nor will any be during this install run).

==> Homebrew is run entirely by unpaid volunteers. Please consider donating:
  https://github.com/Homebrew/brew#donations

==> Next steps:
- Run these commands in your terminal to add Homebrew to your PATH:
    echo >> ~/.bashrc
    echo 'eval "$(/home/linuxbrew/.linuxbrew/bin/brew shellenv)"' >> ~/.bashrc
    eval "$(/home/linuxbrew/.linuxbrew/bin/brew shellenv)"
- Install Homebrew's dependencies if you have sudo access:
    sudo apt-get install build-essential
  For more information, see:
    https://docs.brew.sh/Homebrew-on-Linux
- We recommend that you install GCC:
    brew install gcc
- Run brew help to get started
- Further documentation:
    https://docs.brew.sh
```

このままだとbrewコマンドを実行できないので、Wariningの指示に従っていきます。

下記を実行し、HomebrewにPATHを追加します。

```bash
$ echo >> ~/.bashrc
$ echo 'eval "$(/home/linuxbrew/.linuxbrew/bin/brew shellenv)"' >> ~/.bashrc
$ eval "$(/home/linuxbrew/.linuxbrew/bin/brew shellenv)"
```

依存パッケージをインストールします。

```bash
$ sudo apt-get install build-essential
```

GCCを入れておくことをオススメされているので、いったん入れておきます。

```bash
 $ brew install gcc
```

下記を実行し、brewコマンドが実行できることを確認します。

```bash
$ brew -v
Homebrew 4.5.2 #インストールしたHomebrewのバージョン
```

下記を実行し、awscli  aws-vaultをインストールします。

```bash
$ brew install awscli aws-vault
```

これでインストールは完了です。
