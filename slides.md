---
theme: default
# background: https://source.unsplash.com/collection/94734566/1920x1080
background: https://images.unsplash.com/photo-1621112943521-775623b0f651?crop=entropy&cs=tinysrgb&fit=crop&fm=jpg&h=1080&ixid=MnwxfDB8MXxyYW5kb218fHx8fHx8fHwxNjI0Mzc0Nzg2&ixlib=rb-1.2.1&q=80&utm_campaign=api-credit&utm_medium=referral&utm_source=unsplash_source&w=1920
class: 'text-center'
highlighter: shiki
info: |
  ## Slidev Starter Template
  Presentation slides for developers.

  Learn more at [Sli.dev](https://sli.dev)
fonts:
  sans: 'Roboto'
  serif: 'Roboto'
  mono: 'Fira Code'
---

# "How to Structure Your Data" から<br>学ぶ Firestore のデータモデリング

2021-06-23 (水) GDG Saitama LT by Kosuke Saigusa

---

## 自己紹介

- @KosukeSaigusa (GitHub, Twitter, Qiita, Zenn)
- 1995 年生まれ 26 歳
- 都内の漫画を販売する会社で Flutter エンジニアとして勤務
- Django/Python の バックエンド、Nuxt.js/Vue.js の Web フロントエンドのタスクも
- フルタイムでエンジニアとして働くようになったのは 2021 年の 3 月から
- 学生時代は機械振動学の研究に従事。C, Fortran で数値計算、Python で機械学習・深層学習なども
- 卒業後は友人と医療系 SaaS の会社を創業、次第にコードは書かずに PO, PM の役割へ
- いろいろあって退社、やっぱり自分で手を動かすエンジニアでいたい！と思い、現在の会社へ転職
- Firebase は個人アプリや友人とのプロジェクトの範囲で好んで触ってきた

エンジニアとしての経験はまだまだですが、エンジニアの**コミュニティ活動・OSS 活動の考え方に共感**しており、LT にもチャレンジしてみることにしました！

---

## 背景

いくつか個人開発や友人とのプロジェクトで、Cloud Firestore を用いてアプリをリリースしてみたが、より良いデータモデリングをきちんと勉強していなかった。

Firebase 公式の "How to Structure Your Data" から学んだ内容をまとめる LT とします！

<iframe width="560" height="315" src="https://www.youtube.com/embed/haMOUb3KVSo" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

---

## 今日学べること

1. レビューデータはどこに保存するべきか<br>• サブコレクション vs. トップレベルコレクション<br>• パフォーマンスが良いのはどちら？ / セキュリティルールを書きやすいのはどちら？<br>
2. Cloud Firestore でリレーションを表現する方法<br>• サーバサイドジョインとクライアントサイドジョイン / N + 1 問題<br>• 非正規化を活用する
3. 権限管理のための情報を保存する<br>• User ID の漏洩は大丈夫？<br>• 機密レベルの異なるフィールドを混在させない原則
4. Cloud Firestore で N 対 N のリレーション<br>• お気に入りのレストランに★マークを付ける<br>•「私のお気に入りレストラン一覧」を表示する<br>• レストランが日本料理屋からハンバーガー屋に変わったらどうする...？<br>（Cloud Functions のバックグラウンド関数）

---
layout: image-right
image: /images/restaurant-list.png
---

## 対象にするアプリ

レストランのレビューアプリ

- 一般ユーザーは、レストランを探したり、レビューを見たり、書いたり、レストランをお気に入りに登録したりする
- レストランのユーザーは、自分のレストランの情報を編集する

---

## レビューデータはどこに保存するべきか？

候補 1：レストランドキュメントの **Array や Map のフィールド**に保存する

- レストランドキュメントを取得した時点で、レビューが既に取得できている
- 「東京にある日本料理レストランのレビュースコア TOP 10」のようなクエリは発行できない
- 「サービス全体の最新のレビュー 10 件」のようなクエリも発行できない

```json
"<restaurant ID（ドキュメント）>": {
  "name": "割烹メジャーリーグ",
  "address": "東京都新宿区1-2-3",
  "category": "日本料理",
  "averageScore": 4.7,
  "reviews": [
    {
      "name": "大谷翔平",
      "score": 5,
      "content": "ランチの焼き魚定食が絶品です！..."
    },
    {
      "name": "ダルビッシュ有",
      "score": 4,
      "content": "旬の魚を刺し身にして出してくれます..."
    }
  ]
}
```

---

## レビューデータはどこに保存するべきか？

候補 2：レストランドキュメントの**サブコレクション**に保存する

- レストラン–レビューの 1:N のリレーションが直感的に表現できる
- CollectionGroup クエリを使えば「東京にある日本料理レストランのレビュースコア TOP 10」や「サービス全体の最新のレビュー 10 件」のようなクエリも実現できる
- ネストされた子のドキュメントに対するセキュリティルールは、その親のドキュメントに対する読み書きを許すかどうかの条件を引き継げるので記述が楽になる

```json
"<restaurant ID（ドキュメント）>": {
  "name": "割烹メジャーリーグ",
  "address": "東京都新宿区1-2-3",
  "category": "日本料理",
  "averageScore": 4.7,
  "<reviews（サブコレクション）>": {
    "<review ID（ドキュメント）>": {
      "name": "大谷翔平",
      "score": 5,
      "content": "ランチの焼き魚定食が絶品です！..."
    },
    // ...
  }
}
```

---

## レビューデータはどこに保存するべきか？

候補 3：**トップレベルコレクション**に保存する

- レストラン–レビューの 1:N リレーションは、レビュードキュメントのフィールドにレファレンスを持たせることで実現する
- たとえばトップレベルの 300 万件のレビューの中から、特定のレストランのレファレンスをもつレビュードキュメントを取得する時間的コストは、サブコレクションの場合と大して変わらない
- 当然「東京にある日本料理レストランのレビュースコア TOP 10」や「サービス全体の最新のレビュー 10 件」を取得するクエリは容易に発行できる

```json
"<reviews（トップレベルコレクション）>": {
  "<review ID（ドキュメント）>": {
    "<review ID（ドキュメント）>": {
      "restaurantRef": "/restaurants/{restaurant ID}",
      "name": "大谷翔平",
      "score": 5,
      "content": "ランチの焼き魚定食が絶品です！"
    },
    // ...
  }
}
```

---

## サブコレクション vs. トップレベルコレクション

- CollectionGroup クエリが実装される前は、トップレベルコレクションに実現できるクエリの観点で優位性があった
- セキュリティルールの記述については、サブコレクションに分がある
- "How to Structure Your Data" の動画では「**どちらもあり得る**」という結論

---

## クライアントサイドジョインとサーバサイドジョイン

あるレストランの詳細画面に、そのレストランの最新のレビュー 10 件が表示されているとすると...

- 元のレストランドキュメント 1 個に加えて、レビュードキュメントを 10 個読み取る必要がある
- RDB のサーバサイドで行う結合は Cloud Firestore にはないので、**クライアントサイドジョイン**を活用していく
- Cloud Firestore のクエリは十分高速だとされているが、いわゆる **N + 1 問題** のような状況が生じることになる（必ずしも深刻なことではない）

SQL で結合

```sql
SELECT * FROM review
INNER JOIN restaurant ON review.restaurant_id = restaurant.id;
WHERE review.restaurant_id = "{レストランの ID}"
```

Django で `select_related`

```python
reviews = Review.objects.all().select_related('restaurant')
for review in reviews:
  print(f'レビューのタイトル：{review.title}, レストラン名：{review.restaurant.name}')
```

---

## 非正規化の活用を検討する

- **あえて重複したデータを持たせる**ことで、レファレンス型のリレーションによる読み取り回数を抑える
- 非正規化して冗長に持たせるデータは、レビューの全情報でなくても良い
- **アプリケーションのユースケースや、アプリケーションが提供したい体験から判断する**
- 一括更新は Cloud Functions のバックグラウンド関数で実現できる

```json
"<restaurant ID（ドキュメント）>": {
  "name": "割烹メジャーリーグ",
  "address": "東京都新宿区1-2-3",
  "category": "日本料理",
  "averageScore": 4.7,
  "reviews": [
    {
      "reviewRef": "/reviews/{review ID}",
      "name": "大谷翔平",
      "title": "ランチの焼き魚定食が絶品です！",
      "score": 5
    },
    // ...
  ] 
}
```

---

## アクセス権限のような情報を保存する

「他のユーザーと何かを共同編集する」ようなアプリでは、誰がそれを閲覧・編集できるかという権限情報を保存しておく必要があるが、Cloud Firestore ではどうする？

- レストランのレビューアプリで言えば、レストランの管理者ユーザーが、同僚・部下ユーザーの何人かに限定して、レストラン情報の編集を許可するようなユースケース
- セキュリティルールによって、特定のドキュメントにアクセスできるユーザーを制御できる
- Array や Map で保存する？

```json
"<restaurant ID（ドキュメント）>": {
  "name": "割烹メジャーリーグ",
  "address": "東京都新宿区1-2-3",
  "category": "日本料理",
  // ...
  "role": {
    "{user-1 ID}": "オーナー",
    "{user-2 ID}": "編集者",
    "{user-3 ID}": "編集者",
  }
}
```

---

## データの機密レベルとUser ID の漏洩

権限情報をレストランドキュメントに直接保存するとすれば、通常のユーザーがそのレストラン名や住所を取得するときに、権限情報の Array や Map も一緒に取得されてしまう

- つまり、全ユーザーにそれらのレストランユーザーの User ID が漏洩するので、ちょっと気持ち悪い
- User ID はそのアプリ上でしか使わないユニークなもので、User ID から個人を特定するような心配は**あまりない**
- セキュリティルールをフィールドごとに設定することは不可能 →「**機密レベルの異なるデータを同一ドキュメントに保存してはならないという原則**」を守る<br>例：ユーザーの表示名と、メールアドレスや住所を同じドキュメントに保存してはならない

```json
"<restaurant ID（ドキュメント）>": {
  // ... 全ユーザーに公開するフィールド
  "<privateData（サブコレクション）>": {
    "roles": {
      "{user-1 ID}": "オーナー",
      "{user-2 ID}": "編集者",
      "{user-3 ID}": "編集者",
    },
    // ... アクセス権限を絞るべきその他の秘匿情報
  }
}
```

---

## お気に入りのレストランを保存する

ユーザーがお気に入りのレストランを保存しておく機能はどう実現するか？

- あるユーザーは複数のレストランをお気に入りに登録することができる
- あるレストランは複数のユーザーからお気に入りにされることがある

いわゆる N:N のリレーションシップ。

Cloud Firestore のような NoSQL のデータベースでは度々問題になることがある。

---

## お気に入りのレストランを保存する

候補 1：ユーザードキュメントの Array フィールドに、お気に入りのレストラン ID を保存する

- お気に入りの登録や解除の書き込みは簡単
- ユーザー情報を取得しておけば、レストラン一覧画面で、お気に入りに登録済みのレストランに★マークをつけておくようなユースケースも満たせる
- 「私のお気に入りレストラン一覧」を表示するのに、お気に入りのレストランの数だけ、レストラン ID を取得するクエリを発行する必要がある（クライアントサイドジョイン, N + 1 問題）
- または Cloud Functions の Callable functions を使う？


```json
"<users（コレクション）>": {
  "<user ID（ドキュメント）>": {
    // ...
    "favoriteRestaurantRefs": [
      "/restaurants/{restaurant-1 ID}",
      "/restaurants/{restaurant-2 ID}",
      "/restaurants/{restaurant-3 ID}"
    ]
  }
}
```

---

## お気に入りのレストランを保存する

候補 2：ユーザードキュメントに「私のお気に入りレストラン一覧」を表示するのに必要なデータを予め持たせておく

- 「私のお気に入りのレストラン一覧」を表示するために必要な情報を非正規化データとして Map で保持しておく（全情報でなくて良い）
- 個別のレストランを詳しく見たくなったら、レファレンスから個別のレストランドキュメントを取得すれば良い

```json
"<user ID>（ドキュメント）": {
  // ...
  "favoriteRestaurants": {
    "<restaurant-1 ID（ドキュメント）>": {
      "restaurantRef": "/restaurants/{restaurant-1 ID}",
      "name": "割烹メジャーリーグ",
      "address": "東京都新宿区1-2-3",
      "category": "日本料理",
    },
    // ...
  }
}
```

---

## お気に入りのレストランを保存する

候補 2：ユーザードキュメントに「私のお気に入りレストラン一覧」を表示するのに必要なデータを予め持たせておく

- レストランのカテゴリーが、いつの間にか日本料理からハンバーガー屋さんになっていたらどうする？
- レストランドキュメントの変更を検知して Cloud Functions で一括更新する
- 1 MB のドキュメントサイズやフィールド数 20,000 までの上限があるので、一人のユーザーがお気に入りに登録しておけるレストランの数に上限を設けておくと良いかもしれない

```ts {5-8}
export const onUpdateRestaurant = functions.region('asia-northeast')
    .firestore.document('restaurants/{restaurantID}')
    .onUpdate(async (change) => {
        const restaurant = change.after
        const users = await admin.firestore()
                      .collection('users')
                      .where('favoriteRestaurants.restaurant_1', '>', '')
                      .get()
        for (const user of users.docs) {
            batch.update(user.ref, {
                'favoriteRestaurants.restaurant_1.category': restaurant.get('category'),
            })
        }
        await batch.commit()
    })
```

---

## お気に入りのレストランを保存する

候補 3：トップレベルコレクションに全ユーザーのお気に入りレストランを保存する

- ユーザー、レストランに対する参照に加え、非正規化データとして「私のお気に入りのレストラン一覧」の画面に必要な情報を持たせておく
- あるユーザーで絞り込んで「私のお気に入りのレストラン一覧」を取得するのも容易
- あるレストランで絞り込んで「こんな人がこのレストランをお気に入りにしました」といったクエリも容易
- レストランのカテゴリーが変わった際にも、非正規化データの一括更新は Cloud Functions のバックグラウンド関数で対応可能

```json
"<favoriteRestaurants（トップレベルコレクション）>": {
  "<favorite-1 ID（ドキュメント）>": {
    "userRef": "/users/{userID}",
    "restaurantRef": "/restaurants/{restaurantID}",
    "name": "割烹メジャーリーグ",
    "category": "日本料理",
    "address": "東京都新宿区 1-2-3"
  },
  // ...
}
```

---

## まとめ

LT を聴く前よりも、次のようなキーワードに対する知識や感覚は磨かれましたか？

- サブコレクション vs. トップレベルコレクション
- 読み込み回数が少なくなる設計、クエリがシンプルになる設計、セキュリティルールを書きやすい設計
- Cloud Firestore でリレーションを表現する方法（Array や Map、サブコレクション、レファレンス）
- クライアントサイドジョイン、非正規化の活用、N + 1 問題
- 機密レベルの異なるフィールドを混在させない原則、User ID の漏洩
- N 対 N のリレーション、Cloud Functions のバックグラウンド関数


アプリケーションのユースケースや、保守管理、開発者が許容できること・できないことなどを考慮して、Cloud Firestore のより良いデータモデリングを考えていきましょう 🧑‍💻

---

素敵な機会をありがとうございました！

- [GitHub (@KosukeSaigusa)](https://github.com/KosukeSaigusa)
- [Twitter (@KosukeSaigusa)](https://twitter.com/kosukesaigusa)
- [Firebase 公式動画から『Firestore の DB 設計の基礎』を学ぶ](https://qiita.com/KosukeSaigusa/items/860b5a2a6a02331d07cb)
- [Firestore Security Rules の書き方と守るべき原則](https://qiita.com/KosukeSaigusa/items/18217958c581eac9b245)
