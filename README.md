# Hands On phpMyAdmin on Cloud Run to Cloud SQL

## 概要

Cloud Run 上の phpMyAdmin を経由して Cloud SQL にアクセスする環境を構築するハンズオンです

![](./overview.png)

+ 使用する phpMyAdmin は Docker Hub の公式コンテナイメージを使用します

```
### Docker Hub
https://hub.docker.com/_/phpmyadmin

### GitHub
https://github.com/phpmyadmin/docker
```

## ハンズオン

### 0. 事前の定義

+ 環境変数を設定します

```
export _gc_pj_id='Your Google Cloud Project ID'

export _common='phpmyadmin'
export _zone='asia-northeast1-b'
export _region='asia-northeast1'
export _date=`date +"%Y%m%d%H%M"`
```

+ Google Cloud と認証します

```
gcloud auth login --no-launch-browser
```

+ API の有効化をします
  + 一回やれば OK

```
gcloud beta services enable sqladmin.googleapis.com --project ${_gc_pj_id}
```

### 1. Cloud SQL の準備

+ Cloud SQL instance を作成します

```
gcloud beta sql instances create ${_common}-insrance-${_date} \
  --database-version=MYSQL_8_0 \
  --tier=db-custom-1-3840 \
  --zone=${_zone} \
  --root-password=password123 \
  --project ${_gc_pj_id} \
  --async
```

+ Cloud SQL database を作成します

```
gcloud beta sql databases create ${_common}-db \
  --instance=${_common}-insrance-${_date} \
  --project ${_gc_pj_id} \
  --async
```

+ Cloud SQL 用の User を作成します
  + **DB 内のユーザ名** = **${_common}** とした
  + **DB 内のユーザのパスワード** = **${_gc_pj_id}** とした

```
gcloud sql users create ${_common} \
  --instance=${_common}-insrance-${_date} \
  --password=${_gc_pj_id} \
  --project ${_gc_pj_id} \
  --async
```

![](./_img/01.png)

### 2. Service Account の作成

+ Cloud Run 用の Service Account を発行し、 Cloud SQL に接続できる最低限の Role を付与する

```
gcloud beta iam service-accounts create ${_common}-sa \
  --description="Cloud Run 用のサービスアカウント" \
  --display-name="${_common}-sa" \
  --project ${_gc_pj_id}
```

+ 上記で作成した Service Account に以下の Role を付与します
  + Role: **Cloud SQL Client**( `roles/cloudsql.client` )
  + [Cloud SQL | Connect from Cloud Run](https://cloud.google.com/sql/docs/mysql/connect-run#configure)

```
gcloud beta projects add-iam-policy-binding ${_gc_pj_id} \
  --member="serviceAccount:${_common}-sa@${_gc_pj_id}.iam.gserviceaccount.com" \
  --role="roles/cloudsql.client"
```

### 3. Cloud Run にデプロイをする

+ Cloud Run の Service をデプロイします
  + [phpMyaAmin](https://hub.docker.com/_/phpmyadmin) の最新ののコンテナイメージ `phpmyadmin:5.2.1-apache` を使う
  + `--service-account` : Cloud Run の Service Account の指定
  + `--add-cloudsql-instances` : 接続したい Cloud SQL Instance の情報
  + `--set-env-vars` : **phpmyadmin:5.2.1-apache** が読み込める設定を Cloud Run の環境変数として設定
  + `--session-affinity` : phpMyAdmin がセッションを必要とするため
  + `--max-instances` : 起動するインスタンス数の最大値
  + `--allow-unauthenticated` : サンプルなので allUser からのアクセスを許可

```
gcloud beta run deploy ${_common} \
  --project ${_gc_pj_id} \
  --image phpmyadmin:5.2.1-apache \
  --region ${_region} \
  --service-account "${_common}-sa@${_gc_pj_id}.iam.gserviceaccount.com" \
  --add-cloudsql-instances ${_gc_pj_id}:${_region}:${_common}-insrance-${_date} \
  --set-env-vars DB_USERNAME=${_common} \
  --set-env-vars DB_PASSWORD=${_gc_pj_id} \
  --set-env-vars PMA_HOST=localhost \
  --set-env-vars PMA_SOCKET="/cloudsql/${_gc_pj_id}:${_region}:${_common}-insrance-${_date}" \
  --set-env-vars APACHE_PORT=8080 \
  --session-affinity \
  --max-instances 1 \
  --allow-unauthenticated
```

+ Cloud Run の Service の状態を確認します

```
gcloud beta run services describe ${_common} \
  --region ${_region} \
  --project ${_gc_pj_id}
```

### 4. Web ブラウザから確認する

![](./_img/02.png)

![](./_img/03.png)

![](./_img/04.png)

![](./_img/05.png)

---> Cloud Run 上の phpMyAdmin 経由して Cloud SQL にアクセスすることが出来ました :)

## クリーンアップ

<details>
<summary>1. Cloud Run の Service と専用に作成した Service Account を削除する</summary>

+ Cloud Run の Service の削除をします

```
gcloud beta run services delete ${_common} \
  --region ${_region} \
  --project ${_gc_pj_id}
```

+ Service Account の削除をします

```
gcloud beta iam service-accounts delete ${_common}-sa@${_gc_pj_id}.iam.gserviceaccount.com \
  --project ${_gc_pj_id}
```


</details>

<details>
<summary>2. Cloud SQL のリソースを削除する</summary>

+ DB 用のユーザの削除をします

```
gcloud sql users delete ${_common} \
  --instance=${_common}-insrance-${_date} \
  --project ${_gc_pj_id} \
  --async
```

+ DB 内の Database の削除をします

```
gcloud beta sql databases delete ${_common}-db \
  --instance=${_common}-insrance-${_date} \
  --project ${_gc_pj_id}
```

+ Cloud SQL Instance 自体の削除をします

```
gcloud beta sql instances delete ${_common}-insrance-${_date} \
  --project ${_gc_pj_id} \
  --async
```

</details>

これでハンズオンは終了です。お疲れ様でした !! :)

## 次のステップ

以下のことに挑戦してみましょう!!

+ Cloud Run の前に [Cloud Load Balancing](https://cloud.google.com/load-balancing) を設置し [Identity-Aware Proxy](https://cloud.google.com/iap) にて Cloud IAM による認証をする
+ ID トークンを使って Cloud Run にて Cloud IAM による認証をする( 参考 [はてなぶろぐ | Cloud Run を ID トークンによる簡易認証で見てみる ( Authorization: Bearer )](https://iganari.hatenablog.com/entry/2022/02/12/173547) )

## 最後に

Have Fan!! :)
