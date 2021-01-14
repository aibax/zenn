---
title: "Vessel で Laravel のステップ実行が止まらなくなったら Xdebug のバージョンを確認しよう"
emoji: "🐛"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [ "PHP", "xdebug", "Laravel", "vessel" ]
published: true
---

# 概要
Vessel の PHP 環境にインストールされる Xdebug のメジャーバージョンアップによって、リモートでバッグ時のステップ実行が止まらなくなった。
ステップ実行をするためには、Vessel の設定を Xdebug 3 に対応した内容に変更する必要がある。

## Vessel とは
Vessel は Docker を使った Laravel のアプリケーション開発環境

Vessel - Docker dev environments for Laravel
https://vessel.shippingdocker.com/

## Xdebug とは
PHPのデバッグ用拡張モジュール

Xdebug - Debugger and Profiler Tool for PHP
https://xdebug.org/


# ステップ実行が止まらなくなった原因
- Vessel は環境を作る際に PECL で最新版の Xdebug をインストールしている。
- 2020年11月25日に Xdebug 3 がリリースされた。
- Xdebug 3 から php.ini で指定するリモートデバッグの設定が変更されたが、Vessel は Xdebug 2 の設定になっている。
- Xdebug 3 リリース以降にビルドした Vessel の開発環境は、設定を変更しないと Xdebug でリモートデバッグができなくなった。


# Xdebug 3 で何が変わったのか

Upgrading from Xdebug 2 to 3
https://xdebug.org/docs/upgrade_guide

以下、ポイントだけまとめると、

## 1. ステップ実行を有効にする設定が変わった
Xdebug 2 : `xdebug.remote_enable = 1`
Xdebug 3 : `xdebug.mode=debug`

## 2. Xdebugから接続するアドレスの設定項目が変わった
Xdebug 2 : `xdebug.remote_host = ホスト名/IPアドレス`
Xdebug 3 : `xdebug.client_host = ホスト名/IPアドレス`

## 3. Xdebugから接続するポートの設定項目とポート番号が変わった
Xdebug 2 : `xdebug.remote_port = 9000`
Xdebug 3 : `xdebug.client_port = 9003`

## Xdebug 3 の設定コンセプト

機能ごとに設定で有効化していた Xdebug 2 に対して、Xdebug 3 では「モード」を指定することで目的の状態に設定するという考え方だそう。このモードを設定するのが `xdebug.mode` ということ。

あと、`remote_*` という設定項目が `client_*` になった。
Xdebug 2 の時は、手元の開発環境は Xdebug から見た「リモート」という意味だったのに対して、
Xdebug 3 は Xdebug が「サーバ」で動作していて開発しているローカル環境が「クライアント」という意味なのだろう。
直感的で分かりやすくなったと思うが、互換性が...


# Vessel の設定を Xdebug 3 に対応させる手順

## Vessel の開発環境を作成

まずは通常通りの手順で Vessel の開発環境を作成する。

↓公式サイトの手順はこちら

Getting Started
https://vessel.shippingdocker.com/docs/get-started/

開発環境が作成できたらアプケーションが正しく動作することを確認しておく。

## Xdebug の設定を変更する

### 設定を変更するファイル
`docker/app/xdebug.ini` 

```
% tree
.
├── LICENSE
├── README.md
├── app
├── artisan
├── bootstrap
├── composer.json
├── composer.lock
├── config
├── database
├── docker
│   ├── app
│   │   ├── Dockerfile
│   │   ├── default
│   │   ├── h5bp
│   │   ├── start-container
│   │   ├── supervisord.conf
│   │   ├── vessel.ini
│   │   └── xdebug.ini          ← このファイルを編集する
│   ├── mysql
│   │   ├── conf.d
│   │   └── logs
│   └── node
│       └── Dockerfile
├── docker-compose.yml
├── package.json
├── phpunit.xml
├── public
├── resources
├── routes
├── server.php
├── storage
├── tests
├── vendor
├── vessel
└── webpack.mix.js
```

### Xdebug 3 の設定内容

```
zend_extension=xdebug.so
xdebug.remote_enable=1
xdebug.remote_handler=dbgp
xdebug.remote_port=9000
xdebug.remote_autostart=1
xdebug.remote_connect_back=0
xdebug.idekey=docker
xdebug.remote_host=192.168.1.2
xdebug.max_nesting_level = 500
; ↑ XDebug 2 の設定

; XDebug 3 の設定 ※ここを追加する
xdebug.mode = debug
xdebug.client_host = host.docker.internal ← コンテナから見たホストのIPアドレス
; xdebug.client_port = 9000 ← XDebug 2 と同じポート 9000 に接続する場合はここで指定
```
`host.docker.internal` はコンテナから見たホストのIPアドレスを自動解決する設定
https://docs.docker.jp/docker-for-mac/networking.html

## 変更した設定を反映する
`xdebug.ini` の設定を反映させるためにはコンテナをビルドして再起動する。

```
./vessel stop
./vessel build
./vessel start
```

## IDE で必要な設定を行う

Xdebug 3 が接続してくるポートが `9000` から `9003` に変更になったので、IDE 側で待ち受けるポート番号も `9003` に変えておく。

または、`xdebug.ini` で Xdebug 3 の接続ポートを `9000` に変えてもよい。

### PhpStorp

Configure Xdebug - PhpStorm
https://www.jetbrains.com/help/phpstorm/configuring-xdebug.html

### Visual Studio

デバッグ構成ファイルの設定例
https://github.com/aibax/vessel-xdebug3-sample/blob/main/.vscode/launch.json


# 参考

この件についてサンプルコードを書きました
https://github.com/aibax/vessel-xdebug3-sample
