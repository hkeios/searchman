* WALログファイルへの書き込みタイミング

|内容|備考|
|-|-|
|実行中のトランザクションがcommitされたとき|wal buffer※<br>→walファイルへ書き込まれる|
|WALバッファがあふれたとき||
|checkpoint、vacuum時||
|wal writerが作動した時||
|更新系SQL、検索系SQL(HOTによる不要なタプルの削除と並べ替え発生時)||
※wal bufferには更新データを保存。データ更新は、共有バッファとWALバッファに対して行われる。


* VACUUM処理の流れ  
・SharedUpdateExecuteLockをかける  
　→VACUUM処理中もテーブルの読書可  
・VisibilityMapをもとに不可視タプルを含むブロックのみ処理を行う  
SQL:vacuum {テーブル}  
　※｛テーブル｝をつけないとシステムカタログを含むデータベース全体がVACUUMされる  
OS:vacuumdb {DB}

|順番|内容|備考|
|-|-|-|
|1|インデックスがあれば削除||
|2|不要なタプルを削除||
|3|有効なタプルを並び替え|ブロック内のみ|
|4|FSM更新||
|5|pg_classのrelpages、reltuplesを更新||
|6|CLOGを更新||


* VACUUM FULLの流れ  
・処理の流れはVACUUMと同様。ただ、大きな違いとして下記点が挙げられる。  
・AccessExclusiveLockをかける  
　→処理中の読書不可  
・ブロックをまたいで有効なタプルを並び変える  
・空のファイルを作成し、そのファイルにコピーしていく(容量が2倍になる)  
・ブロックを切り詰める
