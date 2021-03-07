# 導入手順
## インストール
### マスタ構築
#### postgresql.conf修正
#### pg_hba.conf修正

### スレーブ構築
#### マスタの複製
#### postgresql.conf修正
#### pg_hba.conf修正


* WALログファイル(物理ファイル)への書き込みタイミング

|内容|備考|
|-|-|
|実行中のトランザクションがcommitされたとき|wal buffer※1<br>→walファイルへ書き込まれる|
|WALバッファがあふれたとき||
|checkpoint、vacuum時||
|wal writerが作動した時|wal_writer_delayで設定<br>デフォルト200ミリ秒|
|更新系SQL実行時<br>検索系SQL(HOTによる不要なタプルの削除と並べ替え発生時)||
※1:wal bufferには更新データを保存。データ更新は、共有バッファとWALバッファに対して行われる。


* VACUUM処理の流れ  
・SharedUpdateExecuteLockをかける  
  →VACUUM処理中もテーブルの読書可  
・VisibilityMapをもとに不可視タプルを含むブロックのみ処理を行う  
SQL:vacuum {テーブル}  
  ※｛テーブル｝をつけないとシステムカタログを含むデータベース全体がVACUUMされる  
OS:vacuumdb {DB}

|順番|内容|備考|
|-|-|-|
|1|先頭からテーブルを読み込む。<br>VMをチェックし、不要行を含むページがあれば、ゴミタプルとして記録<br>※対象ページの全行をチェック<br>※meintenance_work_memを超えたら次のフェーズへ|Scanフェーズ|
|2|全ページを走査後、1の記録した不要行に対して、インデックスを削除|vacuumフェーズ|
|3|不要なタプルを削除|vacuumフェーズ|
|4|有効なタプルを並び替え(ブロック内のみ)|cleanupフェーズ|
|5|FSM更新|cleanupフェーズ|
|6|pg_classのrelpages、reltuplesを更新|cleanupフェーズ|
|7|CLOGを更新|cleanupフェーズ|

* VACUUM FULLの流れ  
・処理の流れはVACUUMと同様。ただ、大きな違いとして下記点が挙げられる。  
・AccessExclusiveLockをかける  
  →処理中の読書不可  
・ブロックをまたいで有効なタプルを並び変える  
・空のファイルを作成し、そのファイルにコピーしていく(容量が2倍になる)  
・ブロックを切り詰める

* クラッシュリカバリの流れ

|順序|内容|備考|
|-|-|-|
|1|pg_controlからチェックポイント位置を確認||
|2|WALから前回チェックポイント後に発生したトランザクション情報を読み込み||
|3|チェックポイント後は、最初の更新でブロック全体がWALに書き込まれている。<br>よって、ブロック全体をリカバリ||
|4|WALから更新トランザクションを適用||
|5|最新のWALまで更新トランザクションを適用||


* アーカイブ処理のタイミング

|内容|備考|
|-|-|
|60秒間隔||
|SIGUSR1シグナルを受信したとき||
|pg_log/archive_statusを検索し、.readyファイルを発見<br>archive_commandを実行し、.ready→.doneに変更||
|WALの切り替え時ではなく、メモリの更新がディスクに反映されて不要になった時||


* チェックポイント実施のタイミング

|内容|備考|
|-|-|
|checkpoint実行時||
|checkpoint_timeout間隔||
|max_wal_sizeに到達時||
|オンラインBK開始時<br>pg_start_backup<br>pg_stop_backup||
|インスタンス停止時||
|インスタンス構成時||
|pg_ctl stop -m immediateコマンド以外|強制停止(ロールバックされる)|
|create database、drop database時||

* WALログ(pg_wal)の切り替えタイミング

|内容|備考|
|-|-|
|WALログを使い切った時||
|pg_switch_xlog()を実行時||
|アーカイブモード+archive_timeoutを過ぎたとき||

* WALログ(pg_wal)への書き込みの流れ

|順番|内容|備考|
|-|-|-|
|1.|Update処理を実行||
|2.|CLOG更新、XIDをIN_PROGRESSに設定||
|3.|共有バッファにタプルを追加後、XLogInsert()でWALバッファへ更新データを書き込む||
|4.|commitに関するWALログをWALバッファへ書き込む||
|5.|WALログファイルに同期書き込み||
|6.|CLOGの更新、XIDをCOMMITEDに変更。VACUUMが削除するまで残る||

前回のチェックポイント後に、初めてタプルが追加されたブロックは、ブロック全体をWALへ書き込む。  
同じブロックへのタプルの追加は差分のみWALへ書き込む

* FILFACTOR
ブロックの使用率が指定の値以上になった場合、不要なタプルを削除して有効なタプルを並び替える。  
空き領域を確保しておくことで、データがブロックにまたがることを防ぐ。

* Index Only Scan
selectの検索がインデックスのKeyのみの場合、テーブルへのアクセスを省略する。
VisibilityMapを見て、可視のブロックはIndexOnlyScan、それ以外はテーブルまでアクセスを実施する。


* レプリケーションスロット
masterがslaveに必要なWALを判別して、自動削除しないように操作する仕組み。レプリケーション方法と合わせて、下記に設定を示す。  

#### レプリケーション【非同期】

##### <font style color=blue>master側</font>

${PGDATA}/postgresql.conf  

|設定|内容|
|-|-|
|max_wal_senders=2|1以上|
|wal_level=replica|V9.5以前のarchive、hot_standbyに相当|
|archive_mode=on|スタンバイDBの数⁺1|
|max_replication_slots=1|スタンバイDBの数|
|archive_command='test ! -f /pg_archive/%f && /bin/cp %p /pg_archive/%f' ※設定例|WALをアーカイブ領域にコピーするコマンド|
|synchronous_commit=※1|同期レベル|
|synchronous_standby_names=設定なし|この設定で同期か非同期に分かれる|

* レプリケーションスロット設定
`select * from pg_create_physical_replication_slot('repl_slot')`

##### <font style color=green>slave側</font>

${PGDATA}/postgresql.conf  

|設定|内容|備考|
|-|-|-|
|hot_standby=on|スレーブで検索系SQLを受け付けるか|

${PGDATA}/recovery.conf  

|設定|内容|備考|
|-|-|-|
|standby_mode=on||
|primary_conninfo='host=マスターのホスト名 port=5432 application_name=任意の名前' ※設定例|プライマリへの接続情報<br>application_nameはpg_stat_activityビューやログファイル上で該当接続を識別可能にする|
|primary_slot_name='repl_slot'|マスターのレプリケーションスロット名|
|recovery_target_timeline=latest||
|restore_command=’cp /pg_archive/%f %p’ ※設定例|アーカイブをpg_walに戻すコマンド|

#### レプリケーション【同期】

##### <font style color=blue>master側</font>

${PGDATA}/postgresql.conf  

|設定|内容|
|-|-|
|max_wal_senders=2|1以上|
|wal_level=replica|V9.5以前のarchive、hot_standbyに相当|
|archive_mode=on|スタンバイDBの数⁺1|
|max_replication_slots=1|スタンバイDBの数|
|archive_command='test ! -f /pg_archive/%f && /bin/cp %p /pg_archive/%f' ※設定例|WALをアーカイブ領域にコピーするコマンド|
|synchronous_commit=※1|同期レベル|
|synchronous_standby_names='slave1'|同期するスタンバイ名|
|host_standby_feedback=on|自身の情報をマスターに送信|

* レプリケーションスロット設定
`select * from pg_create_physical_replication_slot('repl_slot')`

##### <font style color=green>slave側</font>

${PGDATA}/postgresql.conf  

|設定|内容|備考|
|-|-|-|
|hot_standby=on|スレーブで検索系SQLを受け付けるか|

${PGDATA}/recovery.conf  

|設定|内容|備考|
|-|-|-|
|standby_mode=on||
|primary_conninfo='host=マスターのホスト名 port=5432 application_name=任意の名前' ※設定例|プライマリへの接続情報<br>application_nameはpg_stat_activityビューやログファイル上で該当接続を識別可能にする|
|primary_slot_name='repl_slot'|マスターのレプリケーションスロット名|
|recovery_target_timeline=latest||
|restore_command=’cp /pg_archive/%f %p’ ※設定例|アーカイブをpg_walに戻すコマンド|

※1:非同期/同期に関連するパラメータ

<table>
<td></td><td colspan="2" align=center><b>synchronous_standby_names</b></td>
</tr>
<tr>
<td><b>synchronous_commit</b></td>
<td><b>設定なし</b></td>
<td><b>設定あり</b></td>
</tr>
<tr>
<td>off</td>
<td colspan="2">プライマリのWALも非同期で書き込む</td>
</tr>
<tr>
<td>local</td>
<td colspan="2">プライマリのWALは同期書き込み、スタンバイは非同期</td>
</tr>
<tr>
<td>on</td>
<td>プライマリのWALのみ同期書き込み</td>
<td>スタンバイのWALを同期で書き込むのをプライマリは待つ</td>
</tr>
</table>


#### 統計情報
||アクセス統計情報|テーブル(カラム)統計情報|備考|
|-|-|-|-|
|用途|autovacuum workerで利用<br>データベースの動作状況を格納|プランナがコスト計算で利用||
|収集方法|stats_collectorプロセス<br>パ:track_activities=on<br>パ:track_counts=on|ANALYZEコマンド ※1<br>VACUUM ALALYZE ※2<br>vacuumdb -z {テーブル名}<br>vacuumdb -avz<br>パ:autovacuum=on ※3<br>パ:track_counts=on|※1:サンプリングによる。負荷はレコード数に関係なく軽量。<br>※2:全件走査のため制度が高い|
|収集タイミング|・各サーバから待機直前に送信される<br>・500msに1回反映<br>・track_activitiesにはリアルタイム反映<br>・pg_stat_テーブルにはトランザクション外で参照|プランナがコスト計算で利用|autovacuum実行時、vacuumコマンド実行時|
|格納テーブル|主要なビュー<br>pg_stat_database<br>pg_stat_activity<br>pg_stat_bgwriter<br>pg_statio_user_tables<br>pg_statio_user_indexes|pg_class(テーブル統計情報)※4<br>pg_statistic(カラム統計情報)※5<br>ともにシステムカタログ|※4:relpages,reltaplesに格納※5:pg_statisticはpg_statsビューで参照|

※3:<br>閾値:autovacuum_analyze_threshold(50)＋autovacuum_analyze_scale_factor(0.1)×レコード数<br>
サンプリング数:default_statistics_target(100)×300行<br>カッコはデフォルト値<br>
例えば、1000行のテーブルに対しては、(50+0.1×1000)の150回の追加、削除、更新があればサンプリング値に従って、統計情報を取得する。

#### REINDEX

〇:ロックする
×:ロックなし
||テーブル書き込み|テーブル読み込み|インデックス読み込み|
|-|-|-|-|
|REINDEX|〇|×|〇|
|CREATE INDEX|〇|×|-|
|CREATE INDEX|〇|〇|-|

* `REINDEX index index名;`
→インデックス再構築
* `REINDEX table table名;`
→テーブルに貼られた全インデックス再構築
* `REINDEX database database名;`
→データベースに存在する全インデックス再構築

##### システムテーブル(pg_xxx)のインデックスの破損対応

|順序|対応|コマンド|
|-|-|-|
|1|DB停止|`$ pg_ctl stop`|
|2|シングルユーザーモード<br>or<br>postgresql.confのignore_system_indexes=on|`# postgres --single -O -P -D $PGDATA`|
|3|REINDEX実行|`REINDEX index pg_xxxx_index;`<br>Ctrl+Dで終了|
|4|DB再起動|`$ pg_ctl restart`|

※インデックス破損は、実際にインデックスを使用する状況にならないと気づかない可能性あり
