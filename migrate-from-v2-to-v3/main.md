---
marp: true
theme: future

---
<!--
class: title
-->
![bg](./img/title.png)

# React Native + Firestoreで運用しているアプリをRESTful APIからGraphQLに移行した話


2020/12/17
@wheatandcat

---


## 自己紹介

<!--
class: content
-->


![bg](./img/introduction.png)

 - wheatandcat
 - フリーランスのエンジニア
 - React Native / Go / Nuxt.js / Firebase 周りが得意です


---

# ペペロミアについて

![bg right:43% height:100%](./img/about2.png)

React Native(Expo)で作成しているライフログアプリです。

Apple Store / Google Play Storeで配信中 
& 全コードGitHubで公開中

---

<!--
class: force
-->

# 今回やったこと!
---

アプリの**ｖ2**から**v3**へバージョンアップで、
API実装を**RESTful API**から**GraphQL**に完全移行しました。

---

# v2の構成

<!--
class: content
-->

|  項目  |  ログイン前  |  ログイン後  |
| ---- | ---- | ---- |
|  書き込み  |  SQLite  |  RESTful APIでサーバーを経由してFirestoreに書き込む  |
|  読み込み  |  SQLite  |  clientから直接Firestoreを読み込む  |

 - APIドキュメント: Swagger

---

# v2構成での問題点

<!--
class: thin
-->

 - ログイン前のSQLiteのDB構成を、**そのままFirestoreのデータ構成**にしていたのでFirestoreのメリットが活かなかった
 - clientから直接Firestoreを読み込む処理を宣言する関係でアプリのコードが多くなり、複雑になった
 - RESTful APIやFirestoreに関するTypeをフロントエンド、バックエンドの**プロジェクト毎にTypeを管理するのが辛い**
 - Firestoreのsecurity rulesが複雑で管理が難しい
 - Swaggerの更新を忘れて、**実装と乖離する**

---

# Firestoreの特徴

<!--
class: content
-->

 - データモデルは**階層型データ構造**
 - RDBのJOINは存在しない
 - where-inやarray-contains-anyは、**双方10件までしか取得できない制約がある**

<br />

### 結論:

上記の性質からRDBと同じような設計思想で進めると、どこかしらで躓きます。
という事でv3で、どの辺を改善したかという話が次になります。

---

# v3の構成

|  項目  |  ログイン前  |  ログイン後  |
| ---- | ---- | ---- |
|  書き込み  |  SQLite  |  GraphQLを経由してFirestoreに書き込む  |
|  読み込み  |  SQLite  |  GraphQLを経由してFirestoreを読み込む  |


---

# v3での改善点①
<!--
class: thin
-->

```
ログイン前のSQLiteのDB構成を、そのままFirestoreのデータ構成にしていたので
Firestoreのメリットが活かなかった 
```
 - データ構成を**Firestoreの活かせる作りに変更**した

```
clientから直接Firestoreを読み込む処理を宣言する関係でアプリのコードが多くなり、複雑になった
```

 - Firestoreのアクセスコードが**全てbackend側になったのでアプリ側の実装がシンプルになった**

---

# v3での改善点②

```
RESTful APIやFirestoreに関するTypeをアプリ内のコードで宣言し管理していたが、
サーバー側やWeb版のことを考慮するとプロジェクト毎にTypeを宣言する手間が
```
 - **graphql-codegen**を使用することでtypeは全て自動生成できるようになった

```
Firestoreのsecurity rulesが複雑で管理が難しい
```
 - Firestoreのアクセスが全てbackendになったため考慮する必要がなくなった

```
Swaggerの更新を忘れて、実装と乖離する
```
 - APIドキュメントは、**graphql-markdown**で自動生成できるので実装と乖離しなくなった

---

# 逆にv3の構成で失ったもの

 - Firestoreのアクセス全てbackendで実装したため**リアルタイムアップデートの機能は失った**
 
※一応GraphQLの**Subscriptions**を実装すれば、リアルタイムアップデートの実装も可能だがコストと見合わないため今回は見送っています

---

# データ構成の比較

### v2でのFirestoreのデータ構成

<!--
class: content
-->

```
├──users/:id
├──calendars/:id
├──items/:id
├──itemDetails/:id
└──expoPushTokens/:id
```

### v3でのFirestoreのデータ構成

```
version/1
    └── users/:id
             ├── expoPushTokens/:id 
             └── calendars/:date 
                           └──items/:id
                                  └──itemDetails/:id

```

---

# v3でのFirestoreのデータ構成のメリット

 - user_idのみでユーザーの全情報を取得できる
 - user_idと日付でcalendarの情報が取得できる
 - calendarのdocumentが日付なので、必ずユニークの構造になる
 - where-inやarray-contains-anyを使わずに書くデータを取得できる

 <br />

```
version/1
    └── users/:id
             ├── expoPushTokens/:id 
             └── calendars/:date 
                           └──items/:id
                                  └──itemDetails/:id

```

---

<!--
class: force
-->

# 各画面とデータの読み込み