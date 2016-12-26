---
layout: post
title:  "Developing Angular2 Dart Asset Services"
date:   2016-12-26  08:10:00
categories: dart angular2 software 
---

# Developing Angular2 Dart Asset Services, Part 1

Most web apps utilize digital assets to some degree.  These could be as simple as images in your repo and HTML in your templates.  However, with more complex apps, the reasons for these assets to be external multiply rapidly.  For example, an app may need to leverage CDNs, interface with third party providers, or conform to asset licensing restrictions. 

In this series of articles, I'll demonstrate how easily we can develop asset services in Angular2 Dart to provide images and content to an app.  With Angular2's dependency injection, we will  effortlessly swap in mocked content providers that accelerate development cycles by deferring the need for production assets.


# The Fictive Scenario

Our manager has asked us to develop a very simple promotional app that displays articles about kittens.  But there are two problems:

1. The articles haven't yet been written
2. Due to licensing reasons, we cannot include the kitten pictures in our source repository

Naturally, our manager wants the app ready Monday, but we don't want to work a weekend.

We can keep our manager happy and enjoy our weekend by abstracting the selection (and retrieval, if necessary) of assets into service components.  This will allow us to develop the application with placeholder content that effectively models eventual real content.  Later, when production assets are ready, we need only, at most, provide another implementation of these encapsulated components.

In this article, Part 1, I will illustrate the development of an HTML asset provider and its usage in our fictive kitten app.  Part 2 will build more robust HTML asset providers, and Part 3 will introduce an image asset provider.


# Preparation

We start with a [very simple Angular2 Dart app](https://github.com/ilikerobots/angular2-dart_asset_service_example/tree/prep) that displays fake articles.  The app displays several topics as navigation. When the user taps a section, a collection of articles is displayed pertinent to that topic.  The focal point of this app is `SiteStructureService` which defines the structure of sections and articles.

```dart
@Injectable()
class SiteStructureService {
  static final List<ArticleSection> structure = <ArticleSection>[
    new ArticleSection("About Kittens")
      ..articles.add(new Article("Fuzzy"))
      ..articles.add(new Article("Warm"))
      ..articles.add(new Article("Curious")),
    new ArticleSection("Anatomy")
      ..articles.add(new Article("Paws"))
      ..articles.add(new Article("Whiskers"))
      ..articles.add(new Article("Tail")),
  [...]
```

The rest of the app builds from this service, by establishing routing

```dart
 List<RouteDefinition> _getRouteConfig(List<ArticleSection> pages) {
    final List<RouteDefinition> config = <RouteDefinition>[];
    for (int i = 0; i < pages.length; i++) {
      config.add(
          new Route(path: "/" + pages[i].routeSlug,
              name: pages[i].routeName,
              component: pages[i].component,
              data: <String,dynamic>{'id': pages[i].name}));
    }
    return config;
  }

```

and laying out articles

```html
  <div *ngFor="let article of articles" class="content-item">
    <h2>{{article.name}}</h2>

    <div class="content-html">
      <div>Article content goes here</div>
    </div>
  </div>
```

resulting in a simple but working article reader app.

![Screenshot of simple kitten reader app](/assets/developing-angular2-dart-asset-services/Screenshot_kitten_articles.png)


There's obviously a bit more code, but that's not the focus of this article.  Clone the [complete source code of this starter app](https://github.com/ilikerobots/angular2-dart_asset_service_example/tree/prep) on GitHub if you'd like to follow along with the next steps.


# A Content Service
 
Our example app works nicely such as it is, but all our articles read *"Article content goes here"*. Our manager isn't buying it.  How can we be sure we'll be able to plug into our real articles when they're complete?  What will the site actually look like with real content?  What happens if there's a delay in loading the content?  We've got more work to do.

We start by defining the interface `ContentService` which will provide article HTML.  The goal is for the service to return article HTML when provided with an article identifier.  We keep this app very simple, so the interface accepts a `String` identifier and returns a `Future` for a `String` containing the appropriate HTML.  We utilize `Future` because we anticipate that fetching production HTML will be an async network call to an outside resource.


```dart
import 'dart:async';

abstract class ContentService {
 Future<String> getContent(String id);
}
```

Next, we create a very simple implementation of this service which utilizes the `lorem` Dart package to produce placeholder HTML.  Generating this content is a synchronous operation, so we randomize a delay to better mimic real-life behavior.

```dart
import 'dart:async';
import 'dart:math';
import 'package:lorem/lorem.dart';
import 'package:angular2/core.dart';
import 'package:angular2_dart_asset_service/src/asset/content/content_service.dart';

@Injectable()
class PlaceholderContentService implements ContentService {
  static const int maxDelay = 1500; //max simulated delay in milliseconds
  final Random _rnd = new Random();
  final Lorem lorem = new Lorem();

  @override
  Future<String> getContent(String id) async {
    final String content = await new Future<String>.delayed(
        new Duration(milliseconds: _rnd.nextInt(maxDelay)), () => _generateSampleContent());
    return content;
  }

  String _generateSampleContent() {
    final StringBuffer sb = new StringBuffer();
    do {
      sb.write(_rndSection(minPars: 2, maxPars: 5));
    } while (_rnd.nextDouble() < 0.65);

    return sb.toString();
  }

  String _rndSection({int headerLength: 5, int minPars: 2, int maxPars: 5}) {
    final StringBuffer sb = new StringBuffer();
    sb.writeln("<h2>${lorem.createSentence(sentenceLength: headerLength)}</h2>");
    sb.writeln("<p>${lorem.createParagraph(numSentences: minPars + _rnd.nextInt(maxPars - minPars))}</p>");
    return sb.toString();
  }

}
```

We must also ensure `lorem` is included in the `pubspec.yaml`. 

# Using the Content Service

We now have prepared a content service and a sample implementation, so lets use it.  Our first step is to use Angular dependency injection to provide this component and inject it where needed.  In `main.dart`, we update the bootstrap to provide our content service

```dart
  bootstrap(AppComponent, <Provider>[
    provide(ContentService, useClass: PlaceholderContentService)
  ]);
```

and then inject this service into `ArticleComponent`:

```dart
class ArticleComponent implements OnInit, CanReuse {
  String pageId;

  final SiteStructureService _struct;
  final ContentService _contentService;
  final RouteData _routeData;
  String flowDirection = "row";

  ArticleComponent(this._struct, this._contentService, this._routeData);
```

At this point, the content service is now available to `ArticleComponent`. Let's make some use of it.  First we update the `ngOnInit` method to fetch the content for each article in the currently displayed section:

```dart
  void ngOnInit() {
    pageId = _routeData.data['id'];
    articles.forEach(_getContent);
  }
```

For each article, we call `_getContent` in which we invoke the injected `ContentService` to store  the resolved content into a map `contents`:

```dart
  Map<String, String> contents = <String, String>{};

  ArticleComponent(this._struct, this._contentService, this._routeData);
 
  void _getContent(Article article) {
    _contentService.getContent(article.id).then((String content) {
      contents[article.id] = content; 
    });
  }
```

And then we update `ArticleComponent`'s HTML template to display the content from the `contents` map:

```html
  <div *ngFor="let article of articles" class="content-item">
    <h2>{{article.name}}</h2>

    <div class="content-html">
      <div>{{contents[article.id]}}</div>
    </div>
  </div>
```

That's it... almost. If we view the app now, we can see randomized placeholder HTML, but the HTML isn't rendered properly.   It's just printed as a text, HTML tags and all.  


![Screenshot of kitten reader app](/assets/developing-angular2-dart-asset-services/Screenshot_kittens_html_as_string.png)


## Safe Inner HTML

The article content is treated as simple text because we've simply interpolated the string in the template.  Instead, we need to consider article content as HTML which should be rendered to the client.  However, because doing this opens up the possibility of security vulnerabilities such as Cross Site Scripting (XSS), Angular2 rightfully makes us establish the "safeness" of any HTML we wish to render from outside sources.  In order to proceed, we need to explicitly declare that article contents can be considered safe.  We do so using the `safeInnerHtml` directive:

```html
      <div [safeInnerHtml]="contents[article.id]"></div>
```

which requires that we include this directive in `ArticleComponent`'s directives list:

```dart
    directives: const <dynamic>[SafeInnerHtmlDirective],
``` 

This directive requires usage of of `SafeHtml` rather than a simple `String`, meaning we must convert `String` content  into `SafeHtml`.  One such way is to utilize Angular's DOM sanitization service to designate our trust in the retrieved HTML. First we inject the sanitization service:

```dart
  final DomSanitizationService _trustService;

  ArticleComponent(this._struct,
      this._contentService,
      this._trustService,
      this._routeData);
```

Then we modify `ArticleComponent`'s `_getContent` method to declare this trust by simply bypassing all security.  Note that this denotes we have implicit trust in our provided HTML.  **If this is not the case, it's absolutely mandatory that we manage our security by proper sanitization**.

```dart
    _contentService.getContent(article.id).then((String content) {
      // we must implicitly trust this content.  If not, DON'T DO THIS
      contents[article.id] = _trustService.bypassSecurityTrustHtml(content); 
    });
```

And the article content map changes from containing strings to containing `SafeHtml`

```dart
  Map<String, SafeHtml> contents = <String, SafeHtml>{};
```

And now we're ready.  Out article HTML is rendered in its proper place.

![Screenshot of kitten reader app](/assets/developing-angular2-dart-asset-services/Screenshot_kittens_rendered_html.png)

We've now delegated provision of article HTML content to an abstracted injectable service.  But our placeholder content is too simple and doesn't exercise the app very well.  In Part 2, we'll implement much more capable `ContentService`s, demonstrating how easy Angular2 makes swapping alternative service implementations. 

# Source

The [full source code of this example is available on GitHub](https://github.com/ilikerobots/angular2-dart_asset_service_example/tree/content_service).


