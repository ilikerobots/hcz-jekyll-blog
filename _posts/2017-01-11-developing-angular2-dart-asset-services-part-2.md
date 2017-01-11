---
layout: post
title:  "Developing Angular2 Dart Asset Services, Part 2"
date:   2017-01-11  02:00:00
categories: dart angular2 software 
---

_Continued from [Part 1]({% post_url 2016-12-26-developing-angular2-dart-asset-services %})_

# Developing Angular2 Dart Asset Services, Part 2

For the [fictive scenario]({% post_url 2016-12-26-developing-angular2-dart-asset-services %}#the-fictive-scenario) described in the previous article, we built a simple content service that would provide randomized *lorem ipsum* text to our article application. Now our manager tells us the latest news: the content will now be developed in-house by our company's ad department, who are non-techy.  The copywriters have agreed to author articles in Markdown and commit using GitHub's web interface, but that's the extent of their willingness to meet us halfway.

Since we planned ahead so nicely in [Part 1]({% post_url 2016-12-26-developing-angular2-dart-asset-services %}) by creating an injectable `ContentService`, this isn't a problem.  In fact, we need only write one new class and alter one line of our original app!  As for the content itself, we'll utilize Dart's powerful transformers to build a content repository that is authored in markdown but served in HTML; this with only a handful of markup code.

# Client-based Content Service

Recall from [Part 1]({% post_url 2016-12-26-developing-angular2-dart-asset-services %}) that we designed a very simple interface to define our content services behavior:

```dart
abstract class ContentService {
 Future<String> getContent(String id);
}
```

We begin the new network-based implementation by constructing a injectable skeleton service that implements the above interface:


```dart
@Injectable()
class ClientContentService implements ContentService {

  ClientContentService();

  @override
  Future<String> getContent(String id) async {
    //TODO: implement
  }
}
```

Our goal is that this implementation fetches HTML from an outside network resource.  We know there will be some base URI common to all articles. We'll complete the URL by simply tppending the article identifier and  ".html" extension.  As first attempt at implementing this, we declare a field to store the base URL to the content, and then update `getContent` to build the full URL using the article identifier and extension.

```dart
@Injectable()
class ClientContentService implements ContentService {
  final String _contentUrl = "http://localhost:8090/content/section";

  ClientContentService();

  @override
  Future<String> getContent(String id) async {
    final String url = path.join(_contentUrl, "$id.html");
    //TODO: fetch an article
  }
}
```

While this works, the hard-coded URI base makes our implementation brittle.  Our service will be much more flexible if we make this field variable.  One such way is to use Angular's dependency injection.  We rewrite `ClientContentService` constructor to inject the value of `_contentUrl`:


```dart
  final String _contentUrl;

  ClientContentService(@Inject(contentUrl) this._contentUrl);
```

Every injection needs a provider, so somewhere we must provision the `contentUrl` token.  We'll handle that later in the article.  For now, we finish our `ClientContentService` by implementing the details of network retrieval of HTML:

```dart
@Injectable()
class ClientContentService implements ContentService {

  final Client _http;
  final String _contentUrl;

  ClientContentService(@Inject(contentUrl) this._contentUrl, this._http);

  @override
  Future<String> getContent(String id) async {
    final String url = path.join(_contentUrl, "$id.html");
    try {
      final Response response = await _http.get(url);
      return response.body;
    } catch (e) {
      _handleError(url, e.runtimeType);
      return "<h3>Error</h3><p>Failed to locate content at $url</p>";
    }
  }

  void _handleError(String url, dynamic e) {
    //TODO: something useful with an error 
  }

}
```

The implementation utilizes `Client.get()` to fetch the network HTML resource, simply returning the response body.  Note we modified the constructor to inject a `Client` implementation.



# Providing the New Content Service

We've finished coding our new `ContentService`, but the app is still utilizing the original *lorem ipsum* implementation.   Recall from Part 1 our bootstrap:

```dart
  bootstrap(AppComponent, <Provider>[
    provide(ContentService, useClass: PlaceholderContentService)
  ]);
```

Modifying the bootstrap to use our new implementation is simple.  We need only swap the `PlaceholderContentService`  with our new  `ClientContentService`  and add the two new providers it requires: `Client` and `contentUrl`:

```dart
  bootstrap(AppComponent, <Provider>[
    provide(ContentService, useClass: ClientContentService),
    provide(Client, useFactory: () => new BrowserClient(), deps: <Object>[]),
    provide(contentUrl, useValue: "http://localhost:8090/content/article")
  ]);
```

That's it.  Without touching any code other than the bootstrap, our application is now ready to utilize the new network-client content service. When the app is run, the *loreum ipsum* content is gone and instead is shown a *404 Not Found* message.  This makes sense, as we're trying to fetch article content that hasn't yet been written.  Let's do something about that.  But first:


# A Note About Same Origin Policy and CORS

Our app, being run in a browser, is subject to [browser same-origin policy](https://en.wikipedia.org/wiki/Same-origin_policy) which places constraints on retrieval of HTML content. If our app and content are served from the same host and port, then we do not run afoul of the same-origin policy. However, if our content is served from another host or port, some allowance for Cross-Origin Resource Sharing (CORS) must be made.

In a production environment, possible solutions include a server configuration that proxies the third party content or inclusion of
CORS headers to authorize our app origin.  

During development, we have more options as we can afford to be more liberal.  If our content is served via `pub`, then the Access-Control-Allow-Origin header
is [already set to the wildcard](https://github.com/dart-lang/pub/issues/1215) and thus can be used from any origin.  If our content is served from another source, and we do not have the ability to adjust the CORS headers, we can temporarily disable security on our browser.  This can be done on Dartium with the flag `--disable-web-security`.  **NB! Disabling browser web security opens significant security risks and should be used only for development.  Use caution.**


# An External Content Repository

We've implemented everything we need to fetch article content remotely.  However, our ad-team needs a dedicated repository to author articles in Markdown.  We'll accomplish this simply by building a new Dart web project that is little more than a collection of markdown pages with a single transformer to convert these to HTML.  In fact, just such a [transformer already exists in Pub](https://pub.dartlang.org/packages/md_to_html).

Starting from a stagehanded simple Dart web project, we update to `pubspec.yaml` to include the `md_to_html` transformer:

```yaml
dependencies:
  md_to_html: ">=0.1.0 <0.2.0"
dev_dependencies:
  browser: '>=0.10.0 <0.11.0'
  dart_to_js_script_rewriter: '^1.0.1'
transformers:
- dart_to_js_script_rewriter
- md_to_html:
    template: "web/content/article/template.html"
```

Per the `md_to_html` documentation, we've added a Mustache template `template.html` which informs the transformer how to construct the HTML.  Ours is trivial , consisting only of the single line:

{% raw %}
```
{{content}}
```
{% endraw %}

That's it.  Really.  Our ad team can now author markdown files in the `web/content/article` directory. We provide an example with `fuzzy.md`:

```markdown 
Kittens are **fuzzy** *wuzzy*.
```

When we use pub to serve this project (e.g. ```pub serve --port=8090```), the `md_to_html` transformer consumes the markdown files and outputs HTML files in their stead.   When we reload the original app in our browser, we now see the Fuzzy article has our new content, retrieved from our article content repository.


![Screenshot of kitten article](/assets/developing-angular2-dart-asset-services-part-2/screenshot_kittens_fuzzy.png)

When all the articles are finalized, `pub build` will likewise produce HTML and it would only remain for our system administrator to host these pages somewhere our app can reach.

# Conclusion

Angular Dart's dependency injection makes building and using alternative service implementations fast and fun.  With reasonable planning and just a few lines of code, we've easily adapted to non-trivial requirements changes.  

In Part 3, we'll build a more elaborate asset provider and content repository for photographic images, allowing us to externalize not only the assets themselves but also license and attribution concerns.


# Source

The [full source code of this example is available on GitHub](https://github.com/ilikerobots/angular2-dart_asset_service_example/tree/content_service_client).


