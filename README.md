# svn_database-disk-image-is-malformed
Tortoise SVN 異常處理
Tortoise SVN 出現類似如下問題的兩種解決方案。
svn: E200030: database disk image is malformed
一、最簡單的方法，複製其它人的.svn/wc.db替換。
二、類Unix系統（如Mac OS X），會自帶sqlite3，windows系統要先下載sqlite3。
操作之前，切記備份.svn/wc.db
在命令列終端cd到.svn資料夾，
cd 項目資料夾/.svn
sqlite3 wc.db "pragma integrity_check"
sqlite3 wc.db "reindex nodes"
sqlite3 wc.db "reindex pristine"

至此，問題一般解決了。還不行，可以使用以下命令排查問題
sqlite3 wc.db "select sql from sqlite_master where name='NODES'"
sqlite3 wc.db "select sql from sqlite_master where name='I_NODES_PARENT'"
建立表NODES_COPY，作為表NODES的複本，sql語句：
sqlite3
 wc.db "CREATE TABLE NODES_COPY ( wc_id INTEGER NOT NULL REFERENCES 
WCROOT (id), local_relpath TEXT NOT NULL, op_depth INTEGER NOT NULL, 
parent_relpath TEXT, repos_id INTEGER REFERENCES REPOSITORY (id), 
repos_path TEXT, revision INTEGER, presence TEXT NOT NULL, moved_here 
INTEGER, moved_to TEXT, kind TEXT NOT NULL, properties BLOB, depth TEXT,
 checksum TEXT REFERENCES PRISTINE (checksum), symlink_target TEXT, 
changed_revision INTEGER, changed_date INTEGER, changed_author TEXT, 
translated_size INTEGER, last_mod_time INTEGER, dav_cache BLOB, 
file_external TEXT, PRIMARY KEY (wc_id, local_relpath, op_depth) )"
複製表NODES的內容到表NODES_COPY，sql語句：
sqlite3 wc.db "insert into NODES_COPY select * from NODES"
刪除NODES，sql語句：
sqlite3 wc.db "drop table NODES"
重新建立表NODES，sql語句：

sqlite3 wc.db "CREATE TABLE NODES ( 
wc_id INTEGER NOT NULL REFERENCES WCROOT (id), local_relpath TEXT NOT 
NULL, op_depth INTEGER NOT NULL, parent_relpath TEXT, repos_id INTEGER 
REFERENCES REPOSITORY (id), repos_path TEXT, revision INTEGER, presence 
TEXT NOT NULL, moved_here INTEGER, moved_to TEXT, kind TEXT NOT NULL, 
properties BLOB, depth TEXT, checksum TEXT REFERENCES PRISTINE 
(checksum), symlink_target TEXT, changed_revision INTEGER, changed_date 
INTEGER, changed_author TEXT, translated_size INTEGER, last_mod_time 
INTEGER, dav_cache BLOB, file_external TEXT, PRIMARY KEY (wc_id, 
local_relpath, op_depth) )"
sqlite3 wc.db "create index I_NODES_PARENT on NODES (wc_id, parent_relpath, op_depth)"
Copy NODES_COPY into NODES:

sqlite3 wc.db "insert into NODES select * from NODES_COPY"

Drop table NODES_COPY:

sqlite3 wc.db "drop table NODES_COPY"

Then you need to do something similar for PRISTINE, although this time
there is no extra index:

sqlite3 wc.db "select sql from sqlite_master where name='PRISTINE'"

Create PRISTINE_COPY
Copy PRISTINE into PRISTINE_COPY
Drop and recreate PRISTINE
Copy PRISTINE_COPY into PRISTINE
Drop PRISTINE_COPY
