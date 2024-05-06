# Elasticserach-Security

- 作業用メモ

## 1. Elastic Stack の基本的なセキュリティをセットアップする
- 参考サイト
  - https://www.elastic.co/guide/en/elasticsearch/reference/8.13/security-basic-setup.html

### 1-1. 認証局を生成する
1. クラスターのCAを生成する
  - command
    ```
    ./bin/elasticsearch-certutil ca
    ```
  - output
    - elastic-stack-ca.p12
    - CAの公開証明書、ノードの証明書の署名に使用される秘密キーが含まれる

2. 任意の単一ノードでクラスター内のノードの証明書・秘密鍵を生成する
  - command
    ```
    ./bin/elasticsearch-certutil cert --ca elastic-stack-ca.p12
    ```
  - output
    - elastic-certificates.p12
    - ノード証明書、ノードキー、および CA 証明書が含まれる

3. クラスター内のすべてのノード で、elastic-certificates.p12ファイルをディレクトリにコピーします$ES_PATH_CONF

### 1-2. TLSによるノード間通信の暗号化

1. elasticsearch.ymlの編集
- 次の設定を追加してノード間通信を有効にし、ノードの証明書へのアクセスを提供します
  ```
  xpack.security.transport.ssl.enabled: true
  xpack.security.transport.ssl.verification_mode: certificate
  xpack.security.transport.ssl.client_authentication: required
  xpack.security.transport.ssl.keystore.path: elastic-certificates.p12
  xpack.security.transport.ssl.truststore.path: elastic-certificates.p12
  ```

2. ノード証明書のパスワードをkeystoreに登録する
- command
  ```
  ./bin/elasticsearch-keystore add xpack.security.transport.ssl.keystore.secure_password
  ./bin/elasticsearch-keystore add xpack.security.transport.ssl.truststore.secure_password
  ```

## 2. Elastic Stack の基本的なセキュリティと安全な HTTPS トラフィックをセットアップする
- 参考サイト
  - https://www.elastic.co/guide/en/elasticsearch/reference/8.13/security-basic-setup-https.html#encrypt-kibana-http

### 2-1. Elasticsearch の HTTP クライアント通信を暗号化する

1. 証明書署名リクエスト (CSR) を生成
  - command
    ```
    ./bin/elasticsearch-certutil http
    ```
  - output
    - elasticsearch-ssl-http.zip

2. ノードの証明書を生成した後、プロンプトが表示されたら秘密キーのパスワードを入力

3. elasticsearch-ssl-http.zipを解凍する
  - elasticsearch
    ```
    /elasticsearch
    |_ README.txt
    |_ http.p12
    |_ sample-elasticsearch.yml
    ```
  - kibana
    ```
    /kibana
    |_ README.txt
    |_ elasticsearch-ca.pem
    |_ sample-kibana.yml
    ```

- elasticsearch.yml
  ```
  xpack.security.http.ssl.enabled: true
  xpack.security.http.ssl.keystore.path: http.p12
  ```

- command
  ```
  ./bin/elasticsearch-keystore add xpack.security.http.ssl.keystore.secure_password
  ```

### 2-2. Kibana の HTTP クライアント通信を暗号化する

#### 2-2-1. Kibana と Elasticsearch 間のトラフィックを暗号化する
- kibana.yml
  ```
  elasticsearch.ssl.certificateAuthorities: $KBN_PATH_CONF/elasticsearch-ca.pem
  elasticsearch.hosts: https://<your_elasticsearch_host>:9200
  ```

#### 2-2-2. ブラウザと Kibana 間のトラフィックを暗号化する
- command
  ```
  ./bin/elasticsearch-certutil csr -name kibana-server -dns example.com,www.example.com
  ```
- output
  - csr-bundle.zip

```
/kibana-server
|_ kibana-server.csr
|_ kibana-server.key
```

- kibana.yml
  ```
  server.ssl.enabled: true
  server.ssl.certificate: $KBN_PATH_CONF/kibana-server.crt
  server.ssl.key: $KBN_PATH_CONF/kibana-server.key
  ```
