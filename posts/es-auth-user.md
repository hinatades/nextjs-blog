---
title: 'Elasticsearchの認証ユーザーを構築前に事前登録する'
date: '2020-04-04'
---

# 概要

ElasticsearchはX-packセキュリティを有効にすることで、**無料の範囲内で**Basic認証を設定可能です。
設定方法は[Elasticのブログ](https://www.elastic.co/jp/blog/getting-started-with-security)などに詳しくまとまっていますが、クラスター構築後にユーザー登録で手作業が発生してしまいます。
ESをDocker imageやTerraformなどを使って自動構築できる作りにしてあると、なんとか手作業を避けたいです。

この記事ではESの起動時に各ノードにBasic認証が設定された状態で起動する方法を紹介します。
対象のバーションはElasticsearch7で、Docker imageで構築することを前提としています。


## Elasticsearch

```yaml:elasticsearch.yml
xpack.security.enabled: true
```

の一行を追加して X-packセキュリティを有効にします。 
これだけでBasic認証が有効になります。 この設定のESを起動して、`curl -I http://<ES_NODE>:9200` などでアクセスを試みると、`HTTP/1.1 401 Unauthorized` が返されることがわかります。
ESはビルトインユーザとして4ユーザー(`elastic` 、`kibana`、`logstash_system`、`beats_system`）が事前に登録されています。

これらのユーザーのパスワードは

```sh
$ sudo bin/elasticsearch-setup-passwords interactive
```


を実行することでインタラクティブに設定可能なので、これをDockerfileに記述して起動前に実行することで、各ユーザーのパスワードを事前に設定しておくことができます。
`elastic` のパスワードについては、以下のコマンドを使うことで、Dockerfileから直接実行することができます。

```Dockerfile:Dockerfile
RUN echo <YOUR_PASSWORD> | bin/elasticsearch-keystore add "bootstrap.password" -xf
```

 `<YOUR_PASSWORD> `には設定したいパスワードを記載します。
パスワードは秘匿情報のため、Github等でバージョン管理している場合は、CIから渡す作りにするといいでしょう。

```sh

$ sed -i "s/<YOUR_PASSWORD>/${YOUR_PASSWORD}/g" Dockerfile
```


以上の設定で、 `elastic` ユーザのパスワードが設定された状態で起動されます。構築されたESに対して

```
$ curl -u elastic:<YOUR_PASSWORD> <ES_NODE>:9200/_cluster/state
```

のように認証付きでリクエストすると、検索やindexingが可能なことがわかります。


## Kibana

kibanaからBasic認証を有効にしたESクラスターに対してアクセスすると、以下のように認証情報の入力を求められます。

<img width="40%" alt="kibana" src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/81805/6b29e2ad-da7a-6aaf-7407-41e33d91a9a8.png">

これを突破するには、 kibana.yml にESに登録した認証情報を追加します。

```yml:kibana.yml
x-packxpack.monitoring.enabled: true
elasticsearch.username: elastic
elasticelasticsearch.password: <YOUR_PASSWORD>
```

## Kibana Monitoring


Basic認証を有効にしたESクラスターに対して、Kibanaモニタリングを使う場合は、別途 elasticsearch.ymlにモニタリング用の認証情報を追加する必要があります。

```yml:elasticsearch.yml
xpack.monitoring.exporters.my_remote.auth.username: elastic
xpack.monitoring.exporters.my_remote.auth.password: <YOUR_PASSWORD>
```

これを設定することで、Kibanaモニタリングを使うことができます。

<img width="70%" alt="kibana" src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/81805/7c0d6274-1492-a8df-c015-b164683a2a77.png">
