# Spring Cloud Stream

## RabbitMQを起動する(via Docker)

管理画面を見たいので`-management`が付いているバージョンを使用する。

```sh
docker run -d --name mq -h usaq -p 5672:5672 -p 15672:15672 rabbitmq:3-management
```

次のURLで管理画面を開く。

- http://localhost:15672/

ユーザー名・パスワードはデフォルトだとどちらも`guest`。

## メッセージ送信側アプリケーションを起動する

HTTPで受け取った名前を`Person`にセットしてキューへ送信するアプリケーション。

```sh
cd source-app
mvnw package
java -jar target/source-app.jar
```

## メッセージ受信側アプリケーションを起動する

キューから受信した`Person`を標準出力へ書き出すアプリケーション。

```sh
cd sink-app
mvnw package
java -jar target/sink-app.jar
```

## メッセージを送信する

`source-app`へHTTPで`name`を送る。

```sh
curl localhost:8080 -d name=hoge
```

そうすると、`SourceApp#handle`がHTTPリクエストを受け取って`sample` exchangeへメッセージを送信する。

ここで送信先となるexchangeは`application.properties`に書かれた`spring.cloud.stream.bindings.output.destination`の値で設定される。
デフォルトだとbinding target name、つまり今回だと`output`になる。

exchangeへ送信されたメッセージはバインドされているキューへ送信される。

キューはデフォルトだと`sink-app`のインスタンス毎に1つ用意されるが、グループが設定されている場合はグループ毎に1つ用意される。
グループは`application.properties`の`spring.cloud.stream.bindings.input.group`の値で設定される。

キューがどのexchangeへバインドされるかは`application.properties`に書かれた`spring.cloud.stream.bindings.input.destination`の値で設定される。
デフォルトだとbinding target name、つまり今回だと`input`になる。

`SinkApp#handle`に受信したメッセージが渡され、標準出力に書き出される。

### 参考

- binding target name: `org.springframework.cloud.stream.annotation.Output.value()`
- binding target name: `org.springframework.cloud.stream.annotation.Input.value()`
- `org.springframework.cloud.stream.config.BindingServiceProperties`
- `org.springframework.cloud.stream.config.BindingProperties`

## 冗長化

RabbitMQクラスタを構築する。

まず1つめのRabbitMQを起動する。

```sh
docker run -d -h usaq1 --name mq1 -e RABBITMQ_ERLANG_COOKIE=secret -p 5672:5672 -p 15672:15672 rabbitmq:3-management
```

次に2つめのRabbitMQを起動して`docker exec`で`bash`を起動する。

```sh
docker run -d -h usaq2 --name mq2 -e RABBITMQ_ERLANG_COOKIE=secret -p 5673:5672 --link mq1 rabbitmq:3-management
docker exec -it mq2 bash
```

`rabbitmqctl`コマンドを使用してクラスタを構築する。

```sh
rabbitmqctl stop_app
rabbitmqctl join_cluster rabbit@usaq1
rabbitmqctl start_app
rabbitmqctl set_policy ha-two "^sample" '{"ha-mode":"exactly","ha-params":2,"ha-sync-mode":"automatic"}'
```

メッセージ送信アプリケーションを複数起動する。

```sh
java -jar target/source-app.jar --server.port=8080 --spring.rabbitmq.addresses=localhost:5672,localhost:5673
java -jar target/source-app.jar --server.port=8081 --spring.rabbitmq.addresses=localhost:5672,localhost:5673
```

メッセージ受信アプリケーションを複数起動する。

```sh
java -jar target/sink-app.jar --server.port=8082 --spring.rabbitmq.addresses=localhost:5672,localhost:5673
java -jar target/sink-app.jar --server.port=8083 --spring.rabbitmq.addresses=localhost:5672,localhost:5673
```

`source-app`へHTTPで`name`を送る。

```sh
# 本来は前にロードバランサーを置く
curl localhost:8080 -d name=hoge
curl localhost:8081 -d name=hoge
```
