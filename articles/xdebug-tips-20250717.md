---
title: "Xdebugのえらー"
emoji: "😎"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---

- Xdebug2→3へのアップデートによってエラーが発生

原因
IDE（VSCodeやPHPStorm）上でデバッガーを起動していない
→ 9003のインスタンス（デバッガーを起動しているインスタンス）がみつからないよと言われている

なぜ？
startmode=yesになっている
→ 常時デバッガーに接続する設定になっている
→ 常にデバッガーを起動しているインスタンスに接続しようとしている

えらー出さないためには次の二通方法がある
startmode=trigger
→ 必要な時だけデバッガーを接続する設定にする

level=0
エラーレベルを下げて、エラーログを出力しない

startmode=trigger
→ デバッガーを起動する方法
このままだとデバックが動かない
トリガーを渡す必要がある
POST COOKIE ENV
例）ブラウザ・シェル
- jetbrainsの拡張機能が使えそう

参考になったサイト