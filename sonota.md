# Windowsにdocker環境を構築する

#### 1.Hyper-VとContainerの有効化
|No|手順|
|-|-|
|1|Windowsメニューからコントロールパネルを開いて、「プログラム」を選択|
|2|「Windowsの機能の有効化または無効化」を選択|
|3|「Hyper-V」と「Container」にチェックを入れて「OK」|

#### 2.BIOSでVirtualization Technologyを有効化
|No|手順|
|-|-|
|1|Intel Virtualization Technologyを「Enabled」に変更|

#### 3.エラー回避 ※エラーが出力された場合
|No|手順|
|-|-|
|1|Windowsセキュリティを起動|
|2|アプリとブラウザのコントロールをクリック|
|3|一番下にあるExploit protectionの設定をクリック|
|4|プログラム設定のタブを選択|
|5|C:\WINDOWS\System32\vmcompute.exeを選んで編集|
|6|制御フローガード(CFG)のシステム設定の上書きのチェックを外して適用|

# dockerとdocker composeのセットアップ

手順は下記のとおりである。  
```
# curl -fsSL https://get.docker.com -o get-docker.sh
# sh ./get-docker.sh
# systemctl enable docker
# systemctl start docker
# yum -y install docker-compose
```

# dockerでPostgreSQL導入

|No|内容|手順|
|-|-|-|
|1|PostgreSQLを起動|`$ docker run -d --rm --name postgres -e POSTGRES_PASSWORD=postgres00 -e POSTGRES_INITDB_ARGS="--encoding=UTF-8 --locale=C" -p 5432:5432 -v ~/data:/var/lib/postgresql/data postgres`<br>※rmを付与しているので、停止時にコンテナ削除<br>※データを永続化するには、下記のコマンドを実行する。<br>`$ docker run -p 5432:5432 -d -it --name pg -e POSTGRES_PASSWORD=パスワード -v ~/data:/var/lib/postgresql/data postgres`<br>※~/dataは、ホームディレクトリの直下にdataディレクトリを予め作成しておく。<br>/var/lib/postgresql/dataの部分は、PostgreSQLのデフォルトのデータの保存場所である。|
|2|起動確認|`$ docker container ls`|
|3|コンテナへログイン|`$ docker exec -it postgres bash`|
|4|データベースへログイン|`$ psql -U postgres -d template1`|

### ファイルの編集

|No|内容|手順|
|-|-|-|
|1|ホスト側にコンテナ内のファイルをコピー|`$ docker container cp コンテナ名:/var/lib/postgresql/data/postgresql.conf /home/docker/.`|
|2|ホスト側からコンテナにコピー|`$ docker container cp /home/docker/postgresql.conf コンテナ名:/var/lib/postgresql/data/postgresql.conf`|
|3|コンテナを再起動|`$ docker restart コンテナ名`|

# Dockerにzabbixを構築

docker-composeを利用して構築する手順は下記の通り。  
```
# mkdir docker
# cd docker/
# git clone https://github.com/zabbix/zabbix-docker.git
# cd zabbix-docker/
# cp -p docker-compose_v3_centos_pgsql_latest.yaml docker-compose.yaml ※DBにPostgreSQLを利用する場合
あるいは
# cp docker-compose_v3_centos_mysql_latest.yaml docker-compose.yaml ※1:DBにMySQLを利用する場合
# docker-compose pull
# docker images
# docker-compose up -d
```

※1:YAMLファイルで、MySQLのユーザやパスワードの設定ファイルは.MYSQL_USER、.MYSQL_PASSWORD、.MYSQL_ROOT_PASSWORDである。必要に応じて各ファイルを編集する。

ブラウザから下記を入力する。  
https://Zabbixを構築したサーバのIP/  
Username： Admin  
Password： zabbix  

ログイン直後から、Zabbix Server が「利用不可」となっている。これを回避するには、以下の手順で設定を変更します。監視先を 127.0.0.1 から zabbix-agent に変更する。  

|No|手順|
|-|-|
|1|左メニューの「設定」→「ホスト」をクリックし、「Zabbix server」の名前をクリック。|
|2|それから「インターフェース」の「IPアドレス」が 「127.0.0.1」を消す。|
|3|「DNS名」に zabbix-agent を入力し、接続方法の「DNS」をクリックし 、「更新」をクリックする。|

# LVM構成

<table>
<tr align=center>
<td>MT</td>
<td>/boot</td>
<td>/</td>
<td>/var</td>
<td>/backup(追加ディスク)</td>
<td rowspan="2">-</td>
</tr>
<tr align=center>
<td>FS</td>
<td>xfs</td>
<td>xfs</td>
<td>xfs</td>
<td>xfs</td>
</tr>
<tr align=center>
<td>LV</td>
<td>-</td>
<td>root</td>
<td>var</td>
<td>backup</td>
<td rowspan="3">LVM</td>
</tr>
<tr align=center>
<td>VG</td>
<td>-</td>
<td colspan="2">rhel</td>
<td>rhel2</td>
</tr>
<tr align=center>
<td>PV</td>
<td>-</td>
<td colspan="2" >PV(/dev/sda2)</td>
<td>PV(/dev/sdb1)</td>
</tr>
<tr align=center>
<td>パーティション</td>
<td>/dev/sda1</td>
<td colspan="2">/dev/sda2</td>
<td>/dev/sdb1</td>
<td  rowspan="3">-</td>
</tr>
<tr align=center>
<td>デバイス</td>
<td colspan="3">/dev/sda</td>
<td>/dev/sdb</td>
</tr>
<tr align=center>
<td>ディスク</td>
<td colspan="3">HDD1</td>
<td>HDD2</td>
</tr>
</table>

### LVM拡張手順

新たにディスクを切り出し、既存のボリューム(rhel)に追加する手順  
ここでは、/varを拡張する

|NO|内容|手順|
|-|-|-|
|1|パーティション作成|`fdisk /dev/sdg`<br>1:n - 新規作成<br>2:p - 基本パーティション<br>3:1 - パーティション番号<br>4:2048 - ファイルセクター開始位置 何も入力せずエンターで可<br>5:31457279 - ファイルセクター終了位置(例は15GBの場合)何も入力せずエンターで可<br>6:t - パーティションのシステムid変更<br>7:1 - パーティションの選択<br>8:L - リストでシステムid一覧確認<br>9:8e - (Linux LVM)<br>10:p - パーティションの確認<br>11:w - 書き込み|
|2|物理ボリューム作成|`pvcreate /dev/sdg1`<br>`pvscan`|
|3|拡張したいLVが含まれるVGに2を追加|`vgextend rhel /dev/sdg1`<br>`vgscan`|
|4|LVの拡張|`lvextend -l +100%FREE /dev/rhel/var`<br>`lvscan`|
|5|ファイルシステム拡張|`reseize2fs /dev/rhel/var`<br>or<br>`xfs_growfs /dev/rhel/var`|

# zabbix容量見積

## 1秒あたりに受信する平均個数

例)350アイテムが登録されている前提で計算  
350アイテム÷60秒間隔=6個  
→毎秒6個の新しいデータがZabbixへ追加される

## ヒストリデータ

2年間で保存されるデータ個数  
6個×(2年×365日×24H×3600秒)=378,432,000個  
→約3.8億個  

2年間保存するのに必要なディスク容量  
3.8億個×50バイト=19GB  
※50バイト:1つの値の保存容量(概算値)

## トレンドデータ

1時間ごとに平均値の1レコードを保存  

2年間保存するのに必要なディスク容量  
350アイテム×2年×365日×24H×128バイト=784,896,000バイト  
→800MB  
※128バイト:1つのトレンドデータの保存容量(概算値)

## イベントデータ

1秒に1個のイベントが発生すると仮定する  

2年間保存するのに必要なディスク容量
2年×365日×24H×3600秒×130バイト=8,199,360,000
→8GB

## 各種設定に必要なサイズ

約10MB前後  

## 2年間保存するのに必要な容量

19GB+800MB+8GB+10MB=約30GBが必要


# anacron

anacronはcronが1時間に1回実行している。  
<br>
* /etc/cron.d/0hourly  
<br>
1.cronが/etc/cron.d/0hourlyを1時間ごとに起動
`01 * * * * root run-parts /etc/cron.hourly >> /var/log/cron-execution 2>&1`  
2./etc/cron.hourly/0anacronを実行→/usr/sbin/anacron -sを実行(非常駐デーモン)  
3./etc/anacrontabのスケジュールに従い、/etc/cron.daily/XX、/etc/cron.weekly/XX、/etc/cron.monthly/XXを実行する  
4.正常に実行したら/var/spool/anacron/{cron.montyly、cron.weekly、cron.daily}に日付を書き込む  

※anacronは実行時にメール配信する機能がある  
このメールにはcronの標準出力、エラー出力、つまり実行結果を通知する  
/var/log/cronは実行ログは出力するが、実行結果については記録されない  
停止するには/etc/sysconfig/crondにCRONDARGS="-m off"を追加する

# 鍵認証によるSSH
```
サーバAAA：接続先サーバ
踏み台BBB：接続元サーバ

1.USER1を作成(サーバAAAでの作業)
# useradd user1
# ls /home

2.キーペアの作成(サーバAAAでの作業)
# su - user1
$ ssh-keygen
下記2ファイルが作成
→/home/user1/.ssh/id_rsa.pub(公開鍵)
→/home/user1/.ssh/id_rsa(秘密鍵)

3.秘密鍵をダウンロードしておく(サーバAAAでの作業)
メモ帳とかにコピペでもOK
その後、秘密鍵は削除しておく
下記1ファイルになる
→/home/user1.ssh/id_rsa.pub

4.公開鍵の名前を変更する(サーバAAAでの作業)
$ ls
id_rsa.pub
$ mv id_rsa.pub authoroized_keys
$ ls
authorized_keys

5.公開鍵のパーミッションを変更(サーバAAAでの作業)
$ chmod 600 authorized_keys

6.「.ssh」ディレクトリのパーミッションの確認(サーバAAAでの作業)
$ ls -la
700であること
下記1ファイルである
→/home/user1.ssh/authorized_keys

7.接続確認(踏み台BBBの「任意のユーザ」での作業) 秘密鍵を「任意のユーザ」のホームディレクトリ配下に置いておく
$ ssh -i id_rsa user1@XXX.XXX.XX.XX
※id_rsaはサーバAAAのuser1の秘密鍵
踏み台BBBの任意のユーザからサーバAAAのuser1へSSHログインする
```

# dockerとdocker composeのセットアップ
```
curl -fsSL https://get.docker.com -o get-docker.sh
sh ./get-docker.sh
systemctl enable dockern
systemctl start docker
yum -y install docker-compose
```

# Dockerにzabbixを立てる
```
# mkdir docker
# cd docker/
# git clone https://github.com/zabbix/zabbix-docker.git
# cd zabbix-docker/
# cp -p docker-compose_v3_centos_pgsql_latest.yaml docker-compose.yaml
あるいは
# cp docker-compose_v3_centos_mysql_latest.yaml docker-compose.yaml
# docker-compose pull
# docker images
# docker-compose up -d
Username： Admin
Password： zabbix
```
※YAML ファイル中、MySQL のユーザやパスワードの設定ファイルは .MYSQL_USER 、 .MYSQL_PASSWORD 、 .MYSQL_ROOT_PASSWORD 。必要に応じて編集。  
ログイン直後は、Zabbix Server が「利用不可」の障害となっている。デフォルトのままでは、Zabbix Server の状態を Zabbix agent経由で取得できないため、以下の手順で設定を変更する。  
監視先を 127.0.0.1 から zabbix-agent に変更。

```
① docker上にmysqlコンテナを立てる
# docker run --name mysql-server -t -e MYSQL_DATABASE="zabbix" -e MYSQL_USER="zabbix" -e MYSQL_PASSWORD="password" -e MYSQL_ROOT_PASSWORD="password" -d mysql:5.7 --character-set-server=utf8 --collation-server=utf8_bin

② docker上にjava-gatewayコンテナを立てる
# docker run --name zabbix-java-gateway -t -d zabbix/zabbix-java-gateway:latest

③ zabbix-server-mysqlコンテナを立てる
# docker run --name zabbix-server-mysql -t \
-e DB_SERVER_HOST="mysql-server" \
-e MYSQL_DATABASE="zabbix" \
-e MYSQL_USER="zabbix" \
-e MYSQL_PASSWORD="password" \
-e MYSQL_ROOT_PASSWORD="password" \
-e ZBX_JAVAGATEWAY="zabbix-java-gateway" \
--link mysql-server:mysql \
--link zabbix-java-gateway:zabbix-java-gateway \
-p 10051:10051 \
-d zabbix/zabbix-server-mysql:latest

④ zabbix-web-nginx-mysqlコンテナを立てる
# docker run --name zabbix-web-nginx-mysql -t \
-e DB_SERVER_HOST="mysql-server" \
-e MYSQL_DATABASE="zabbix" \
-e MYSQL_USER="zabbix" \
-e MYSQL_PASSWORD="passowrd" \
-e MYSQL_ROOT_PASSWORD="password" \
--link mysql-server:mysql \
--link zabbix-server-mysql:zabbix-server \
-p 80:80 \
-d zabbix/zabbix-web-nginx-mysql:latest
```

# Windows docker環境の構築

①Hyper-VとContainerの有効化  
1.Windowsメニューからコントロールパネルを開いて、「プログラム」を選択  
2.「Windowsの機能の有効化または無効化」を選択  
3.「Hyper-V」と「Container」にチェックを入れて「OK」  
<br>
②BIOSでVirtualization Technologyを有効化  
Intel Virtualization Technologyを「Enabled」に変更  
<br>
③エラー回避  
1.Windowsセキュリティを起動  
2.アプリとブラウザのコントロールをクリック  
3.一番下にあるExploit protectionの設定をクリック  
4.プログラム設定のタブを選択  
5.C:\WINDOWS\System32\vmcompute.exeを選んで編集  
6.制御フローガード(CFG)のシステム設定の上書きのチェックを外して適用  