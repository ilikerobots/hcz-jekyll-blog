---
layout: post
title:  "Setting Document Titles in Angular2 Dart - Improved"
date:   2016-11-25  17:20:00
categories: dart angular2 software 
---

In my [previous post]({% post_url 2016-11-22-setting-document-title-in-angular2-dart %}) I explained one method of setting the document title on route changes in Angular2 Dart.  There were a couple shortcomings, however:  

  1. Our page components set document titles themselves directly.  However, this is mighty presumptuous of such lowly components.  Ideally, we'd like the task of naming the page to be handled by some higher-level component.  
  2. The post was as faithful a reproduction of the strategy described for Typescript as I could manage, but one of the considerations presented there (platform agnostic components) is moot in Dart.

This post will rectify both situations, making the example simpler and more useful. 


# Solution Overview

In this iteration we will create a custom service to set document titles based on route events.  This service will be injected into the `AppComponent` which is already handling routing.  Thus, the lowly page components need not be aware of any document title responsibilities.  We could use them in different ways, even in different apps, without worry that they would disrupt document titles.

As before, we'll continue to rely on Angular's `Title` to abstract away any DOM considerations.  However, this time we will locate the registration of this service in `AppComponent`.  Because Angular2 Dart has no concept of platforms (or has only one platform, depending on how you look at it), then my previous attempt to keep things platform agnostic was meaningless.  And while there's nothing particularly wrong with registering providers in the bootstrap, I prefer it done in `AppComponent` as a matter of style.

Our prep and simple app will be the exact same as in the prior post. I've included these two sections below to make this post easier to read.  If you're looking for quick answers, skip ahead to [A Simple Title Setting Service](#a-simple-title-setting-service) or straight to the [source code](#source).  

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


# A Simple Title Setting Service

We will create a service `TitleSetService` that will subscribe to routing events and update document titles accordingly.  Thus our new service will require two services itself: `Title` and `Router`, so we'll inject those two into the component and assign them to fields for later use.  Here's a basic skeleton:

```dart
import 'dart:async';
import 'package:angular2/core.dart';
import 'package:angular2/platform/browser.dart';
import 'package:angular2/router.dart';

@Injectable()
class TitleSetService {
  Router _router;
  Title _title;

  TitleSetService(this._router, this._title); 
}
```

Now we update our service to do something useful, i.e. set the title on routing events.  We start simple, subscribing to router events and setting the page title to the currently activated route name.

```dart
  TitleSetService(this._router, this._title) {
    _router.subscribe(_setTitleFromRoute);
  }
  
  void _setTitleFromRoute(String url) async {
    //identify component instruction from routed url
    ComponentInstruction compInst = (await _router.recognize(url))?.component;
    if (compInst != null) {
      _title.setTitle(compInst.routeName);
    }   
  }    
```

We now have a basic `TitleSetService` so let's update our app to use it by providing this new service and its dependency `Title` in `AppCompponent`:


```dart
    providers: const [
      ROUTER_PROVIDERS,
      const Provider(Title, useClass: Title),
      TitleSetService,
    ]
```

and finally injecting the `TitleSetService` into `AppComponent`.

```
AppComponent(TitleSetService _titleSet);
```

When we run the app now, we'll see the document titles update to the route name as we navigate.  We've accomplished the goal of updating page titles without sub-components being aware.  Instead, these concerns are nicely encapsulated in a single service.  

The only problem is that the route names don't make very good document titles.  Let's improve that.



# Improving the Title Setting Service


We update the `TitleSetService` to accept a Function that will dictate how page titles will be set:

```dart
typedef String TitleNamingFunction(ComponentInstruction c);

@Injectable()
class TitleSetService {
  TitleNamingFunction nameStrategy;

  Router _router;
  Title _title;

  TitleSetService(this._router, this._title) {
    nameStrategy = _defaultNameStrategy;
   _router.subscribe(_setTitleFromRoute);
  }

  Future<Null> _setTitleFromRoute(String url) async {
    //identify component instruction from routed url
    ComponentInstruction compInst = (await _router.recognize(url))?.component;
    if (compInst != null) {
      _title.setTitle(nameStrategy(compInst));
    }
  }

  String _defaultNameStrategy(ComponentInstruction compInst) {
    return compInst.routeName;
  }
```

There are several new items here.  First, we've made a typedef that will define a callback interface for naming functions.  Such functions should accept a `ComponentInstruction` and return a `String`.

We've declared a field named `nameStrategy` to store a custom naming function and, in the constructor, initialized it to a default implementation, `_defaultNameStrategy`.

Lastly, we've updated `_setTitleFromRoute` to use a custom naming function.

```dart
    if (compInst != null) {
      _title.setTitle(nameStrategy(compInst));
    }

```

Running the app now, everything behaves exactly as before, but we now have the opportunity to adjust the naming strategy.  Let's do so, by altering the `AppComponent` constructor to provide a custom naming strategy to `TitleSetService`:


```dart
  AppComponent(TitleSetService _titleSet) {
    _titleSet.nameStrategy = _setTitle;
  }


  String _setTitle(ComponentInstruction c) {
    StringBuffer sb = new StringBuffer();
    sb.write("Title Set Demo | ");

    if (c.routeData.data.containsKey('title')) { // if title is in data, use it
      sb.write(c.routeData.data['title']);
    } else { //otherwise use route name
      sb.write(c.routeName);
    }

    if (c.params.containsKey('id')) { // if detail id in params, append it
      sb.write(": ${c.params['id']}");
    }
    return sb.toString();
  }

```

The `_setTitle` method is more sophisticated than the default naming strategy provided by `TitleSetService`.  This new strategy
  1. includes a base prefix to all document titles
  2. uses a `title` data attribute if available instead of route name 
  3. appends a route id if available

We can now update `AppComponent`'s  `RouteConfig` to include some title data where needed

```dart
 @RouteConfig(const [
   const Route(path: '/players', name: 'Players', component: PlayersComponent, useAsDefault: true),
   const Route(path: '/teams', name: 'Teams', component: TeamsComponent),
   const Route(path: '/fields', name: 'Fields', component: FieldsComponent, 
               data: const{'title': 'Ballparks'}),
   const Route(path: '/player/:id', name: 'PlayerDetail', component: PlayerDetailComponent,
               data: const{'title': 'Player'}),
 ])
```

Now, when we browse about the app, we see much more reasonable page titles.  The fields page has been titled "Ballparks" and the player detail page is titled based on the current player.

# Source

The [full source code of this example is available on Github](https://github.com/ilikerobots/angular_dart_page_titles_on_route).


