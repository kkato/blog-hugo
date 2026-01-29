---
title: "PostgreSQL16をソースコードからビルドしてみる"
date: 2024-09-18T14:47:07+09:00
draft: false
tags: ["postgresql"]
---

PostgreSQL16をソースコードからビルドしてみたのですが、いくつかつまづいたポイントがあるので、それらにご紹介します。

PostgreSQL16をソースコードからビルドするためのコマンドは以下のとおりです。前提として M1 Mac (macOS 14.4.1) を使っています。

```sh
% git clone git://git.postgresql.org/git/postgresql.git
% cd postgresql
% ./configure --enable-debug --enable-cassert --enable-tap-tests --prefix=$HOME/pgsql CFLAGS=-O0
% make -j 4
% make install
```
configureコマンドには以下のオプションを指定しています。
- デバッグしやすくするためのオプション: --enable-debug --enable-cassert --enable-tap-tests
- インストール先のディレクトリを指定するオプション: --prefix

参考: https://www.postgresql.jp/document/16/html/install-make.html#CONFIGURE-OPTIONS


### `icu-uc`, `icu-i18n`のパッケージが見つからない

`icu-uc`, `icu-i18n`のパッケージが見つからないと言われました。

```text
% ./configure --enable-debug --enable-cassert --enable-tap-tests --prefix=$HOME/pgsql CFLAGS=-O0
(省略)
checking for icu-uc icu-i18n... no
configure: error: Package requirements (icu-uc icu-i18n) were not met:

No package 'icu-uc' found
No package 'icu-i18n' found

Consider adjusting the PKG_CONFIG_PATH environment variable if you
installed software in a non-standard prefix.

Alternatively, you may set the environment variables ICU_CFLAGS
and ICU_LIBS to avoid the need to call pkg-config.
See the pkg-config man page for more details.
```

PKG_CONFIG_PATHという環境変数を設定してあげれば良いみたいです。
以下のコマンドを実行すると、どう設定すればいいか教えてくれました。

```sh
% brew info icu4c
(省略)
For pkg-config to find icu4c you may need to set:
  export PKG_CONFIG_PATH="/opt/homebrew/opt/icu4c/lib/pkgconfig"
```

言われた通りに設定しました。

```sh
echo 'export PKG_CONFIG_PATH="/opt/homebrew/opt/icu4c/lib/pkgconfig"' >> ~/.zshrc
source ~/.zshrc
```

### TAPテストに必要なモジュールがインストールされていない

TAPテストに必要なPerlのmoduleがインストールされていなかったみたいです。

```text
checking for Perl modules required for TAP tests... Can't locate IPC/Run.pm in @INC (you may need to install the IPC::Run module) (@INC contains: /Library/Perl/5.34/darwin-thread-multi-2level /Library/Perl/5.34 /Network/Library/Perl/5.34/darwin-thread-multi-2level /Network/Library/Perl/5.34 /Library/Perl/Updates/5.34.1 /System/Library/Perl/5.34/darwin-thread-multi-2level /System/Library/Perl/5.34 /System/Library/Perl/Extras/5.34/darwin-thread-multi-2level /System/Library/Perl/Extras/5.34) at ./config/check_modules.pl line 14.
BEGIN failed--compilation aborted at ./config/check_modules.pl line 14.
configure: error: Additional Perl modules are required to run TAP tests
```


TAPテストに必要なPerlのmoduleをインストールしました。

参考: https://www.postgresql.jp/document/16/html/regress-tap.html

```sh
sudo cpan IPC::Run
```

### 参考
- [公式ドキュメント](https://www.postgresql.jp/document/16/html/install-make.html)
- [PostgreSQL開発コミュニティに参加しよう！](https://www.slideshare.net/slideshow/postgresql-global-development-group-osc2022-online-kyoto-nttdata/252356101)
