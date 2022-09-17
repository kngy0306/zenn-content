---
title: "オンプレミスのDBサーバリプレイスでやったこと"
emoji: "😊"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["CentOS", "MySQL"]
published: true
---

# はじめに

先日業務にて、オンプレミスで動作しているDBサーバのリプレイス案件にメイン担当として関わりました。
OSのインストールだけがされているサーバにiptablesの設定、MySQLの構築、データの移行などを行いました。

DBサーバとして動作させるまでの構築手順を、自分用の振り返り、同じ作業を行う方への参考になればと思い記事を作成しました。

:::message
設定ファイルに記載した内容は、ほとんどがリプレイス前の設定ファイルをそのまま流用していたり、機密情報なので記載しておりません。
実行したコマンドのみを主に掲載しています。
:::

# 環境

**CentOS 7.9**

```sh
$ cat /etc/redhat-release
CentOS Linux release 7.9.2009 (Core)
```

**DBサーバの構成**

DBサーバは3台あり、1つがマスター、他2つがスレーブとして動作しており、3台すべてのリプレイスを行いました。

**DB切り替えの概要**

入れ替え元のDB3台をDB1(マスター),2,3とし、新たに用意したDB3台をDB4,5,6とします。

まずDB1のスレーブとしてDB4,5,6を追加し、マスター1台、スレーブ5台の構成にしました。
その後、サービスにメンテナンスを設け、DB4をマスター、DB5,6をスレーブとしてDBの切り替えを行いました。

# ファイアウォール設定（iptables）

他サーバからのアクセスを、iptablesを使用して通信を許可するルールを記載しました。

## インストール

```sh
# iptables-serviceインストール
yum install -y iptables-services

# 設定ファイル編集
vi /etc/sysconfig/iptables

# iptables起動
systemctl start iptables

# 自動起動するように設定
systemctl enable iptables

# 設定の確認
systemctl status iptables # Active になっていれば起動している

systemctl is-enabled iptables # enabled になっていれば自動起動が有効になっている
```

## ルールの設定

ルールの設定には以下の2つを検討しました。

1. ルールが記載されているファイルを直接編集する方法
1. すでに設定されているルールにコマンドで追加していく方法

今回はまだ何もルールが設定されていなかったため、**1.** のファイルを直接編集する方法を用いました。

すでにルールが設定されている場合、`-A INPUT -p tcp -m tcp -m state --state NEW -s XXX.XXX.XXX.XXX  --dport 22 -j ACCEPT`などとして追加していく方法を用いて、iptablesを再起動させずにルールを追加していくことが可能です。

## 参考文献

https://manual.sakura.ad.jp/vps/support/security/iptables.html

# 日本語設定

```sh
# 現状のlocaleを確認
localectl status

# localをja_JP.utf8へ変更
localectl set-locale LANG=ja_JP.utf8

# 現状のlocaleを確認
localectl status

# 設定の反映
source /etc/locale.conf
```

## 参考文献

https://zero-config.com/centos/changelocale-002.html

# 時刻設定

```sh
# 設定を記載
vi /etc/ntp.conf

# NTP再起動
systemctl restart ntpd

# 通信状態を確認（同期には時間がかかるので、再起動から数分後に実行）
ntpq -p
```

## 参考文献

https://lpi.or.jp/lpic_all/linux/network/network05.shtml

# ファイルシステムのマウント

```sh
# NFSサーバを利用するためのパッケージをインストール
yum install -y nfs-utils

# マウントの定義を記述
vi /etc/fstab

# fstabファイル反映
mount -a

# 反映確認
df -h
```

## 参考文献

https://www.infraeye.com/study/linuxz24.html
https://qiita.com/kihoair/items/03635447591358210772

# MySQL

元のサーバではMySQL5.6を使用していたため、新しいサーバでも同じバージョンをインストールしました。

## インストール

```sh
yum -y install http://dev.mysql.com/get/mysql57-community-release-el6-7.noarch.rpm
yum -y install yum-utils # 下記で yum-config-manager を使用するためにインストール

yum-config-manager --disable mysql57-community
yum-config-manager --enable mysql56-community

yum -y install mysql-community-server
```

## 起動

```sh
# バージョン確認
mysqld --version

# 設定ファイルの記述
vi /etc/my.cnf

# 起動
systemctl start mysqld.service

# 状態確認
systemctl status mysqld.service

# MySQLへログイン
mysql -u root
```

## MySQLユーザ作成

元のサーバでMySQLにログインし、ユーザ確認

```sql
-- ユーザ確認
mysql> SELECT user, host FROM mysql.user;
+-------------+-----------------------------+
| user        | host                        |
+-------------+-----------------------------+
| aaa         | xxx.xxx.xxx.xxx             |
〜 略

-- 権限確認
mysql> SHOW GRANTS FOR 'aaa'@'xxx.xxx.xxx.xxx';
+-----------------------------------------------------------------------------+
| Grants for aaa@xxx.xxx.xxx.xxx                                              |
+-----------------------------------------------------------------------------+
| GRANT USAGE ON *.* TO 'aaa'@'xxx.xxx.xxx.xxx' IDENTIFIED BY PASSWORD 'zzzzz'|
〜 略
```

上記のように、元サーバで確認したユーザを新たなサーバのMySQLで新規作成

```sql
mysql> CREATE USER 'aaa'@'xxx.xxx.xxx.xxx' IDENTIFIED BY PASSWORD 'zzzzz';
```

## スレーブ追加

元の構成

```
DB1 (マスター)
├── DB2 (スレーブ)
└── DB3 (スレーブ)
```

スレーブ追加後の構成

```
DB1 (マスター)
├── DB2 (スレーブ)
├── DB3 (スレーブ)
├── DB4 (スレーブ)
├── DB5 (スレーブ)
└── DB6 (スレーブ)
```

スレーブを追加する手順は、DB1,2,3のサーバを稼働させたまま行いました。
DB1に対して`mysqldump`コマンドを実行し、ダンプファイルを取得、DB4,5,6へダンプファイルを移行後、リストアを行いました。

DB1でダンプファイルを取得

```sh
mysqldump --databases テーブル名 --single-transaction --hex-blob --master-data=2 --flush-logs --add-drop-database -u ユーザ名 -pパスワード > dumpfile.sql
```

DB4,5,6でダンプファイルからデータをリストア

```sh
mysql -u ユーザ名 -pパスワード < dumpfile.sql
```

ダンプファイル内からDB1の、ダンプファイル取得時の**バイナリログ名**と**ポジション**を確認

```sh
grep "CHANGE MASTER TO" dumpfile.sql
```

MySQLへログイン後、以下を実行

```sql
mysql> CHANGE MASTER TO
MASTER_HOST='xxx.xxx.xxx.xxx',
MASTER_PORT=3306,
MASTER_USER='ユーザ名',
MASTER_PASSWORD='パスワード',
MASTER_LOG_FILE='バイナリログ名',
MASTER_LOG_POS=ポジション;

-- 上記で設定した値が反映されているか確認
SHOW SLAVE STATUS\G

-- スレーブを開始する
START SLAVE;
```

スレーブを追加後、DB1(マスター)にてスレーブの確認

```sql
SHOW SLAVE HOSTS\G
```

## 参考文献

https://www.tohoho-web.com/ex/mysql-mysqldump.html
https://weblabo.oscasierra.net/installing-mysql56-centos7-yum/

# DB切り替え

切り替え前の構成

```
DB1 (マスター)
├── DB2 (スレーブ)
├── DB3 (スレーブ)
├── DB4 (スレーブ)
├── DB5 (スレーブ)
└── DB6 (スレーブ)
```

上記の状態から、サービスにメンテナンスを入れ、DB切り替えを行いました。
切り替え後の構成

```
DB4 (マスター)
├── DB5 (スレーブ)
└── DB6 (スレーブ)
```

切り替えは以下のような流れで行いました。

1. DB1の書き込みをロック
2. DB2~6のスレーブを停止
3. DB4の設定ファイルをマスター用に書き換え
4. DB5,6をDB4のスレーブとして設定


**1. DB1の書き込みをロック**

```sql
-- プロセスがないことを確認する
mysql> SHOW PROCESSLIST;

-- テーブルロック
mysql> FLUSH TABLES WITH READ LOCK;
```

**2. DB2~6のスレーブを停止**

```sql
-- マスターバイナリログ読み取りを停止
mysql> STOP SLAVE IO_THREAD;

-- リレーログからイベントの読み取りが完了するまで待つ
-- 「Slave has read all relay log」が表示されるまで待つ 
mysql> SHOW SLAVE STATUS\G

-- スレーブ停止
mysql> STOP SLAVE;
```

**3. DB4の設定ファイルをマスター用に書き換え**

```sh
# MySQL停止
systemctl stop mysqld

# 設定ファイル書き換え（マスター用に編集）
vi /etc/my.cnf

# MySQL起動
systemctl start mysqld
```

**4. DB5,6をDB4のスレーブとして設定**

DB4（マスター）にて実行

```sql
-- レプリケーション系の設定を初期化
mysql> RESET SLAVE ALL;

mysql> RESET MASTER;

-- File、Position をメモしておく
mysql> SHOW MASTER STATUS\G
```

DB5,6（スレーブ）にて実行

```sql
-- レプリケーション系の設定を初期化
mysql> RESET SLAVE ALL;

mysql> RESET MASTER;

-- DB1をマスターとして設定
mysql> CHANGE MASTER TO
MASTER_HOST='xxx.xxx.xxx.xxx', -- DB4のIPアドレス
MASTER_PORT=3306,
MASTER_USER='ユーザ名',
MASTER_PASSWORD='パスワード',
MASTER_LOG_FILE='バイナリログ名', -- DB4で確認した File
MASTER_LOG_POS=ポジション; -- DB4で確認した Position

-- スレーブ開始
START SLAVE;
```

DB4（マスター）にて実行

```sql
-- スレーブが設定されているか確認
mysql> SHOW SLAVE HOSTS\G
```

## 参考文献

https://dev.mysql.com/doc/refman/5.6/ja/replication-administration-pausing.html
https://qiita.com/shotaTsuge/items/83e4387d6c8488677e17

# cron設定

rootユーザのcronを作成

```sh
# rootユーザにログイン
sudo su -

# cronを設定
crontab -e

# 設定されているcronを確認
crontab -l
```

## 参考文献

https://www.express.nec.co.jp/linux/distributions/knowledge/system/crond.html

# Cacti設定

サーバのリソース監視には**Cacti**を使用しました。

Cactiを使用するために、監視対象のサーバ（DB4,5,6）にSNMPエージェントを配置し、対象サーバを管理する管理サーバにSNMPマネージャを配置します。
SNMPマネージャはリプレイス前と変更がないため、管理サーバにて行った操作はありません。

DB4〜6にSNMPエージェントを配置

```sh
# パッケージインストール
yum -y install net-snmp net-snmp-utils

# 設定ファイル編集
vi /etc/snmp/snmpd.conf

# SNMP起動
systemctl start snmpd

# 自動起動するように設定
systemctl enable snmpd

# 設定の確認
systemctl status snmpd

systemctl is-enabled snmpd
```

## 参考文献

https://qiita.com/Kenji_TAJIMA/items/6bf9eb9d5d9ef4c8c3b6

# Nagios設定

サーバの監視ツール、アラート通知のためのソフトウェアとして**Nagios**を使用しました。
監視対象のサーバにNRPEを配置し、監視サーバから監視内容を設定します。

DB4〜6にNRPEを配置

```sh
# epelのインストール
yum install -y epel-release

# 使用するプラグインのインストール 例として1つだけ記載
yum install -y nagios-plugins

# NRPEインストール ... Nagios Remote Plugin Executor
yum install -y nrpe

# 設定ファイル編集
vi /etc/nagios/nrpe.cfg

# NRPE起動
systemctl start snmpd

# 自動起動するように設定
systemctl enable snmpd

# 設定の確認
systemctl status snmpd

systemctl is-enabled snmpd
```

監視サーバに設定ファイルを配置

```sh
vi /etc/nagios/servers/DB4.cfg

# チェックコマンドの実行
/usr/lib64/nagios/plugins/check_nrpe -H xxx.xxx.xxx.xxx -c check_load

# nrpe再起動
service nagios restart
```

## 参考文献

https://qiita.com/tukiyo3/items/9fbc65fc212606ab0846

# 最後に

設定ファイルに記載した詳細内容などは記載しておりませんが、以上がDBリプレイスのさいに行った大きな流れです。
**DB切り替え**の項目では、実際にはメンテナンスを行い、サービスを停止していました。手順書は綿密に作成していましたが、想定していたより時間がかかる項目などがあったため、模擬試験を行える環境を用意して事前にリハーサルを行うことが重要だなと感じました。
