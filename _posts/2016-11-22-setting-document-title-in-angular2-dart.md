---
layout: post
title:  "Setting Document Titles in Angular2 Dart"
date:   2016-11-21  20:51:00
categories: dart software 
---

Setting the document title in an Angular2 app isn't as straightforward as simply binding a property to the HTML `<title>` tag.   Since an Angular app lives within the `<body>` of a DOM, Angular2 has no way to bind properties into `<head>` where the title is located.

The solution is [documented nicely for Angular2 Typescript](https://angular.io/docs/ts/latest/cookbook/set-document-title.html), but the Dart documentation for the same is pending.  In this blog, I'll explain how to perform this basic task in Angular2 Dart.   If you're looking for the short answer, skip ahead to [Injecting the Title Service](#injecting-the-title-service) or straight to the [source code](#source).  

 So this post doesn't become entirely irrelevant once the Dart documentation is written, I will further illustrate one way in which title changes can be tied to routing changes with a start-to-finish example.

# Solution Overview

We know Javascript provides a direct way to change a doc title (e.g. ```document.title="No Sweat!"```) which we could emulate, but we would lose abstraction which could make it difficult to run the app in a non-browser environment someday.  Instead, Angular provides a service which provides an API for the rather unpretentious task of getting/setting a document title.  If we rely on this API, we can be sure that wherever the app runs, whatever represents a title will be accordingly updated when we want.


# Preparation

This demonstration is built on a basic bare-bones Angular2 app (generated from [stagehand](http://stagehand.pub/)).  Read the [quickstart](https://angular.io/docs/dart/latest/quickstart.html) if you need help getting up and running.

The stagehand template provides us a `pubspec.yaml` sufficient for our project and an AppComponent serving as starting point.

# A Simple App with Routing

Because I like baseball, this simple app will show baseball players, teams, and fields, using a separate component to display each.  Additionally, we allow users to tap a player name to view more detailed info about that player.

To begin, we create three very simple and nearly-identical components: `PlayersComponent`, `TeamsComponent`, `FieldsComponent`. Here's `FieldsComponent`:

```dart
import 'package:angular2/core.dart';

@Component(
    selector: 'my-fields',
    template: '<h2>Fields</h2>'
)
class FieldsComponent {
  FieldsComponent();
}
```

We want `AppComponent` to route users to these three components, so we update the main `AppComponent` to include a `RouteConfig` that provides easy navigation:

```dart
import 'package:angular2/core.dart';
import 'package:angular2/router.dart';
import 'package:angular_dart_page_titles_on_route/page/fields_component.dart';
import 'package:angular_dart_page_titles_on_route/page/players_component.dart';
import 'package:angular_dart_page_titles_on_route/page/teams_component.dart';

@Component(
    selector: 'my-app',
    styleUrls: const ['app_component.css'],
    templateUrl: 'app_component.html',
    directives: const [ROUTER_DIRECTIVES],
    providers: const [ROUTER_PROVIDERS],
    )
@RouteConfig(const [
  const Route(path: '/players', name: 'Players', component: PlayersComponent, useAsDefault: true),
  const Route(path: '/teams', name: 'Teams', component: TeamsComponent),
  const Route(path: '/fields', name: 'Fields', component: FieldsComponent),
])
class AppComponent {}
```

Now that the routes are paved, we update `AppComponent`'s HTML to show navigation links and an outlet for routed components:

```html
<nav>
  <ul>
    <li><a [routerLink]="['Players']">Players</a></li>
    <li><a [routerLink]="['Teams']">Teams</a></li>
    <li><a [routerLink]="['Fields']">Fields</a></li>
  </ul>
</nav>

<div class="content" id="content">
  <router-outlet></router-outlet>
</div>
```

If we run the project now, we have a simple menu with three items, which we can navigate amongst to show placeholder pages with headings. 

Let's update the app so that we can drill down to specific players.

First, we create a `PlayerDetailComponent` that will handle display of individual players.  

```dart
import 'package:angular2/core.dart';
import 'package:angular2/router.dart';

@Component(
    selector: 'my-player-detail',
    template: '''<h2>Player Detail</h2> <h3>{{player}}</h3>''',
)
class PlayerDetailComponent implements OnInit {
  final RouteParams _routeParams;
  String player = "Not selected";

  PlayerDetailComponent(this._routeParams);

  void ngOnInit() {
    player = _routeParams.get('id');
  }

}
```

Here we've anticipated we will retrieve the specific player we are interested in from the route parameters.  We use Angular's `RouteParams` service to obtain the identifier, and then set a property accordingly.   Our detail component doesn't actually provide much useful detail, of course, but this is just an illustration.

Again, we need to provide a route to the new `PlayerDetailComponent` by adding the following to `AppComponent`'s `RouteConfig`:

```dart
  const Route(path: '/player/:id', name: 'PlayerDetail', component: PlayerDetailComponent)
```

As we planned, this route expects that a player identifier is provided.  So let's update `PlayerComponent` to display a list of of players.  First we include a property containing a simple list of player names

```dart
List<String> players = ["John, Tommy", "Carey, Max", "Nehf, Art", "Brown, Mordecai"];
```

_Note: if you now what these ball players have in common, [let me know](https://twitter.com/ilikerobotz/); I'll be very impressed._ 


Then update the corresponding template to display this list, registering a click handler for each player shown.

```dart
    template: '''
    <h2>Players</h2>
    <ul>
      <li *ngFor="let player of players" (click)="onSelect(player)">{{player}}</li>
    </ul>''',
```

Finally, we implement the handler on `PlayerComponent` to navigate to the new detail route, using the clicked player name as our identifier parameter.

```dart
  void onSelect(String player) {
    _router.navigate(['/PlayerDetail', {'id': player}]);
  }
```

If we run the app now, our players page show a list of player names. Tapping any will show a placeholder detail page for that player.   

We now have an app that provides some simple routing.  We'd really like if the document titles updated when we routed to a new component!


_NB! We've cut a lot of corners with our app to keep our example simple!_  

# Injecting the Title Service

The Angular `Title` service provides a simple API for getting/setting the document title. To utilize the `Title` service, we'll need to inject it into any components that require it.  Usually a service intended to be used commonly throughout the application is best registered in the parent `AppComponent`, e.g.

```dart
import 'package:angular2/platform/browser.dart' show Title;
[...]
const Provider(Title, useClass: Title) // not the best approach in this case
```

However, the import of platform/browser implies that `AppComponent` is aware it is running in the browser.  But if we knew that already, we may as well have set the document title directly via the DOM!  We want our individual components to be platform-agnostic, so it's better if we push the registration of the `Title` service into the Angular bootstrap in `main.dart`.


In `main.dart`, we modify our simple bootstrap to additionally register our `Title` provider, which will be made available throughout our application:

```dart
  bootstrap(AppComponent, [
    provide(Title, useFactory: () => new Title(), deps: const[])
  ]);
```
Here we've registered the `Title` token with a factory constructor that simply instantiates a new `Title` object. 

We can now inject this service into all of our components by altering our component constructors to require it.  Here's an updated `FieldsComponent` that accepts the injected `Title` service and assigns it to a field. 

```dart
import 'package:angular2/core.dart';
import 'package:angular2/router.dart';
import 'package:angular2/platform/browser.dart' show Title;

@Component(
    selector: 'my-fields',
    template: '<h2>Fields</h2>',
)
class FieldsComponent {
  final Title _titleService;

  FieldsComponent(this._titleService);

}
```
The remaining components follow the same pattern.

//FIXME: we still have to import platform/browser here, doesn't this violate our requirement that we are platform agnostic?

# Updating Titles on Route Changes

Now that we've made our `Title` service available to our components, updating the title is straightforward.  We'd like our three components `FieldsComponent`, `PlayersComponent`, and `TeamsComponent` to update the document title whenever they are activated from a route event.  Here's the updated `FieldsComponent` doing exactly that:

```dart
class FieldsComponent implements OnActivate {
  final Title _titleService;
  final RouteParams _routeParams;

  FieldsComponent(this._titleService, this._routeParams);

  @override
  void routerOnActivate(ComponentInstruction nextInst, ComponentInstruction prevInst) {
    _titleService.setTitle("List of Fields");
  }
```
We've implemented the `OnActivate` interface and its abstract method `routerOnActivate`.  This method simply uses the `Title` field to set the document title to "List of Fields".  `PlayersComponent` and `TeamsComponent` are similarly updated.

Running our app now will show an updated document title when we navigate among the three links.  But we still need to address the detail component.  

We could modify the `PlayerDetailComponent` in the same manner above with a new `routerOnActivate`.  But we can also add it to the existing `onInit` that we use to capture our route parameter:   

```dart 
  void ngOnInit() {
    player = _routeParams.get('id');
    _titleService.setTitle("Player Detail: $player");
  }
```

And we're done.  The page title will now update to include our player name when we click a player.

# Source

The [full source code of this example is available on Github](https://github.com/ilikerobots/angular_dart_page_titles_on_route).


