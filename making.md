


## Azure Database for PostgreSQL（マスタ）を作る  
東日本リージョンに Azure Database for PostgreSQL マスタサーバを作成します。  
```
az postgres server create \
    -g sudo-rsg \
    -n sudopgsql-e-20191231 \
    -l japaneast \
    --admin-user pgadmin \
    --admin-password sudo#xxxxxxx \
    --sku-name GP_Gen5_2 \
    --ssl-enforcement Enabled \
    --storage-size 5120 \
    --backup-retention 7 \
    --version 10

az postgres server configuration set \
    -g sudo-rsg \
    --server-name sudopgsql-e-20191231 \
    --name azure.replication_support \
    --value "REPLICA"

az postgres server restart \
    -n sudopgsql-e-20191231 \
    -g sudo-rsg

# Creates the service endpoint
az network vnet subnet update \
    -g sudo-rsg \
    -n sudo-subnet02 \
    --vnet-name sudo-vnet \
    --service-endpoints Microsoft.SQL

az postgres server vnet-rule create \
    -n sudo-pgsqlRule \
    -g sudo-rsg \
    -s sudopgsql-e-20191231 \
    --vnet-name sudo-vnet \
    --subnet sudo-subnet02

```
## 読み取りレプリカを作成する  
西日本リージョンに Azure Database for PostgreSQL リードレプリカを作成します。  
```
az postgres server replica create \
    --name sudopgsql-w-20191231 \
    --source-server sudopgsql-e-20191231 \
    -g sudo-rsg \
    --location japanwest
```  



マスター サーバーのレプリカの一覧を表示します。  
```
az postgres server replica list \
    --server-name sudopgsql-e-20191231 \
    -g sudo-rsg \
    -o table
```


## マスタに userdb を作る  
非同期ではありますが、マスタサーバに`userdb`を作成し、リードレプリカに反映されるか見て見ましょう。  

```
az postgres db create \
    -n userdb \
    -g sudo-rsg \
    --server-name sudopgsql-e-20191231 \
    --charset UTF8 \
    --collation C
```  
マスタサーバの DB リストを見てみます。  
```
$ az postgres db list -g sudo-rsg --server-name sudopgsql-e-20191231 -o table
Charset    Collation                   Name               ResourceGroup
---------  --------------------------  -----------------  ---------------
UTF8       English_United States.1252  postgres           sudo-rsg
UTF8       English_United States.1252  azure_maintenance  sudo-rsg
UTF8       English_United States.1252  azure_sys          sudo-rsg
UTF8       C                           userdb             sudo-rsg
```

時間をおいてリードレプリカの DB リストを見てみます。  
```
$ az postgres db list -g sudo-w-rsg --server-name sudopgsql-w-20191231 -o table
Charset    Collation                   Name               ResourceGroup
---------  --------------------------  -----------------  ---------------
UTF8       English_United States.1252  postgres           sudo-w-rsg
UTF8       English_United States.1252  azure_maintenance  sudo-w-rsg
UTF8       English_United States.1252  azure_sys          sudo-w-rsg
UTF8       C                           userdb             sudo-w-rsg
```

`userdb`が反映されています。  



## マスタサーバのデータを更新する VM を作る  

プライベートサーバとして、`userdb`を更新する VM を構築します。 
余り難しいことはしないので、Ubuntu 18.04 LTS でチャチャっと作ります。  

## cron.dにファイルを作成する  
拡張子のない任意の名前のファイル = test_cron とします
```
# cp /etc/crontab /etc/cron.d/teikijikko_cron
```

`/etc/cron.d/teikijikko_cron`を編集します。  
`50 *    * * *   root    bash /home/ysdse/teikijikko.sh >> /var/log/teikijikko.log`を最終行に追記します。  
毎時 50 分に`/home/ysdse/teikijikko.sh`を実行します。  
設定が済んだら、サービスを再起動して動作確認をします。  

```
sudo service cron restart
sudo service cron status
```
`teikijikko.sh` は以下のようなないようにしています。  
```
$ cat teikijikko.sh
#!/bin/bash
date "+%Y-%m-%dT%H:%M:%S.%3NZ"
psql "host=sudopgsql10-20191226.postgres.database.azure.com port=5432 dbname=userdb user=sudopgadmin@sudopgsql10-20191226 password=sudo#00050546 sslmode=require" -c "UPDATE items SET  name = CONCAT('item-', id),  description = SUBSTRING(md5(clock_timestamp()::text), 1, 30)::varchar,  price = CEIL(random() * 10000),  delete_flag = mod((random() * 100)::int,2)::boolean,  created_at = to_date('2018-' || round((random() * (12 - 1))::numeric, 0) + 1 || '-' || round((random() * (28 - 1))::numeric, 0) + 1, 'YYYY-MM-DD'),  updated_at = to_date('2019-' || round((random() * (12 - 1))::numeric, 0) + 1 || '-' || round((random() * (28 - 1))::numeric, 0) + 1, 'YYYY-MM-DD');"
date "+%Y-%m-%dT%H:%M:%S.%3NZ"
```
上記の psql コマンドで下記のフォーマットのテーブルを全件書き変えています。  
```
# psql "host=sudopgsql10-20191226.postgres.database.azure.com port=5432 dbname=userdb user=sudopgadmin@sudopgsql10-20191226 password=sudo#00050546 sslmode=require" -c "select * from items limit 10;"
 id |  name   |          description           | price | delete_flag |     created_at      |     updated_at
----+---------+--------------------------------+-------+-------------+---------------------+---------------------
  1 | item-1  | 712d693b4f4141959fea6dd3cf4aa6 |  1021 | t           | 2018-07-24 00:00:00 | 2019-05-18 00:00:00
  2 | item-2  | 712d693b4f4141959fea6dd3cf4aa6 |  8676 | t           | 2018-03-19 00:00:00 | 2019-02-14 00:00:00
  3 | item-3  | 712d693b4f4141959fea6dd3cf4aa6 |   660 | t           | 2018-07-09 00:00:00 | 2019-07-09 00:00:00
  4 | item-4  | 712d693b4f4141959fea6dd3cf4aa6 |  5315 | t           | 2018-12-09 00:00:00 | 2019-08-19 00:00:00
  5 | item-5  | 712d693b4f4141959fea6dd3cf4aa6 |  2887 | t           | 2018-10-15 00:00:00 | 2019-07-23 00:00:00
  6 | item-6  | 712d693b4f4141959fea6dd3cf4aa6 |  5005 | t           | 2018-08-24 00:00:00 | 2019-03-17 00:00:00
  7 | item-7  | 712d693b4f4141959fea6dd3cf4aa6 |  2143 | f           | 2018-01-23 00:00:00 | 2019-05-15 00:00:00
  8 | item-8  | 712d693b4f4141959fea6dd3cf4aa6 |  7442 | f           | 2018-09-23 00:00:00 | 2019-10-08 00:00:00
  9 | item-9  | 712d693b4f4141959fea6dd3cf4aa6 |  6740 | f           | 2018-08-04 00:00:00 | 2019-04-12 00:00:00
 10 | item-10 | 712d693b4f4141959fea6dd3cf4aa6 |  3374 | f           | 2018-12-04 00:00:00 | 2019-10-19 00:00:00
(10 rows)
```
この書き変えのレプリケーションを監視してみようと思います。  
## レプリケーションを監視する  
Azure Database for PostgreSQL には、レプリケーションを監視するための 2 つのメトリックがあります。  

| メトリック 	| メトリックの表示名 	| 単位 	| 説明 	|
|---------------------------------	|------------------------------------------------	|---------	|--------------------------------------------------------------------------------------------------------------------	|
| pg_replica_log_delay_in_bytes 	| Max Lag Across Replicas (レプリカ間の最大ラグ) 	| Byte 	| マスターと最も遅れているレプリカの間のバイト単位でのラグ。 このメトリックは、マスター サーバーのみで使用できます。 	|
| pg_replica_log_delay_in_seconds 	| Replica Lag (レプリカ ラグ) 	| Seconds 	| 最後に再生されたトランザクションからの時間。 このメトリックは、レプリカ サーバーのみで使用できます。 	|

毎時 50 分に発生するマスタサーバでの書き変えが、リードレプリカに反映する際の遅延がポータルにあるメトリックでグラフ化できるはずです。
