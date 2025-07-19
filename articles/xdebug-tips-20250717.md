---
title: "「Xdebug: [Step Debug] Could not connect to ... 」が出た時のメモ"
emoji: "😎"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---

## 📗 背景

Xdebug 2 から 3 にアップグレードしたところ、以下のようなエラーが発生するようになりました。

```text
Xdebug: [Step Debug] Could not connect to debugging client. Tried: host.docker.internal:9003 (through xdebug.client_host/xdebug.client_port) :-(
````

下記の場合などで困ったので、一旦原因を調査してみることにしました。

* PHPUnitを実行する際などに、上記のエラーが原因でテストが落ちてしまう
* 単純にアプリケーションログに混ざってノイズになっている

🔗 [Xdebug公式ドキュメント：Xdebug 2系 から 3系 へのアップグレードについて](https://xdebug.org/docs/upgrade_guide/ja)

---

この時、`php.ini` 内の Xdebug の設定は以下のように設定していました。

```text:php.ini
xdebug.start_with_request=yes
...
```

---

## 🧠 原因

このエラーの主な原因は、**IDE（VSCode や PHPStorm）でデバッガーを起動していないこと**です。

Xdebug がポート `9003` 上で待機しているデバッガーに接続できず、「見つからない」と怒られています。

なぜこうなるのかというと、`php.ini`上に記載しているXdebugの設定が原因になります。

より詳細に説明すると、

`php.ini` にて `xdebug.start_with_request=yes` が設定されている

→ Xdebug が、常時デバッガーへ接続しようとする設定になっている

→ そのため、**IDE側でデバッガーが起動していない**と、接続に失敗してエラーになる

といった状況です。

🔗 [Xdebug公式ドキュメント：xdebug.start\_with\_request](https://xdebug.org/docs/all_settings#start_with_request)

---

## 🛠 対処方法

このエラーを回避するためには、以下のいずれかの方法を取ることができそうです。

### ✅ 1. 必要なときだけ Xdebug をデバッガーへ接続する設定にする

`php.ini` で以下のように設定します：

```text:php.ini
xdebug.start_with_request=trigger
...
```

この設定により、**明示的なトリガー**があるときだけ、Xdebug はデバッガーへ接続を試みます。

#### 💡 トリガーの方法

`$_ENV` / `$_GET` / `$_POST` / `$_COOKIE` に `XDEBUG_TRIGGER` を含めることで、デバッガーを起動することができます。

🔗 [Xdebug公式ドキュメント：xdebug.start\_with\_request → trigger](https://xdebug.org/docs/all_settings#start_with_request#trigger)

トリガーに設定する文字列（`XDEBUG_TRIGGER`）についても、独自にカスタマイズする事ができます。
🔗 [Xdebug公式ドキュメント：trigger\_value](https://xdebug.org/docs/step_debug#trigger_value)


#### 例：
`XDEBUG_TRIGGER`を、`$_ENV` / `$_GET` / `$_POST` / `$_COOKIE` に含めてトリガーする方法をいくつか紹介します。

* JetBrains製のブラウザ拡張（例：**Xdebug Helper**）を使う方法
  ↳ [公式ガイド](https://pleiades.io/help/phpstorm/browser-debugging-extensions.html#xdebug-helper-extension)

* クエリパラメータで明示する方法
  ↳ `?XDEBUG_TRIGGER=1`

* シェルからPHPファイルを実行する方法

```bash
XDEBUG_TRIGGER=1 php index.php
```

---

### 🚫 2. エラーログを出力しないようにする

Xdebug 側でログレベルを下げることで、接続失敗のエラーログを抑制することも可能です。

```text:php.ini
xdebug.log_level=0
...
```

また、エラーログの出力先を変更することで、アプリケーションログに影響を与えないようにすることも可能です。

```text:php.ini
xdebug.log = /tmp/xdebug.log
...
```

⚠️ ただし、根本的な解決にはならないので注意です。

---

## 🔍 参考になったサイト
上記の調査をする際に、下記の方のブログが大変参考になりました🙏
* [https://zenn.dev/naoyukik/books/jetbrains-ides-guide/viewer/xdebug#発動タイミング制御](https://zenn.dev/naoyukik/books/jetbrains-ides-guide/viewer/xdebug#発動タイミング制御)
* [https://lazesoftware.com/ja/blog/210919/](https://lazesoftware.com/ja/blog/210919/)

