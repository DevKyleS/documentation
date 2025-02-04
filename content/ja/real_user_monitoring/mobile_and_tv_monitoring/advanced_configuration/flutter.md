---
aliases:
- /ja/real_user_monitoring/flutter/advanced_configuration
- /ja/real_user_monitoring/otel
- /ja/real_user_monitoring/mobile_and_tv_monitoring/advanced_configuration/otel
- /ja/real_user_monitoring/mobile_and_tv_monitoring/setup/otel
- /ja/real_user_monitoring/flutter/otel_support/
code_lang: flutter
code_lang_weight: 30
description: Flutter Monitoring の構成について説明します。
further_reading:
- link: https://github.com/DataDog/dd-sdk-flutter
  tag: ソースコード
  text: dd-sdk-flutter のソースコード
- link: real_user_monitoring/explorer/
  tag: ドキュメント
  text: RUM データの調査方法
- link: https://www.datadoghq.com/blog/monitor-flutter-application-performance-with-mobile-rum/
  tag: ブログ
  text: Datadog Mobile RUM による Flutter アプリケーションのパフォーマンス監視
title: RUM Flutter の高度なコンフィギュレーション
type: multi-code-lang
---
## 概要

If you have not set up the Datadog Flutter SDK for RUM yet, follow the [in-app setup instructions][1] or refer to the [RUM Flutter setup documentation][2]. Learn how to set up [OpenTelemetry with RUM Flutter](#opentelemetry-setup).

## Automatic View Tracking

If you are using Flutter Navigator v2.0, your setup for automatic view tracking differs depending on your routing middleware. Here, we document how to integrate with the most popular routing packages.

### go_router

Since [go_router][8], uses the same observer interface as Flutter Navigator v1, so the `DatadogNavigationObserver` can be added to other observers as a parameter to `GoRouter`.

```dart
final _router = GoRouter(
  routes: [
    // ルート情報をここに
  ],
  observers: [
    DatadogNavigationObserver(datadogSdk: DatadogSdk.instance),
  ],
);
MaterialApp.router(
  routerConfig: _router,
  // 残りのセットアップ
);
```

ShellRoutes を使用している場合は、以下のようにそれぞれの `ShellRoute` に個別のオブザーバーを指定する必要があります。詳しくは[このバグ][11]を参照してください。

```dart
final _router = GoRouter(
  routes: [
    ShellRoute(build: shellBuilder),
    routes: [
      // 追加ルート
    ],
    observers: [
      DatadogNavigationObserver(datadogSdk: DatadogSdk.instance),
    ],
  ],
  observers: [
    DatadogNavigationObserver(datadogSdk: DatadogSdk.instance),
  ],
);
MaterialApp.router(
  routerConfig: _router,
  // 残りのセットアップ
);
```

Additionally, if you are using `GoRoute`'s `pageBuilder` parameter over its `builder` parameter, ensure that you are passing on the `state.pageKey` value and the `name` value to your `MaterialPage`.

```dart
GoRoute(
  name: 'My Home',
  path: '/path',
  pageBuilder: (context, state) {
    return MaterialPage(
      key: state.pageKey,       // Necessary for GoRouter to call Observers
      name: name,               // Needed for Datadog to get the right route name
      child: _buildContent(),
    );
  },
),
```

### AutoRoute

[AutoRoute][9] は `config` メソッドの一部として `navigatorObservers` の 1 つとして提供された `DatadogNavigationObserver` を使用することができます。

```dart
return MaterialApp.router(
  routerConfig: _router.config(
    navigatorObservers: () => [
      DatadogNavigationObserver(
        datadogSdk: DatadogSdk.instance,
      ),
    ],
  ),
  // 残りのセットアップ
);
```

ただし、AutoRoute のタブルーティングを使用する場合は、Datadog のデフォルトオブザーバーを AutoRoute の `AutoRouteObserver` インターフェイスで拡張する必要があります。

```dart
class DatadogAutoRouteObserver extends DatadogNavigationObserver
    implements AutoRouterObserver {
  DatadogAutoRouteObserver({required super.datadogSdk});

  // オブザーバータブルートへのオーバーライドのみ
  @override
  void didInitTabRoute(TabPageRoute route, TabPageRoute? previousRoute) {
    datadogSdk.rum?.startView(route.path, route.name);
  }

  @override
  void didChangeTabRoute(TabPageRoute route, TabPageRoute previousRoute) {
    datadogSdk.rum?.startView(route.path, route.name);
  }
}
```

この新しいオブジェクトは、よりシンプルな `DatadogNavigationObserver` に代わって AutoRoute の構成を作成します。

### Beamer

[Beamer][10] では、`BeamerDelegate` の引数として `DatadogNavigationObserver` を使用することができます。

```dart
final routerDelegate = BeamerDelegate(
  locationBuilder: RoutesLocationBuilder(
    routes: {
      // ルート構成
    },
  ),
  navigatorObservers: [
    DatadogNavigationObserver(DatadogSdk.instance),
  ]
);
```

## ユーザーセッションの充実

Flutter RUM は、ユーザーアクティビティ、ビュー (`DatadogNavigationObserver` を使用)、エラー、ネイティブクラッシュ、ネットワークリクエスト (Datadog Tracking HTTP Client を使用) などの属性を自動的に追跡します。RUM イベントおよびデフォルト属性については、[RUM データ収集ドキュメント][3]をご参照ください。カスタムイベントを追跡することで、ユーザーセッション情報を充実させ、収集された属性をより細かく制御することが可能になります。

### 独自のパフォーマンスタイミングを追加

RUM のデフォルト属性に加えて、`DdRum.addTiming` を使用して、アプリケーションが時間を費やしている場所を測定できます。タイミング測定は、現在の RUM ビューの開始を基準にしています。

たとえば、ヒーロー画像が表示されるまでにかかる時間を計ることができます。

```dart
void _onHeroImageLoaded() {
    DatadogSdk.instance.rum?.addTiming("hero_image");
}
```

一度設定したタイミングは `@view.custom_timings.<timing_name>` としてアクセス可能です。例えば、`@view.custom_timings.hero_image` のようになります。

ダッシュボードで視覚化を作成するには、まず[メジャーの作成][4]を行います。

### ユーザーアクションの追跡

`DdRum.addAction` を使用すると、タップ、クリック、スクロールなどの特定のユーザーアクションを追跡することができます。

To manually register instantaneous RUM actions such as `RumActionType.tap`, use `DdRum.addAction()`. For continuous RUM actions such as `RumActionType.scroll`, use `DdRum.startAction()` or `DdRum.stopAction()`.

例:

```dart
void _downloadResourceTapped(String resourceName) {
    DatadogSdk.instance.rum?.addAction(
        RumActionType.tap,
        resourceName,
    );
}
```

When using `DdRum.startAction` and `DdRum.stopAction`, the `type` action must be the same for the Datadog Flutter SDK to match an action's start with its completion.

### カスタムリソースの追跡

[Datadog Tracking HTTP Client][5] を使用して自動的にリソースを追跡するほか、[以下の方法][6]を使用して、ネットワークリクエストやサードパーティプロバイダ API など特定のカスタムリソースを追跡することが可能です。

- `DdRum.startResource`
- `DdRum.stopResource`
- `DdRum.stopResourceWithError`
- `DdRum.stopResourceWithErrorInfo`

例:

```dart
// in your network client:

DatadogSdk.instance.rum?.startResource(
    "resource-key",
    RumHttpMethod.get,
    url,
);

// Later

DatadogSdk.instance.rum?.stopResource(
    "resource-key",
    200,
    RumResourceType.image
);
```

Flutter Datadog SDK がリソースの開始と完了を一致させるために、両方の呼び出しで `resourceKey` に使用される `String` は、呼び出すリソースに対して一意である必要があります。

### カスタムエラーの追跡

特定のエラーを追跡するには、エラーが発生したときにメッセージ、ソース、例外、追加属性で `DdRum` に通知します。

```dart
DatadogSdk.instance.rum?.addError("This is an error message.");
```

## カスタムグローバル属性の追跡

Datadog Flutter SDK が自動的に取得する[デフォルトの RUM 属性][3]に加えて、RUM イベントにカスタム属性などのコンテキスト情報を追加して、Datadog 内の観測可能性を高めることができます。

カスタム属性を使用すると、観察されたユーザーの行動に関する情報 (カート値、マーチャント層、広告キャンペーンなど) をコードレベルの情報 (バックエンドサービス、セッションタイムライン、エラーログ、ネットワークヘルスなど) でフィルタリングおよびグループ化することができます。

### カスタムグローバル属性の設定

カスタムグローバル属性を設定するには、`DdRum.addAttribute` を使用します。

* 属性を追加または更新するには、`DdRum.addAttribute` を使用します。
* キーを削除するには、`DdRum.removeAttribute` を使用します。

### ユーザーセッションの追跡

RUM セッションにユーザー情報を追加すると、次のことが簡単になります。

* 特定のユーザーのジャーニーをたどる
* エラーの影響を最も受けているユーザーを把握する
* 最も重要なユーザーのパフォーマンスを監視する

{{< img src="real_user_monitoring/browser/advanced_configuration/user-api.png" alt="RUM UI のユーザー API" style="width:90%" >}}

次の属性は**オプション**ですが、**少なくとも 1 つ**を指定します。

| 属性 | タイプ   | 説明                                                                                              |
|-----------|--------|----------------------------------------------------------------------------------------------------------|
| `usr.id`    | 文字列 | 一意のユーザー識別子。                                                                                  |
| `usr.name`  | 文字列 | RUM UI にデフォルトで表示されるユーザーフレンドリーな名前。                                                  |
| `usr.email` | 文字列 | ユーザー名が存在しない場合に RUM UI に表示されるユーザーのメール。Gravatar をフェッチするためにも使用されます。 |

ユーザーセッションを識別するには、`DatadogSdk.setUserInfo` を使用します。

例:

```dart
DatadogSdk.instance.setUserInfo("1234", "John Doe", "john@doe.com");
```

## RUM イベントの変更または削除

**注**: この機能は、Flutter で構築された Web アプリケーションではまだ利用できません。

Datadog に送信される前に RUM イベントの属性を変更したり、イベントを完全に削除したりするには、Flutter RUM SDK を構成するときに Event Mappers API を使用します。

```dart
final config = DatadogConfiguration(
    // other configuration...
    rumConfiguration: DatadogRumConfiguration(
        applicationId: '<YOUR_APPLICATION_ID>',
        rumViewEventMapper = (event) => event,
        rumActionEventMapper = (event) => event,
        rumResourceEventMapper = (event) => event,
        rumErrorEventMapper = (event) => event,
        rumLongTaskEventMapper = (event) => event,
    ),
);
```

各マッパーは `(T) -> T?` というシグネチャを持つ関数で、 `T` は具象的な RUM イベントの型です。これは、送信される前にイベントの一部を変更したり、イベントを完全に削除したりすることができます。

例えば、RUM Resource の `url` に含まれる機密情報をリダクティングするには、カスタム `redacted` 関数を実装して、`rumResourceEventMapper` で使用します。

```dart
    rumResourceEventMapper = (event) {
        var resourceEvent = resourceEvent
        resourceEvent.resource.url = redacted(resourceEvent.resource.url)
        return resourceEvent
    }
```

エラー、リソース、アクションのマッパーから `null` を返すと、イベントは完全に削除され、Datadog に送信されません。ビューイベントマッパーから返される値は `null` であってはなりません。

イベントのタイプに応じて、一部の特定のプロパティのみを変更できます。

| イベントタイプ       | 属性キー                     | 説明                                   |
|------------------|-----------------------------------|-----------------------------------------------|
| RumViewEvent     | `viewEvent.view.url`              | ビューの URL。                              |
|                  | `viewEvent.view.referrer`         | ビューの参照元。                         |
| RumActionEvent   | `actionEvent.action.target?.name` | アクションの名前。                           |
|                  | `actionEvent.view.referrer`       | このアクションにリンクしているビューの参照元。   |
|                  | `actionEvent.view.url`            | このアクションにリンクされているビューの URL。        |
| RumErrorEvent    | `errorEvent.error.message`        | エラーメッセージ。                                |
|                  | `errorEvent.error.stack`          | エラーのスタックトレース。                      |
|                  | `errorEvent.error.resource?.url`  | エラーが参照するリソースの URL。      |
|                  | `errorEvent.view.referrer`        | このアクションにリンクしているビューの参照元。   |
|                  | `errorEvent.view.url`             | このエラーにリンクされているビューの URL。         |
| RumResourceEvent | `resourceEvent.resource.url`      | リソースの URL。                          |
|                  | `resourceEvent.view.referrer`     | このアクションにリンクしているビューの参照元。   |
|                  | `resourceEvent.view.url`          | このリソースにリンクされているビューの URL。      |

## Retrieve the RUM session ID

Retrieving the RUM session ID can be helpful for troubleshooting. For example, you can attach the session ID to support requests, emails, or bug reports so that your support team can later find the user session in Datadog.

You can access the RUM session ID at runtime without waiting for the `sessionStarted` event:

```dart
final sessionId = await DatadogSdk.instance.rum?.getCurrentSessionId()
```

## トラッキングの同意を設定（GDPR と CCPA の遵守）

データ保護とプライバシーポリシーに準拠するため、Flutter RUM SDK は初期化時に追跡に関する同意を求めます。

`trackingConsent` 設定は以下のいずれかの値で示されます。

1. `TrackingConsent.pending`: Flutter RUM SDK はデータの収集とバッチ処理を開始しますが、Datadog には送信しません。Flutter RUM SDK は新しい追跡に関する同意の値を待って、バッチされたデータをどうするかを決定します。
2. `TrackingConsent.granted`: Flutter RUM SDK はデータの収集を開始し、Datadog へ送信します。
3. `TrackingConsent.notGranted`: Flutter RUM SDK はデータを収集しません。ログ、トレース、RUM イベントなどが Datadog に送信されることはありません。

Flutter RUM SDK の初期化後に追跡同意値を変更するには、`DatadogSdk.setTrackingConsent` API 呼び出しを使用します。Flutter RUM SDK は、新しい値に応じて動作を変更します。

例えば、現在の追跡同意が `TrackingConsent.pending` で、その値を `TrackingConsent.granted` に変更すると、Flutter RUM SDK は以前に記録したデータと今後のデータをすべて Datadog に送ります。

同様に、値を `TrackingConsent.pending` から `TrackingConsent.notGranted` に変更すると、Flutter RUM SDK はすべてのデータを消去し、今後データを収集しないようにします。

## Flutter 固有のパフォーマンスメトリクス

Flutter 固有のパフォーマンスメトリクスの収集を有効にするには、`DatadogRumConfiguration` で `reportFlutterPerformance: true` を設定します。ウィジェットのビルド時間とラスター時間は[モバイルバイタル][17]に表示されます。

## OpenTelemetry setup

The [Datadog Tracking HTTP Client][12] package and [gRPC Interceptor][13] package both support distributed traces through both automatic header generation and header ingestion. This section describes how to use OpenTelemetry with RUM Flutter.

### Datadog のヘッダー生成

追跡クライアントや gRPC インターセプターを構成する際に、Datadog に生成させたい追跡ヘッダーの種類を指定することができます。例えば、`example.com` には `b3` ヘッダーを、`myapi.names` には `tracecontext` ヘッダーを送信したい場合、以下のコードで実現できます。

```dart
final hostHeaders = {
    'example.com': { TracingHeaderType.b3 },
    'myapi.names': { TracingHeaderType.tracecontext}
};
```

このオブジェクトは、初期構成時に使用することができます。

```dart
// For default Datadog HTTP tracing:
final configuration = DatadogConfiguration(
    // configuration
    firstPartyHostsWithTracingHeaders: hostHeaders,
);
```

その後、通常通りトレースを有効にすることができます。

This information is merged with any hosts set on `DatadogConfiguration.firstPartyHosts`. Hosts specified in `firstPartyHosts` generate Datadog Tracing Headers by default.

## 参考資料

{{< partial name="whats-next/whats-next.html" >}}

[1]: https://app.datadoghq.com/rum/application/create
[2]: /ja/real_user_monitoring/mobile_and_tv_monitoring/setup/flutter#setup
[3]: /ja/real_user_monitoring/mobile_and_tv_monitoring/data_collected/flutter
[4]: /ja/real_user_monitoring/explorer/?tab=measures#setup-facets-and-measures
[5]: https://github.com/DataDog/dd-sdk-flutter/tree/main/packages/datadog_tracking_http_client
[6]: https://pub.dev/documentation/datadog_flutter_plugin/latest/datadog_flutter_plugin/
[7]: https://pub.dev/documentation/datadog_flutter_plugin/latest/datadog_flutter_plugin/DatadogNavigationObserver-class.html
[8]: https://pub.dev/packages?q=go_router
[9]: https://pub.dev/packages/auto_route
[10]: https://pub.dev/packages/beamer
[11]: https://github.com/flutter/flutter/issues/112196
[12]: https://pub.dev/packages/datadog_tracking_http_client
[13]: https://pub.dev/packages/datadog_grpc_interceptor
[14]: https://github.com/openzipkin/b3-propagation#single-headers
[15]: https://github.com/openzipkin/b3-propagation#multiple-headers
[16]: https://www.w3.org/TR/trace-context/#tracestate-header
[17]: /ja/real_user_monitoring/mobile_and_tv_monitoring/mobile_vitals/?tab=flutter