## Kubernetes for intra-mart

- intra-mart の検証環境を Kubernetes 上に構築

### 利用手順

#### [1] Docker Desktop のインストール

#### [2] Git のインストール

- 下記の URL より、Git for Windows をダウンロードし、インストールします

https://gitforwindows.org/

#### [3] Docker プロジェクトのダウンロードと初期設定

- 任意のフォルダで、以下のコマンドを実行し、docker プロジェクトをダウンロードします

```sh
> git clone https://github.com/rinne-grid/docker-for-intra-mart im
> cd im
```

- war ファイルの配置用フォルダを作成します

```sh
> mkdir .\ap\war
```

#### [4] Juggling で war ファイルを作成

プロジェクト名を imart にして、必要なモジュールを選択し、設定を行います。
今回の Docker 環境をそのまま利用するためには、下記ファイルの設定を変更する必要があります

- storage-config.xml を設定する
- resin-web.xml を設定する
- 出力する war ファイル名を imart.war とする

##### storage-config.xml の設定

imart/config/storage-config.xml の 19 行目付近を以下のとおりに変更します

```xml
<root-path-name>/im-data/storage</root-path-name>
```

##### resin-web.xml の設定

imart/resin-web.xml 内容を下記のとおりにします

```xml
<web-app xmlns="http://caucho.com/ns/resin" xmlns:resin="urn:java:com.caucho.resin">
    <character-encoding>UTF-8</character-encoding>

    <log-handler name="" class="jp.co.intra_mart.common.platform.log.handler.JDKLoggingOverIntramartLoggerHandler"/>
    <logger name="debug.com.sun.portal" level="warning" />

    <!-- im_service(im_asynchronous) -->
    <resource jndi-name="jca/work" type="jp.co.intra_mart.system.asynchronous.impl.executor.work.resin.ResinResourceAdapter" />
    <jsp>
        <recycle-tags>false</recycle-tags>
    </jsp>
    <database jndi-name="jdbc/default">
        <driver>
            <type>org.postgresql.Driver</type>
            <url>jdbc:postgresql://db:5432/imart</url>
            <user>imart</user>
            <password>imart</password>
            <init-param>
                <param-name>preparedStatementCacheQueries</param-name>
                <param-value>0</param-value>
            </init-param>
        </driver>
        <max-connections>20</max-connections>
        <prepared-statement-cache-size>8</prepared-statement-cache-size>
    </database>
    <session-config>
        <reuse-session-id>false</reuse-session-id>
        <session-timeout>30</session-timeout>
    </session-config>

    <mime-mapping extension=".json" mime-type="application/json"/>
</web-app>
```

##### war ファイルの出力

imart.war という名称で war ファイルを出力したら、
プロジェクトの im/ap/war フォルダの中に、war ファイルをコピーします

#### [5] intra-mart のサイトから Linux の resin-pro をダウンロード

intr-mart のサイトにアクセスし、プロダクトファイルダウンロードボタンを押下します。

https://www.intra-mart.jp/download/library/

ライセンスキーを入力すると、ダウンロード可能なファイル一覧が表示されます。

なお、intra-mart サイトにも書いているとおり、.tar.gz が Linux 用の resin-pro になります。

https://www.intra-mart.jp/download/product/iap/setup/iap_setup_guide/texts/install/linux/resin_linux.html

最新の Resin<b>resin-pro-4.0.xx.tar.gz</b>を入手します。

#### [6] 7zip をダウンロード、インストール

tar.gz 形式のファイルを展開するため、この記事では 7zip を利用します。

https://sevenzip.osdn.jp/

1. resin-pro.4.0.xx.tar.gz を展開します
2. resin-pro.4.0.xx.tar ファイルが作成されます
3. resin-pro.4.0.xx.tar を展開します
4. resin-pro.4.0.xx フォルダが作成されます
5. resin-pro.4.0.xx フォルダの直下に、automake, bin といったフォルダが存在することを確認します

![resin-proフォルダ](http://www.rinsymbol.sakura.ne.jp/github_images/docker/blog.PNG)

#### [7] Docker プロジェクトのフォルダに resin-pro をコピー

- 上記の[6]の 5 のフォルダ「resin-pro.4.0.xx」の名称を resin-pro に変更します
- resin-pro フォルダを im/ap フォルダにコピーします

#### [8] プロジェクトのフォルダ構成の確認

- フォルダを確認し、以下の構成と同じになっていることを確認します
- ポイント
  - im/ap/resin-pro フォルダがあり、直下に automake 等のファイルが存在する
  - im/ap/war フォルダがあり、imart.war ファイルが存在する

```
im
│  .env
│  .gitignore
│  docker-compose.yml
│  README.md
│
└─ap
│  │  Dockerfile
│  │
│  ├─resin-pro
│  │  ├─automake  など
│  │
│  └─war
│     ├─imart.war
│
└─k8s
    │  001_setup.yaml など
```

#### [9] 必要に応じて、設定ファイルを変更する

- im/ap/resin-pro/conf/resin.properties の 82 行目付近 - jvm_args

-Xmx, -Xms の値が、初期状態だと 8192m(8GB)が設定されているため、自分の PC のメモリ状況に合わせて変更します

```ini
jvm_args : -Dfile.encoding=UTF-8 -Djava.io.tmpdir=tmp -Xmx1500m -Xms1500m -XX:+UseConcMarkSweepGC -XX:CMSInitiatingOccupancyFraction=30 -XX:NewSize=512m -XX:MaxNewSize=512m -XX:+CMSClassUnloadingEnabled -verbose:gc -XX:+PrintGCDetails -XX:+PrintGCDateStamps -XX:+HeapDumpOnOutOfMemoryError -Xloggc:log/gc.log -XX:+UseGCLogFileRotation -XX:NumberOfGCLogFiles=5 -XX:GCLogFileSize=10M
```

- HTTP プロキシの設定 im/.env

社内ネットワーク等で、プロキシサーバーを経由する必要がある場合、.env の HTTP_PROXY、HTTPS_PROXY に値を設定します

```env
HTTP_PROXY=http://user:password@server:port/
HTTPS_PROXY=http://user:password@server:port/
```


#### [10] Docker コンテナのビルド

- プロジェクトフォルダに移動します

```sh
> cd any_folder\im
```

- docker-compose を利用し、コンテナをビルドします

```sh
> docker-compose build --no-cache
```

#### [11] Docker DesktopのWSL上にリソースマウント用のディレクトリを作成する

```sh
> wsl -d docker-desktop 
> mkdir -p /mnt/host/wsl/docker-desktop-data/version-pack-data/community/k8s-pvs/pvc-imart-db
> mkdir -p /mnt/host/wsl/docker-desktop-data/version-pack-data/community/k8s-pvs/pv-imart-system-store
> mkdir -p /mnt/host/wsl/docker-desktop-data/version-pack-data/community/k8s-pvs/pv-imart-webapps
```

#### [12] Kubernetes クラスターにデプロイする

```sh
> cd k8s
> kubectl apply -f ./001_setup.yaml
> kubectl apply -f ./002_setup-db.yaml
> kubectl apply -f ./003_setup-ap.yaml
```

* 1分～2分経過後に以下の画面にアクセスします(自動でデプロイが始まるため、画面表示までに時間がかかります)

http://localhost:8080/imart/system/login

![tenant1](http://www.rinsymbol.sakura.ne.jp/github_images/docker/tenant.PNG)

- テナント ID は imart を指定します

![tenant2](http://www.rinsymbol.sakura.ne.jp/github_images/docker/tenant2.PNG)

- リソース参照名は一覧に表示されたものを選択します

![tenant3](http://www.rinsymbol.sakura.ne.jp/github_images/docker/tenant3.PNG)

- テナント登録を行い、しばらく待ちます

![tenant4](http://www.rinsymbol.sakura.ne.jp/github_images/docker/tenant4.PNG)

- テナント環境セットアップが適切に動作しているかどうかについては、Adminer からテーブル作成状況を参照することで確認できます
  - http://localhost:8889にアクセスします
  - Adminer が表示されるので、下記のとおり情報を入力します

| 情報名           | 入力情報   |
| ---------------- | ---------- |
| データベース情報 | PostgreSQL |
| サーバ           | db         |
| ユーザ名         | imart      |
| パスワード       | imart      |
| データベース     | imart      |

- テーブルの作成状況が確認できます(だいたい 500 テーブルくらいができたら、処理完了です)

![adminer2](http://www.rinsymbol.sakura.ne.jp/github_images/docker/adminer12.PNG)

![tenant5](http://www.rinsymbol.sakura.ne.jp/github_images/docker/tenant5.PNG)

- データベースやストレージ情報は WSL(docker-desktop-data) に保存しているため、データは永続化されています

![dashboard](http://www.rinsymbol.sakura.ne.jp/github_images/docker/dashboard.PNG)


#### [13] 複数PodでAPサーバーを起動する

* 起動するまで、長くて10分程度かかるため、気長に待ってあげてください。

```sh
> kubectl delete -f ./003_setup-ap.yaml
> kubectl apply -f ./ap.yaml

> kubectl get statefulset
# READYが2/2になってから、だいたい5～10分程度
# NAME            READY   AGE
# intra-mart-ap   2/2     0s
```

#### (任意手順) 複数Pod(StatefulSet)で起動できていることの確認

* システム管理画面にアクセス
  * http://localhost:8080/imart/system/login
* サービス設定の「ノード情報」からホスト名とIPアドレスを確認
  * http://localhost:8080/imart/system/service/status?
* 該当Podの削除

```sh
> kubectl delete pod/intra-mart-ap-{ここにホスト名の番号が入ります}
# 例：intra-mart-ap-2.intra-mart-ap.default.svc.cluster.local と表示されている場合
> kubectl delete pod/intra-mart-ap-2
```

* 以下のURLにアクセスして、ログアウト
  * http://localhost:8080/imart/logout
* もう一度システム管理画面にアクセス
  * http://localhost:8080/imart/system/login
* ホスト名(Pod)とIPアドレスが変わっていることが確認できます



#### (その他) 停止手順

```sh
> cd k8s
> kubectl delete -f ./ap.yaml
> kubectl delete -f ./002_setup-db.yaml
```

#### (その他) 永続化したデータやストレージの削除手順

```sh
> cd k8s
> kubectl delete -f ./001_setup.yaml
> wsl -d docker-desktop
> rm -rf /mnt/host/wsl/docker-desktop-data/version-pack-data/community/k8s-pvs/pvc-imart-db
> rm -rf /mnt/host/wsl/docker-desktop-data/version-pack-data/community/k8s-pvs/pv-imart-system-store
> rm -rf /mnt/host/wsl/docker-desktop-data/version-pack-data/community/k8s-pvs/pv-imart-webapps

```
