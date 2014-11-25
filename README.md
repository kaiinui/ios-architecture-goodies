Practical iOS Architecturing
============================

1. [Layering](#layering)
  1. [View Controller](#view-controller)
  2. [Model](#model)
    1. [Entity](#entity)
    2. [Interface](#interface)
      * [Wrapper Object](#wrapper-object)
  3. [Service](#service)
    1. [API Service](#api-service)
      * [Response Object](#response-object)
2. [Testing](#testing)
3. [Modules](#modules)
  1. [R](#r)
  2. [Measurement Events](#measurement-events)
  3. [Event Aggregator](#event-aggregator)
  4. [Wireframe](#wireframe)
  5. [Notification Observers](#notification-observer)
  6. [Action](#action)
  7. [Repository](#repository)
  8. [Access Policy](#access-policy)
  9. [Theme](#theme)
  10. [Flag](#flag)
  11. [Permission Manager](#permission-manager)
4. [Miscs](#miscs)
  1. [Router](#router)
  2. [Version](#version)
5. [Others](#others)
  1. [Delivery](#delivery)
    1. [deliver](#deliver)
  2. [Crash](#crash)
    1. [Crashlytics](#crashlytics)

Layering
========

View Controller
---------------

View Controller は iOS の中核となる要素です。View を表示し、ユーザにインタラクションを与えることを責務とします。

Model
-----

Model はアプリケーションのモデル層（データの取り扱い / 保持、ビジネスロジック）を責務とします。

### Entity

データそのものです。Objective-C の場合は NSManagedObject のサブクラスとなることが多いです。

### Interface

Interface は Entity を View から扱うために、Entity 操作の複雑さを隠蔽します。
典型的なタスクは、例えば特定種別の Entity を全て取得したり、特定の Entity を更新したり、Entity を使った何らかの操作を担当します。

このクラスは非常に重要で、View Controller にとって、アプリケーションの操作を全て行ってくれるライブラリのように機能します。
ですので、View Controller は何も考えずに Interface から Entity を取得したり、Interface を通じて操作を行う、といったことが可能になります。

また、更にラップしたい場合は Action を使います。

#### Wrapper Object

Wrapper Object は Entity をラップした Object です。Entity をコピーしたようなオブジェクトとして振る舞い、Entity と
殆ど同じプロパティを持ちます。
特に NSManagedObject の場合、View で Entity を直接扱うと (a) プロパティを書き換えるだけで、変更が適用されてしまう。 (b) スキーマの変更に弱くなる (c) 数値系が NSNumber となってしまう といった欠点があります。

ですので、Interface は Entity をそのまま返すのではなく、Entity を取得して、これらを Wrapper Object に変換してから返します。

どちらにせよ Entity (NSManagedObject) をそのまま View に露出することは避けたいです。Entity は大事に取り扱いましょう。

Service
---

Service はアプリケーションが生きている間、ずっと機能するクラスです。通常、Singleton かまたは Service Locator パターンを用いて永続化されます。

### API Service

API Service はバックエンドの API と通信し、Response Object を返却するサービスです。

このサービスは、Model 層にとって、ライブラリのように機能します。
API 通信系はバックエンドのスキーマと密接に動作しますので、Model 層からは出来るだけ切り離し、独立にバージョン管理出来るようになっていることが望ましいです。

#### Respone Object

Response Object は API から受け取ったレスポンスをモデルのスキーマにマッピングして、保持するオブジェクトです。
通常、Response Object は Entity と同じようなプロパティを持ちます。

なぜ Entity と Response Object を別途扱うかというと、前述したように、Model 層と、API と連携する部分は切り離してバージョン管理出来るようになるべきだからです。

Response Object として Entity をそのまま用いてしまうと、バックエンドのスキーマ変更に対しての対応が難しくなります。
NSManagedObject はプロパティを一つ加えるだけでもマイグレーションが必要になります。フロントエンドのマイグレーションは出来るだけやらないほうが良いので、Entity は丁寧に扱うようにしてください。

JSON から Response Object, Response Object から Entity への変換は、Mantle などを使うとよいでしょう。

Testing
===

- XCTest
- XCTestExpectation
- OCMock
- OHHTTPStubs
- KIF

Module
===

Layering では大雑把に iOS アプリケーションをレイヤー分けするアイデアだけを説明しました。

ロジックを View Controller から切り離すための部品として、更に次のようなモジュールがあります。アプリケーションの規模に対して、適切に使い分けると良いでしょう。

### R

R は定数を提供するクラスです。R はアプリケーション中のラベルや、数値パラメータ、CGSize や文字列定数等を提供します。
アプリケーションは R のクラスメソッドから定数を取得するようにします。

Header での定数宣言ではなくクラスメソッドのほうが良い理由は以下の通りです。

1. 任意のコードを呼べる（NSLocalizedString!!）
2. 引数を取れる（パラメータ付きのラベルなど）
3. OCMock で Stub 出来る

NSLocalizedString は特に R を介して呼び出すことが推奨されます。これは、特にテストのためです。R の返り値が NSLocalizedString のキーそのままでないことをテストすれば、用意に翻訳忘れをテストすることが出来ます。
補完が効くのもポイントです。

引数をとれることもキーで、例えば文章中の数値が可変な定数を作ることが簡単に出来ます。（e.g.「あなたの点数は %POINT です。」）
引数に対してちゃんとした文章が返っていることも、R を介していればテストは用意です。

NSFormatter などでの Formatting もここで行うのが良いです。（e.g. 「空き容量は %VOLUME です。」 => 「空き容量は 45kB です。」）
これはもはや定数ではないですが、大した責務膨張ではありませんので、私は R から必ず取れるという統一感を優先しています。

### Measurement Events

Measurement Events は Analytics のためのイベント送信をテスタブルに、そして統一的に扱う為の仕組みです。
実体は NSNotification であり、次の性質を持ちます。

1. 全て同じ Notification Name を使う
2. userInfo の "event_name" にイベントの名前が NSString として格納されている
3. userInfo の "event_args" にイベントのパラメータが NSDictionary として格納されている

イベントをトラッキングするためには、まずこの NSNotification をイベントを計測したい箇所で post します。この NSNotification は全て同じ name で post されているので、一カ所で統一的に受信を行うことが出来ます。userInfo を適切にパースし、実際の Tracking のコードを呼び出します。

この方式の利点は次の通りです。

1. NSNotification を post していることをテストするのが簡単。Expecta の `.notify()` や OCMock の `observerMock` を用いればすぐにテスト出来ます。
2. Event を受信して実際に Tracking のコードを呼び出す部分で、Event の取捨選択が出来る。従って、実際のアプリケーション中では Event を好きなだけ発行して、後から取捨選択して実際に送信する、といったバッファを作り出すことが出来ます。
3. 実際の Tracking のコードを知らなくても、各モジュールで計測のための Event を発行することが出来ます。

Tracking のコードをアプリケーションの各所で呼び出すと、統一的なテストが非常に難しいですから、Measurement Events は非常に効果的な仕組みです。

また、これらの Event を受信し状態を保持するクラスを作ることで、ユーザの行動によって振る舞いを返る為のフラグマネージャを容易に実装することが出来ます。

### Event Aggregator

Event Aggregator は前述の Measurement Events を受け取り、取捨選択して実際にトラッキングを行います。

AQSEventAggregator

### Wireframe

Wireframe は別名 View Controller Factory です。
全ての View Controller は Wireframe を介してインスタンスを取得するようにします。

1. View Controller には何らかの引数をプロパティを経由して呼び出すことが多いです。ですが、これを View Controller や Router 内で行うとプロパティの渡し忘れが起こりやすく、バグの温床となります。
2. Storyboard からのインスタンス化では、Storyboard 名を String で渡す必要があり、これもバグの温床となりがちです。

また、UIActivityViewController なども Wireframe から取得するようにすると尚良いです。

### Notification Observers

NSNotification は非常に素晴らしい仕組みです。適切に使えば非常に強力です。
しかしながら、NSNotification にはいくつかの欠点があります。

1. selector がリファクタリングの対象でない
2. dealloc にて `- removeObserver` を忘れるとメモリリークする
3. userInfo の取り扱いが、Key-Value ベースである。ゆえに、(a)型安全ではない (b)キーを間違えると nil が返る

これらを解決するモジュールが、Notification Observers です。

Notification Observer は次を行います。

1. Observer は Block を渡してインスタンス化されます
2. Observer は インスタンス化されると、特定の Notification の監視を始めます (- addObserver:)
3. Observer は Notification の受信時に、インスタンス化の際に渡された Block を発火します。
4. Observer は Notification の受信時に、userInfo を適切に型付けして Block に渡します。
5. Observer は dealloc 時に `- removeObserver:` を行います

### Action

View Controller を更にラップしたいときに使います。

一つの View Controller に対して一つの Action を用意して、ヘルパー的に用います。
Interface でラップされているだけで十分なケースも多いので、規模により、用意するかどうかを使い分けます。

### Repository

データの保持、検索を担当します。NSManagedObject の保持やクエリはここに責務を持たせます。

### Access Policy

ユーザの状態（例えば、一般、プレミアム）による機能の制限や開放を管理するクラスです。

### Theme

Theme はアプリケーションの見た目を管理します。

### Flag

ユーザの行動に対して、アプリケーションの振る舞いを変えるのはよくあるケースです。（チュートリアル等）

Flag はユーザの行動を監視し、行動したかどうかのフラグを管理します。
Measurement Events を転用することもあります。

### Permission Manager

iOS はユーザのリソース（写真、カメラ、アドレス帳等）にアクセスする際に承認ダイアログが出ます。ここで、拒否をされると一切ユーザのリソースにはアクセス出来ません。

この Permission を扱うのが Permission Manager です。Permission Manager は (a) 承認状態を返すメソッド (b) 承認ダイアログを出すメソッド を提供します。

写真や位置情報へのアクセスはよく使いますので、活躍するモジュールです。

Miscs
===

頻度は低いですが、便利な小物モジュール郡です。

### Router

Router は URL Scheme をルーティングするモジュールです。
URL Scheme をハンドリングするとき使います

JLRoutes

### Version

アプリケーションのバージョンや、iOS のバージョン等を返すクラスです。これらは、クラスメソッドで返されます。
また、デバッグ用に setter を持ちます。つまり、iOS のバージョン等を意図的に書き換え、振る舞いを観察することが出来ます。

バージョンを直接触らないメリットは、もちろんテスタビリティのためです。

互換性のためにバージョンにより振る舞いを返ることはよくあるケースなので、便利なモジュールです。

AQSVersion

Others
---

アーキテクチャとは直接関係がありませんが、有用な Tips です

### Delivery

iTunes Connect へのデプロイは deliver が便利です。レポジトリ内で各種入力内容 / スクリーンショットを管理することが出来ます。

#### deliver

[deliver](https://github.com/KrauseFx/deliver)

### Crash

Crash ログは Crashlytics を使いましょう。説明の必要がないほど良いです。

#### Crashlytics

[Crashlytics](https://crashlytics.com/)
