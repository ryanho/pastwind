# PostgreSQL 8.4/Pgpool-II 3.3設定 #
大部份提到pgpool的文章，雖然有提到replication的設置，但關於online recovery這個功能都很少提，這篇文件主要是補完這個部份。

[CC-BY-SA][1] @RyanHo

## 環境設定 ##
- Node1: 192.168.2.231
- Node2: 192.168.2.232
- Node3: 192.168.2.233
- File1: 192.168.2.234
- Node1~3安裝上PostgreSQL
- 作業系統為CentOS 6.4
- pgpool安裝在node1
- File1作為NFS Server，讓Node1~3存取WAL archiving

理論上可以用更簡單的做法，因為每一台DB互相有做SSH Trust，可以把WAL archive放在本機其他目錄，然後用scp複製。

Pgpool會使用額外的系統資源，如果可能的話，在獨立的設備上安裝。Pgpool-II要到3.2版才支援HA，在之前的版本執行多個實體會造成意想不到的問題。

以下設定以全新安裝為前提。

## 設定主機名稱對應 ##
在Node1~3與File1編輯/etc/hosts，加入

	192.168.2.231	node1
	192.168.2.232	node2
	192.168.2.233	node3
	192.168.2.234	file1

如果沒有設定主機名稱對應，後面的WAL archiving與Online Recovery會失敗。

## 掛載 WAL archiving folder ##
在File1安裝NFS Server

	yum install nfs-utils -y
	chkconfig rpcbind on
	chkconfig nfs on
	service rpcbind start
	service nfs start
	mkdir -p /exchange/wal/node1
	mkdir -p /exchange/wal/node2
	mkdir -p /exchange/wal/node3
	chmod -R 777 /exchange

編輯/etc/exports

	/exchange	192.168.2.0/24(rw)

在Node1~3安裝NFS支援

	yum install nfs-utils -y
	chkconfig rpcbind on
	service rpcbind start

掛載目錄

	mkdir /exchange
	mkdir /exchange/node1
	mount -t nfs file1:/exchange /exchange/

設定自動載入，編輯/etc/rc.local，加入

	mount -t nfs file1:/exchange /exchange/

## 安裝PostgreSQL與Pgpool-II ##
在node1~3執行

	yum install postgresql-server postgresql-contrib postgresql-devel gcc make -y

	cd /usr/local/src
	wget http://www.pgpool.net/download.php?f=pgpool-II-3.3.0.tar.gz
	tar zxvf pgpool-II-3.3.0.tar.gz
	cd pgpool-II-3.3.0
	./configure
	make; make install

## 設定PostgreSQL ##
### 生成初始資料庫 ###
在node1執行

	/etc/init.d/postgresql initdb

### 編輯postgresql.conf ###
    listen_addresses = '*'
    #設定WAL archiving for Point-In-Time Recovery
    archive_mode = on
    archive_command = 'cp %p /exchange/wal/`hostname -s`/%f'
    archive_timeout = 300

### 編輯pg_hba.conf ###
	host    all     all     192.168.2.0/24  trust

### 啟動PostgreSQL ###
	/etc/init.d/postgresql start

### 複製資料庫到其他Node ###
	su - postgres
	psql
	select pg_start_backup('initial_backup');
	\q
	rsync -avz /var/lib/pgsql/data/* node2:/var/lib/pgsql/data/
	psql
	select pg_stop_backup();
	\q

完成後，到其他Node啟動PostgreSQL。
    
## 設定Pgpool-II ##
### 修改啟動指令稿 ###
複製啟動指令稿，並修改

	/usr/local/src/pgpool-II-3.3.0/redhat
	cp pgpool.init /etc/init.d/pgpool
	cp pgpool.sysconfig /etc/sysconfig/pgpool
	chkconfig --add pgpool
	chkconfig pgpool on

	vim /etc/init.d/pgpool
	
	PGPOOLENGINE=/usr/local/bin
	PGPOOLCONF=/usr/local/etc/pgpool.conf

然後把 killproc /usr/bin/pgpool 改成 killproc $PGPOOLDAEMON。

### 建立設定檔 ###
	cd /usr/local/etc
	cp pcp.conf.sample pcp.conf
	cp pgpool.conf.sample-replication pgpool.conf
	cp pool_hba.conf.sample pool_hba.cof
	touch pool_passwd
	chmod 666 pool_passwd

### 編輯pgpool.conf ###
	listen_addresses = '*'

	backend_hostname0 = 'node1'
	backend_port0 = 5432
	backend_weight0 = 1
	backend_data_directory0 = '/var/lib/pgsql/data'
	backend_flag0 = 'ALLOW_TO_FAILOVER'
	backend_hostname1 = 'node2'
	backend_port1 = 5432
	backend_weight1 = 1
	backend_data_directory1 = '/var/lib/pgsql/data'
	backend_flag1 = 'ALLOW_TO_FAILOVER'
	backend_hostname2 = 'node3'
	backend_port2 = 5432
	backend_weight2 = 1
	backend_data_directory2 = '/var/lib/pgsql/data'
	backend_flag2 = 'ALLOW_TO_FAILOVER'

	#除錯模式，會詳細紀錄運作狀態到/var/log/pgpool.log，正式運行時記得關掉。
	debug_level = 1

	health_check_period = 30
	health_check_user = 'postgres'

	recovery_user = 'postgres'
	

node1預設是master，如果master node掛了，會改由node2作為master，依此類推。node1做完online recovery之後，會提升為master。

### 編輯pcp.conf ###
產生pcp控制用密碼，產生後加上帳號名稱，格式為 usename:password，寫入到pcp.conf。

	pg_md5 -p

### 啟動pgpool ###
在Node1(192.168.2.231)啟動pgpool

	/etc/init.d/pgpool start

### 檢視Node狀態 ###

	sudo -u postgres psql -p9999
	show pool_nodes;

應該會顯示如下訊息，如果status此欄的數值不是2，代表連不上或有其他問題。

     node_id | hostname | port | status | lb_weight |  role  
    ---------+----------+------+--------+-----------+--------
     0       | node1    | 5432 | 2      | 0.500000  | master
     1       | node2    | 5432 | 2      | 0.500000  | slave
     2       | node3    | 5432 | 2      | 0.500000  | slave
    (3 筆資料列)

### 測試replication ###
建立測試資料庫

	su postgres -c 'createdb -p 9999 testdb'

到Node2(192.168.2.233)檢查是否存在

	su postgres -c psql
	\l

## 建立在線復原機制 ##
下載pgpool的原始碼，須符合系統安裝的版本，並安裝到每個Node

	cd /usr/local/src/pgpool-II-3.3.0
	cd sql/pgpool-regclass
	make install
	sudo -u postgres psql -f pgpool-regclass.sql template1
	cd ../pgpool-recovery
	make install
	sudo -u postgres psql -f pgpool-recovery.sql template1
	cd ../
	sudo -u postgres psql -f insert_lock.sql template1

安裝recovery function到template1，這樣新增的資料庫就會有此function，如果是已存在的資料庫也要新增進去，例如剛剛測試用的testdb。

	su postgres -c 'psql -f pgpool-recovery.sql testdb'

建立還原所需的指令檔案

	cd /usr/local/src/pgpool-II-3.3.0/sample
	cp pgpool_recovery_pitr pgpool_remote_start /var/lib/pgsql/data/

手動新增第1階段使用的copy_base_backup

	cd /var/lib/pgsql/data/
	vim copy_base_backup
	
	#! /bin/sh
	DATA=$1
RECOVERY_TARGET=$2
	RECOVERY_DATA=$3

	psql -c "select pg_start_backup('pgpool-recovery')" postgres
	echo "restore_command = 'cp /exchange/wal/`hostname -s`/%f %p'" > /var/lib/pgsql/data/recovery.conf
	tar -C /var/lib/pgsql/data -zcf /exchange/pgsql.tar.gz .
	psql -c 'select pg_stop_backup()' postgres

修改第2階段使用的pgpool_recovery_pitr

	archdir=/exchange/wal/`hostname -s`

修改最後階段使用的pgpool_remote_start

	#! /bin/sh
	
	if [ $# -ne 2 ]
	then
	    echo "pgpool_remote_start remote_host remote_datadir"
	    exit 1
	fi
	
	DEST=$1
	DESTDIR=$2
	PGCTL=/usr/bin/pg_ctl
	
	ssh $DEST tar zxf /exchange/pgsql.tar.gz -C /var/lib/pgsql/data
	ssh root@$DEST /etc/init.d/postgresql start

**原本檔案中應該是以postgres執行pg_ctl來啟動PostgreSQL，但是會發生資料庫已經啟動完成，ssh卻不會斷線的情況，導致online recovery的程序卡住，所以我改用執行init script的方式，因此需要root權限，不安全但是目前可行的作法。**

加上執行權限

	chmod +x copy_base_backup pgpool_recovery_pitr pgpool_remote_start

### 建立SSH key trust ###
建立key trust是因為在online recovery中會需要用到ssh到離線的node上複製DB檔案，以及遠端啟動資料庫。

1) 在Node1~3執行

    su - postgres
    ssh-keygen 
    Generating public/private rsa key pair.
    Enter file in which to save the key (/var/lib/pgsql/.ssh/id_rsa): 
    Enter passphrase (empty for no passphrase): 
    Enter same passphrase again: 
    Your identification has been saved in /var/lib/pgsql/.ssh/id_rsa.
    Your public key has been saved in /var/lib/pgsql/.ssh/id_rsa.pub.
    The key fingerprint is:
    xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx postgres@node1

2) 然後將public key複製到其他兩台

	cat ~/.ssh/id_rsa.pub

複製畫面上的內容之後，在Node2~3上執行

	cd ~/.ssh
	vim authorized_keys

將剛剛複製的內容貼入，並將檔案屬性改成600

	chmod 600 authorized_keys

3) 在Node2~3切回root帳號，同樣也要建立key trust

	mkdir ~/.ssh
	vim ~/.ssh/authorized_keys

貼入node1的public key，然後修正檔案與目錄權限

	chmod 700 ~/.ssh
	chmod 600 ~/.ssh/authorized_keys

到此Node1的postgres帳號便可以不用輸入密碼就可以登入Node2~3的postgres與root帳號。

然後在Node2~3重複前面的步驟1~3，依樣畫葫蘆。

### 測試Online Recovery ###
其實online recovery所需的3個檔案copy_base_backup pgpool_recovery_pitr pgpool_remote_start，也是需要複製到Node2~3的PostgreSQL的Cluster目錄，但是可以透過Online Recovery來做，順便讓資料庫保持一致。 :P

把Node2的PostgreSQL關閉後，執行show pool_nodes;狀態應該會顯示3，此時執行以下指令（如果你的recovery_user設定為postgres，就要切換到postgres去執行）

	pcp_recovery_node  900 node1 9898 postgres postgres_pass 1

可以執行pcp_recovery_node -h來看詳細說明

如果沒有什麼意外，執行完畢之後Node2的PostgreSQL會啟動，而執行show pool_nodes;就會顯示2。

另外要注意的是，在執行online recovery時，執行到第2階段，pgpool都是處於拒絕連線的狀態，除非執行完，這是避免執行online recovery時因為client連入導致資料不一致。

[1]: http://creativecommons.org/licenses/by-sa/3.0/tw/
