# Cloud Foundry x Spring Boot Actuator Hands on Lab

![image](https://user-images.githubusercontent.com/106908/27984834-d2636b42-641a-11e7-8344-009a8f38baaa.png)

## 事前準備

https://github.com/Pivotal-Japan/cf-workshop/blob/master/prerequisite.md

必須ではありませんが、ローカル環境でMySQLが使えると良いです。

---

## Spring Boot Actuatorをローカルで試す

まずは[サンプルアプリ](https://github.com/making/hello-db)をローカルで動かします。

MySQLで`messages`データベースを作成してください。

```
mysql -u ... 
> CREATE DATABASE messages;
```

サンプルアプリをビルドして実行します。

```
git clone https://github.com/making/hello-db
cd hello-db
./mvnw package -DskipTests=true
java -jar target/hello-db-0.0.1-SNAPSHOT.jar
```

> MySQLユーザー名が`root`、パスワードが空の設定になっています。この設定以外でMySQLにアクセスしたい場合は、次のオプションを指定してください。
>
> ```
> java -jar target/hello-db-0.0.1-SNAPSHOT.jar --spring.datasource.username=foo --spring.datasource.password=bar
> ```

> MySQLが使えない場合、インメモリなH2を使えます。
>
> ```
> ./mvnw clean package -P h2 -DskipTests=true
> java -jar target/hello-db-0.0.1-SNAPSHOT.jar --spring.profiles.active=h2
> ```

アプリケーションのエンドポイントは[http://localhost:8080/messages](http://localhost:8080/messages)です。

次のSpring Boot Actuarorエンドポイントにアクセスしてください。

* http://localhost:8080/env
* http://localhost:8080/health
* http://localhost:8080/info
* http://localhost:8080/metrics
* http://localhost:8080/dump
* http://localhost:8080/configprops
* http://localhost:8080/autoconfig
* http://localhost:8080/mappings
* http://localhost:8080/beans
* http://localhost:8080/trace
* http://localhost:8080/auditevents
* http://localhost:8080/loggers
* http://localhost:8080/flyway


次は`cloud`プロファイルを指定して実行しましょう。

```
java -jar target/hello-db-0.0.1-SNAPSHOT.jar --spring.profiles.active=cloud
```

どうなるでしょうか。

> H2を使う場合は
>
> ```
> java -jar target/hello-db-0.0.1-SNAPSHOT.jar --spring.profiles.active=h2,cloud
> ```

---

## Cloud Foundry上でのSpring Boot Actuator

ビルドしたアプリケーションをCloud Foundryに`cf push`デプロイします。
デプロイする前にMySQLのサービスインスタンスを`hello-db`という名前で作成します。

Pivotal Web Services上でこのハンズオンを実施する場合は、サブドメイン名が重複しないように、`--random-route`オプションをつけると良いです。

```
cf create-service cleardb spark hello-db
cf push --random-route
```

デプロイできたら、Spring Boot Actuatorのエンドポイントにアクセスしてみてください。403エラーになるでしょう。

* https://message-api-(random-words).cfapps.io/env
* https://message-api-(random-words).cfapps.io/health

![image.png](https://qiita-image-store.s3.amazonaws.com/0/1852/0db01062-d6e6-a806-d6c4-4669eff8d7fb.png)


> `/info`と`/health`は`management.security.enabled=true`の場合でもデフォルトでアクセス可能です。（`/health`に関しては詳細は出力されません）
> 今回は`application-cloud.properties`に次のプロパティを設定しているため、`/info`も`/health`もアクセスできません。
> 
> ``` properties
> endpoints.sensitive=true
> ```


Pivotal Web Servicesの[Apps Manager](https://console.run.pivotal.io)にアクセスしましょう。

![image.png](https://qiita-image-store.s3.amazonaws.com/0/1852/1bf16ceb-2bf2-5bca-fcf0-7c8a3bd0d63b.png)


### Spring Boot Actuatorのレスポンス確認

Spring Boot Actuatorの各種エンドポイントのレスポンスを確認できます。

#### `/health`エンドポイント

`Overview`タブです。

![image.png](https://qiita-image-store.s3.amazonaws.com/0/1852/2d26ad62-624e-ec92-fc1b-6a5073b2392e.png)

#### `/info`エンドポイント

`Settings`タブです。

![image.png](https://qiita-image-store.s3.amazonaws.com/0/1852/7c498614-c7ae-fdc4-584d-e90b6b294f84.png)

#### `/loggers`エンドポイント

`Logs`タブです。

![image.png](https://qiita-image-store.s3.amazonaws.com/0/1852/8add651f-dd91-e1ac-ec1b-5f2baab2e666.png)


#### `/trace`エンドポイント

`Trace`タブです。

![image.png](https://qiita-image-store.s3.amazonaws.com/0/1852/03651681-652d-bed1-349b-7cc81c7bc498.png)

#### `/dump`エンドポイント

`Threads`タブです。

![image.png](https://qiita-image-store.s3.amazonaws.com/0/1852/7c694738-cdec-8fbb-9b54-f7ad13495b1e.png)

#### `/heapdump`エンドポイント

`Overview`タブです。

![image.png](https://qiita-image-store.s3.amazonaws.com/0/1852/0f6627a8-07c1-7ef5-a0e1-8efdf66d2be5.png)

`hpref`ファイルをダウンロードし、`jvisualvm`コマンドや[Eclipse Memory Analyzer](http://www.eclipse.org/mat/)などで開いてみてください。



### スケールアウト

次のコマンドで2インスタンスにスケールアウトしても2インスタンス分のレスポンスを確認できます。

```
cf scale message-api -i 2
```

![image.png](https://qiita-image-store.s3.amazonaws.com/0/1852/a0b73758-8920-0b1c-22de-241787bad4a0.png)

---

## Metrics Forwarder Service

Java Buildpackが自動で設定してくれる[Metrics Writer](https://docs.spring.io/spring-boot/docs/current/reference/html/production-ready-metrics.html#production-ready-metric-writers)を試します。

Java BuildpackのMetrics Writerを有効にするには[Metrics Forwarder Service](https://github.com/cloudfoundry/java-buildpack/blob/master/docs/framework-metric_writer.md)が対象のアプリケーションにバインドされている必要があります。

> Metrics Fowarder Serviceがバインドされているかどうかは環境変数`VCAP_SERVICES`の中のService Instanceの`name`または`tag`または`label`に`metrics-forwarder`という文字列が含まれている必要があります。


今回はデモ用のMetrics Fowarder ServiceのService Brokerを用意していますので、こちらをマーケットプレースに登録して使用します。

#### Service Brokerのデプロイ

```
git clone git@github.com:making/prometheus-exporter-metrics-forwarder-service.git
cd prometheus-exporter-metrics-forwarder-service
```

`src/main/resources/catalog.json`を開き、次の`●●●●●●●●●●●●●●●●`, `▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲`, `★★★★★★★★★★★★★★★★★`の値を他の参加者と重複しないように設定してください

``` json
{
  "services": [
    {
      "id": "●●●●●●●●●●●●●●●●",
      "name": "▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲",
      "description": "Prometheus Exporter Metrics Forwarder Service",
      "bindable": true,
      "planUpdateable": false,
      "requires": [],
      "tags": [
        "metrics-forwarder"
      ],
      "metadata": {
        "displayName": "Prometheus Exporter Metrics Forwarder Service",
        "documentationUrl": "https://twitter.com/making",
        "imageUrl": "https://avatars0.githubusercontent.com/u/19211531",
        "longDescription": "Prometheus Exporter Metrics Forwarder Service",
        "providerDisplayName": "@making",
        "supportUrl": "https://twitter.com/making"
      },
      "plans": [
        {
          "id": "★★★★★★★★★★★★★★★★★",
          "name": "standard",
          "description": "Standard",
          "free": true,
          "metadata": {
            "displayName": "Standard",
            "bullets": [
              "Standard"
            ],
            "costs": [
              {
                "amount": {
                  "usd": 0
                },
                "unit": "MONTHLY"
              }
            ]
          }
        }
      ]
    }
  ]
}
```

ビルドして、メトリクスを保存するMySQLサービスインスタンスを作成し、`--no-start`をつけて`cf push`します。

```
./mvnw package -DskipTests=true
cf create-service cleardb spark metrics-db
cf push --random-route --no-start
```

startする前に次の環境変数を設定します。

```
cf set-env metrics-forwarder metric-forwarder-service.external-url https://metrics-forwarder-(random-words).cfapps.io
cf set-env metrics-forwarder cf.api-host api.run.pivotal.io
cf set-env metrics-forwarder cf.username (your-username)
cf set-env metrics-forwarder cf.password (your-password)
cf set-env metrics-forwarder cf.skip-ssl-validation false
cf start metrics-forwarder
```

> `metric-forwarder-service.external-url`に設定するURLがわからない場合は次のコマンドで確認してください。
> 
> ```
> $ LANG=C cf app metrics-forwarder
> 
> name:              metrics-forwarder
> requested state:   stopped
> instances:         0/1
> usage:             512M x 1 instances
> routes:            metrics-forwarder-bankable-casualist.cfapps.io
> last uploaded:     Mon 28 Aug 12:15:19 JST 2017
> stack:             cflinuxfs2
> buildpack:         https://github.com/cloudfoundry/java-buildpack#v3.19
> 
> There are no running instances of this app.
> ```


`catalog.json`に設定した`●●●●●●●●●●●●●●●●`の名前でデプロイした`metrics-fowarder`アプリケーションをService Brokerとして登録します。

```
cf create-service-broker ●●●●●●●●●●●●●●●● broker password https://metrics-forwarder-(random-words).cfapps.io --space-scoped
```


次のコマンドでService Brokerが登録されていることを確認してください。

```
$ cf service-brokers
name                      url
metrics-forwarder-tmaki   https://metrics-forwarder-bankable-casualist.cfapps.io
```

また`cf marketplace`で名前が`●●●●●●●●●●●●●●●●`であるサービスが登録されていることを確認してください。

```
$ cf marketplace
Getting services from marketplace in org ikam / space home as tmaki@pivotal.io...
OK

service                       plans                                                                                description
3scale                        free_appdirect, basic_appdirect*                                                     API Management Platform
app-autoscaler                standard                                                                             Scales bound applications in response to load
blazemeter                    free-tier, basic1kmr*, pro5kmr*                                                      Performance Testing Platform
cedexisopenmix                opx_global*, openmix-gslb-with-fusion-feeds*                                         Openmix Global Cloud and Data Center Load Balancer
cedexisradar                  free-community-edition                                                               Free Website and Mobile App Performance Reports
cleardb                       spark, boost*, amp*, shock*                                                          Highly available MySQL for your Apps.
cloudamqp                     lemur, tiger*, bunny*, rabbit*, panda*                                               Managed HA RabbitMQ servers in the cloud
cloudforge                    free, standard*, pro*                                                                Development Tools In The Cloud
elephantsql                   turtle, panda*, hippo*, elephant*                                                    PostgreSQL as a Service
gluon                         free, indie*, business*, enterprise*                                                 Mobile Synchronization and Cloud Integration
ironworker                    production*, starter*, developer*, lite                                              Job Scheduling and Processing
loadimpact                    lifree, li100*, li500*, li1000*                                                      Performance testing for DevOps
memcachedcloud                100mb*, 250mb*, 500mb*, 1gb*, 2-5gb*, 5gb*, 30mb                                     Enterprise-Class Memcached for Developers
memcachier                    dev, 100*, 250*, 500*, 1000*, 2000*, 5000*, 7500*, 10000*, 20000*, 50000*, 100000*   The easiest, most advanced memcache.
●●●●●●●●●●●●●●●●              standard                                                                             Prometheus Exporter Metrics Forwarder Service
mlab                          sandbox                                                                              Fully managed MongoDB-as-a-Service
newrelic                      standard                                                                             Manage and monitor your apps
p-circuit-breaker-dashboard   standard                                                                             Circuit Breaker Dashboard for Spring Cloud Applications
p-config-server               standard                                                                             Config Server for Spring Cloud Applications
p-service-registry            standard                                                                             Service Registry for Spring Cloud Applications
pubnub                        free                                                                                 Build Realtime Apps that Scale
rediscloud                    100mb*, 250mb*, 500mb*, 1gb*, 2-5gb*, 5gb*, 10gb*, 50gb*, 30mb                       Enterprise-Class Redis for Developers
searchify                     small*, plus*, pro*                                                                  Custom search you control
searchly                      small*, micro*, professional*, advanced*, starter, business*, enterprise*            Search Made Simple. Powered-by Elasticsearch
sendgrid                      free, bronze*, silver*                                                               Email Delivery. Simplified.
ssl                           basic*                                                                               Upload your SSL certificate for your app(s) on your custom domain
stamplay                      plus*, premium*, core, starter*                                                      API-first development platform
statica                       starter, spike*, micro*, medium*, large*, enterprise*, premium*                      Enterprise Static IP Addresses
streamdata                    spring, creek*                                                                       Efficiently Turn APIs into Real-time Experiences
```



##### Metrics Forwarder Serviceのサービスインスタンス作成

登録したサービスのサービスインスタンスを`demo-mf`と言う名前で作成します。

```
cf create-service ●●●●●●●●●●●●●●●● standard demo-mf
```

次に`message-api`アプリケーションにこのサービスインスタンスをバインドします。

```
cf bind-service message-api demo-mf
cf restage message-api
```

Message ForwarderがバインドされていることがStaging中に認識されると、

```
Downloaded build artifacts cache (45.7M)
-----> Java Buildpack Version: v3.19 | https://github.com/cloudfoundry/java-buildpack#ba06eae
-----> Downloading Open Jdk JRE 1.8.0_141 from https://java-buildpack.cloudfoundry.org/openjdk/trusty/x86_64/openjdk-1.8.0_141.tar.gz (found in cache)
       Expanding Open Jdk JRE to .java-buildpack/open_jdk_jre (1.2s)
-----> Downloading Open JDK Like Memory Calculator 2.0.2_RELEASE from https://java-buildpack.cloudfoundry.org/memory-calculator/trusty/x86_64/memory-calculator-2.0.2_RELEASE.tar.gz (found in cache)
       Memory Settings: -Xmx681574K -XX:MaxMetaspaceSize=104857K -Xss349K -Xms681574K -XX:MetaspaceSize=104857K
-----> Downloading Client Certificate Mapper 1.1.0_RELEASE from https://java-buildpack.cloudfoundry.org/client-certificate-mapper/client-certificate-mapper-1.1.0_RELEASE.jar (found in cache)
-----> Downloading Container Security Provider 1.7.0_RELEASE from https://java-buildpack.cloudfoundry.org/container-security-provider/container-security-provider-1.7.0_RELEASE.jar (found in cache)
-----> Downloading Metric Writer 1.5.0_RELEASE from https://java-buildpack.cloudfoundry.org/metric-writer/metric-writer-1.5.0_RELEASE.jar (0.1s) <====== ★★★
-----> Downloading Spring Auto Reconfiguration 1.12.0_RELEASE from https://java-buildpack.cloudfoundry.org/auto-reconfiguration/auto-reconfiguration-1.12.0_RELEASE.jar (found in cache)
```


#### Prometheus用のService Key作成

このサービスブローカーはMetrics Forwarderとしてだけでなく、PrometheusのExporterとしても動作します。Prometheusに登録するためのService Key(クライアントCredentials情報に相当)を作成します。

```
cf create-service-key demo-mf prometheus
```


作成したService Keyに対するCredentialsは次のコマンドで確認できます

```
$ cf service-key demo-mf prometheus
{
 "access_key": "'Bearer 79cae485-3979-42f0-bb59-204ff50582ea'",
 "endpoint": "https://metrics-forwarder-bankable-casualist.cfapps.io",
 "prometheus_exporter": "https://metrics-forwarder-bankable-casualist.cfapps.io/prometheus"
}
```

この`access_key`を使って`prometheus_exporter`のエンドポイントにアクセスして動作確認しましょう。

```
$ curl -H "Authorization: Bearer 79cae485-3979-42f0-bb59-204ff50582ea" https://metrics-forwarder-bankable-casualist.cfapps.io/prometheus

# HELP spring_nonheap_used
# TYPE spring_nonheap_used gauge
spring_nonheap_used{application_id="a1e4c622-14f3-45d3-8e76-71201dbed189",application_instance_index="0",application_instance_id="417d45d0-b34d-4f31-494f-2f15",application_name="N/A",} 65260.0 1503893979000

spring_nonheap_used{application_id="a1e4c622-14f3-45d3-8e76-71201dbed189",application_instance_index="1",application_instance_id="5f500ef6-8741-4fe0-74ce-e932",application_name="N/A",} 65345.0 1503894034000

# HELP spring_heap_committed
# TYPE spring_heap_committed gauge
spring_heap_committed{application_id="a1e4c622-14f3-45d3-8e76-71201dbed189",application_instance_index="0",application_instance_id="417d45d0-b34d-4f31-494f-2f15",application_name="N/A",} 300032.0 1503893979000

spring_heap_committed{application_id="a1e4c622-14f3-45d3-8e76-71201dbed189",application_instance_index="1",application_instance_id="5f500ef6-8741-4fe0-74ce-e932",application_name="N/A",} 300032.0 1503894034000

# HELP spring_processors
# TYPE spring_processors gauge
spring_processors{application_id="a1e4c622-14f3-45d3-8e76-71201dbed189",application_instance_index="0",application_instance_id="417d45d0-b34d-4f31-494f-2f15",application_name="N/A",} 4.0 1503893979000

spring_processors{application_id="a1e4c622-14f3-45d3-8e76-71201dbed189",application_instance_index="1",application_instance_id="5f500ef6-8741-4fe0-74ce-e932",application_name="N/A",} 4.0 1503894034000

# HELP spring_heap
# TYPE spring_heap gauge
spring_heap{application_id="a1e4c622-14f3-45d3-8e76-71201dbed189",application_instance_index="0",application_instance_id="417d45d0-b34d-4f31-494f-2f15",application_name="N/A",} 300032.0 1503893979000

spring_heap{application_id="a1e4c622-14f3-45d3-8e76-71201dbed189",application_instance_index="1",application_instance_id="5f500ef6-8741-4fe0-74ce-e932",application_name="N/A",} 300032.0 1503894034000

...
```

> **TODO**
> 
> `application_name="N/A"`となっているのは[prometheus-exporter-metrics-forwarder-service](https://github.com/making/prometheus-exporter-metrics-forwarder-service)のバグです...

#### Prometheusの設定

Prometheusを起動して、このPrometheus Exporterにアクセスします。

https://github.com/prometheus/prometheus/releases/tag/v1.7.1

からOSに合ったファイルをダウンロードしてください。

ダウンロードしたファイルを展開して、フォルダ内の`prometheus.yml`にService KeyのExporterの情報を設定します。

``` yaml
# my global config
global:
  scrape_interval:     15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).

  # Attach these labels to any time series or alerts when communicating with
  # external systems (federation, remote storage, Alertmanager).
  external_labels:
      monitor: 'codelab-monitor'

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
  # - "first.rules"
  # - "second.rules"

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: 'prometheus'

    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.

    static_configs:
      - targets: ['localhost:9090']
     
  #### ★★★★ ここから追加 (Service Keyの値を設定) ★★★★
  - job_name: 'metrics-forwarder'
    scrape_interval: 60s
    scrape_timeout: 10s
    metrics_path: /prometheus
    scheme: https
    static_configs:
    - targets:
      - metrics-forwarder-bankable-casualist.cfapps.io:443
    bearer_token: 'Bearer 79cae485-3979-42f0-bb59-204ff50582ea'
```

設定したら実行可能な`prometheus`ファイルを実行してください。

```
$ ./prometheus 
INFO[0000] Starting prometheus (version=1.7.1, branch=master, revision=3afb3fffa3a29c3de865e1172fb740442e9d0133)  source="main.go:88"
INFO[0000] Build context (go=go1.8.3, user=root@0aa1b7fc430d, date=20170612-11:47:52)  source="main.go:89"
INFO[0000] Host details (darwin)                         source="main.go:90"
INFO[0000] Loading configuration file prometheus.yml     source="main.go:252"
INFO[0000] Loading series map and head chunks...         source="storage.go:428"
INFO[0000] 0 series loaded.                              source="storage.go:439"
INFO[0000] Listening on :9090                            source="web.go:259"
INFO[0000] Starting target manager...                    source="targetmanager.go:63"
```

[http://localhost:9090/targets](http://localhost:9090/targets)にアクセスし、`metrics-forwarder`のStatusが`UP`になっていることを確認してください。

![image.png](https://qiita-image-store.s3.amazonaws.com/0/1852/57da74b4-cf97-35f8-3e7d-30a6f687b1d3.png)



[http://localhost:9090/graph?g0.range_input=15m&g0.expr=spring_mem_free&g0.tab=0&g1.range_input=1h&g1.expr=&g1.tab=1](http://localhost:9090/graph?g0.range_input=15m&g0.expr=spring_mem_free&g0.tab=0&g1.range_input=1h&g1.expr=&g1.tab=1)にアクセスし、Spring Boot Actuatorの`mem_free`メトリクスのグラフを確認してください。

![image.png](https://qiita-image-store.s3.amazonaws.com/0/1852/dac25200-d2d3-b4e6-c3a7-4664a8bb4db0.png)



> PrometheusをCloud Foundryにデプロイしたい場合は次の手順を行ってください。必ず、Linux用のファイルを使用してください。
> `prometheus.yml`の追加内容は同じです。`cf push`する前にymlを修正してください。
>
> ```
> wget https://github.com/prometheus/prometheus/releases/download/v1.7.1/prometheus-1.7.1.linux-amd64.tar.gz
> tar -zxvf prometheus-1.7.1.linux-amd64.tar.gz
> cd prometheus-1.7.1.linux-amd64
> sed -i -e 's|localhost:9090|localhost:8080|' prometheus.yml
> # configure prometheus.yml
> cf push prometheus -b binary_buildpack -c './prometheus -web.listen-address=:8080' -m 128m --random-route
> ```
>
> インスタンスを再起動、あるいはクラッシュした後自動復旧した場合はデータが全て消える点に注意していください。

> Clear DBのsparkプラン（無償プラン)を使用している場合は、Metrics Forwarder Service側で
> `com.mysql.jdbc.exceptions.jdbc4.MySQLSyntaxErrorException: User 'bf53c7c4079117' has exceeded the 'max_questions' resource (current value: 3600)`
> のエラーが出る場合があります。無償プランであるため、時間当たりのクエリ回数が制限されているためです。
>
> Java BuildpackのMetrics Writerはデフォルトで毎分1回メトリクスを送信します。この回数を減らすには`cloudfoundry.metrics`プロパティに送信間隔(ミリ秒)を指定してください。
>
> 5分間隔に変更する例:
>
> ```
> cf set-env message-api cloudfoundry.metrics.rate 300000
> cf restart message-api
> ```

#### Grafanaの設定


https://grafana.com/grafana/download/4.4.3

からOSに合ったtar.gzファイルをダウンロードしてください。(Macの場合は`brew install grafana`)

展開したフォルダの`bin`ディレクトリに移動して、

```
./bin/grafana-server web
```

を実行してください。

Macの場合は

```
brew services start grafana
```

[http://localhost:3000](http://localhost:3000)にアクセスして、ユーザー名:`admin`、パスワード:`admin`でログインしてください。

![image.png](https://qiita-image-store.s3.amazonaws.com/0/1852/a50f3023-051b-c825-6823-5957693a7cf6.png)

"Add data source"をクリックし、

* Name => prometheus
* Type => Prometheus
* Url => http://localhost:9090

を入力し、"Add"をクリックしてください。

![image.png](https://qiita-image-store.s3.amazonaws.com/0/1852/7bc4fc9c-b96c-1868-c324-7f8b38f764a0.png)


メニューから"Dashboards" -> "Import"を選択してください。

![image.png](https://qiita-image-store.s3.amazonaws.com/0/1852/da77d6e3-b68c-31c9-4c88-4fbd58121fbd.png)

"Upload .json File"で`prometheus-exporter-metrics-forwarder-service/dashboards/spring-boot-actuator-metrics.json`を選択してアップロードしてください。

![image.png](https://qiita-image-store.s3.amazonaws.com/0/1852/23de997d-d647-731a-a940-e86179050739.png)

Optionsのprometheusはprometheusを選択してください。

![image.png](https://qiita-image-store.s3.amazonaws.com/0/1852/87ad703d-b857-9aac-b22d-8256e7155b05.png)

"Import"をクリックしてください。次のようなダッシュボードが表示されれば成功です。

![image.png](https://qiita-image-store.s3.amazonaws.com/0/1852/91776fe7-667c-c048-c512-21827788b129.png)


> GrafanaをCloud Foundryにデプロイしたい場合は次の手順を行ってください。必ず、Linux用のファイルを使用してください。
>
> ```
> wget https://s3-us-west-2.amazonaws.com/grafana-releases/release/grafana-4.3.3.linux-x64.tar.gz 
> tar -zxvf grafana-4.3.3.linux-x64.tar.gz 
> mv grafana-4.3.3 grafana
> sed -i -e 's|^http_port = 3000$|http_port = 8080|' ./conf/defaults.ini
> cf push grafana -b binary_buildpack -c './bin/grafana-server web' -m 128m  --random-route
> ```
> 
> この場合は、PrometheusもCloud Foundryにデプロイしてください。datasourceを設定する際に、デプロイしたpromehtuesのURLを指定してください。
> インスタンスを再起動、あるいはクラッシュした後自動復旧した場合はデータが全て消える点に注意していください。



## 環境の削除

使用した環境は次の順番で削除してください。

```
cf d -f -r message-api
cf ds -f hello-db
cf ds -f demo-mf
cf delete-service-key -f demo-mf prometheus
cf delete-service-broker -f ●●●●●●●●●●●●●●●●
cf ds -f metrics-db
```




