# docker-postgres
## 目的
-　Docker を利用してデータベース（PostgreSQLを利用）を構築する手順を確認する

## 注意事項
- この作業で構築したデータベースを、**お客様向けシステムに使用しないこと**

## 前提条件
- 事前に PC に **Docker Desktop** をインストールしていること
- **Docker Desktop** の状態が　`Engine Running` になっていること

## 作業のおおまかな流れ
1. Docker で PostgreSQL のコンテナを立ち上げる
1. 上記 1. で立ち上げた PostgreSQL のデータベースにデータを投入する
1. 上記 2. のデータをアプリ （C# で実装） で取得

## Docker で PostgreSQL のコンテナを立ち上げる
```txt:ディレクトリ構成
docker-postgres
└ docker-compose.yml
```

```yml:docker-compose.yml
# docker-composeで使用するバージョンを定義（2022年5月時点では、3.9が最新）
version: '3.9'
# サービス (コンテナ) を定義
services:
  postgres:
    # Docker Image は postgres:12-alpine を使用
    # postgres:12-alpine は postgres:12 と比較して、イメージサイズが小さい
    image: postgres:12-alpine
    # コンテナの名前を指定
    container_name: postgres
    # 環境変数を設定
    environment:
      - POSTGRES_USER=root
      - POSTGRES_PASSWORD=secret
      - POSTGRES_DB=mydb
    # データの永続化
    volumes:
      # postgresディレクトリを/var/lib/postgresql/dataにマウントする
      - postgres:/var/lib/postgresql/data
    # ポートの指定（HOST:CONTAINER）
    ports:
      - 5432:5432
# データの永続化
volumes:
  postgres:
```

`docker-postgres` ディレクトリに移動した上で、下記コマンドを実行しコンテナを立ち上げる
```
docker compose up -d
```

下記コマンドでコンテナの状況を確認する
（この時 `CONTAINER ID` の値を覚えておく）
```
docker ps
```
```txt:上記コマンド実行結果
CONTAINER ID   IMAGE                COMMAND                  CREATED        STATUS        PORTS                    NAMES
0721f4708e47   postgres:12-alpine   "docker-entrypoint.s…"   26 hours ago   Up 26 hours   0.0.0.0:5432->5432/tcp   postgres
```

**docker-compose.yml のコード引用先**

https://zenn.dev/farstep/books/7a6eb67dd3bf1f/viewer/fd9e23


## PostgreSQL のデータベースにデータを投入する
下記コマンドを実行し、コンテナ内に入る
（`072` の部分は `CONTAINER ID` を入力すること ※`CONTAINER ID` の一部でも良い）
```
docker exec -it 072 /bin/bash
```
```txt:実行結果
0721f4708e47:/# 
```

ルートユーザでログイン
```
0721f4708e47:/# psql -U root -d mydb
```
```txt:実行結果
psql (12.15)
Type "help" for help.
```

データベースの作成
```
mydb=# create database test;
```
```txt:実行結果
CREATE DATABASE
```

使用するデータベースの変更
```
mydb=# \c test;
```
```txt:実行結果
You are now connected to database "test" as user "root".
```

テーブルの作成
```
test=# create table users (id integer,name varchar(10),age integer);
```
```txt:実行結果
CREATE TABLE
```

テーブルへのデータ投入
```
test=# INSERT INTO users (id, name, age) VALUES (1, 'Mike', 30);
```
```txt:実行結果
INSERT 0 1
```
```
test=# INSERT INTO users (id, name, age) VALUES (2, 'Lisa', 24);
```
```txt:実行結果
INSERT 0 1
```
```
test=# INSERT INTO users (id, name, age) VALUES (3, 'Taro', 35);
```
```txt:実行結果
INSERT 0 1
```

データの取得
```
test=# select * from users;
```
```txt:実行結果
 id | name | age 
----+------+-----
  1 | Mike |  30
  2 | Lisa |  24
  3 | Taro |  35
(3 rows)
```

PostgreSQL と Docker コンテナから抜ける
```
test-# \q
```
```txt:実行結果
0721f4708e47:/# 
```
```
0721f4708e47:/# exit
```
```txt:実行結果
exit
```

## Docker コンテナを停止・削除
下記コマンドでコンテナの状況を確認する
```
docker ps
```
```txt:上記コマンド実行結果
CONTAINER ID   IMAGE                COMMAND                  CREATED        STATUS        PORTS                    NAMES
0721f4708e47   postgres:12-alpine   "docker-entrypoint.s…"   26 hours ago   Up 26 hours   0.0.0.0:5432->5432/tcp   postgres
```

下記コマンドでコンテナを停止する
```
docker stop 072
```

下記コマンドでコンテナを削除する
```
docker rm 072
```

下記コマンドでコンテナイメージ一覧を表示する
```
docker images
```

上記コマンド実行結果
（ここで `IMAGE ID` の値を覚えておく）
```
REPOSITORY   TAG         IMAGE ID       CREATED       SIZE
postgres     12-alpine   e4d9d6512901   4 weeks ago   230MB
```

下記コマンドでコンテナイメージを削除
（`e4d` は `IMAGE ID` の値 ※`IMAGE ID`の値の一部でも良い）
```
docker rmi e4d
```


**参考にした資料**

https://www.javadrive.jp/postgresql/


## 〈補足〉C# で実装したアプリを使用してデータを取得
**参考にした資料**

https://itsakura.com/csharp-postgresql-select

```Csharp:Program.cs
using System;
using Npgsql; // NuGet を利用して導入

namespace ConnectPostgreSQL
{
    class Program
    {
        static void Main(string[] args)
        {
            ConnectService service = new ConnectService();
            service.Connect();
        }
    }

     class ConnectService
    {
         public void Connect()
        {
            string sql = "SELECT * FROM Users";
            string ConnectionString = "Server=localhost;"
                + "Port=5432;"
                + "Database=test;"
                + "User ID=root;"
                + " Password=secret;";

             try
            {
                using (var connection = new NpgsqlConnection(ConnectionString))
                {
                    connection.Open();
                     using( var cmd = new NpgsqlCommand(sql, connection))
                    {
                        using ( var reader = cmd.ExecuteReader())
                        {
                            while(reader.Read())
                            {
                                Console.WriteLine(reader["id"] + ":" + reader["name"] + ":" + reader["age"]);
                            }
                        }
                    }
                }
            } catch(Exception e)
            {
                Console.WriteLine(e.Message.ToString());
            }
        }
    }
}

```

## アプリ （C#） 実行結果
```
1:Mike:30
2:Lisa:24
3:Taro:35
```
