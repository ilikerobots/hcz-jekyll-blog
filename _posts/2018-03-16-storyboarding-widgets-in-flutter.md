---
layout: post
title:  "Storyboarding Widgets in Flutter"
date:   2018-03-12  02:00:00
categories: flutter, dart, software 
---

# Storyboarding Widgets in Flutter

While developing a [React Native](https://facebook.github.io/react-native/) app, a tool I found very useful was [Storybook](https://storybook.js.org/), which allows developers to write small isolated "stories" involving widgets.  These stories, especially when teamed with a hot deploy mechanism, allows quick iteration of widget design, and also promotes improved reusability and testability of your widgets.

Now I'm developing for [Flutter](https://flutter.io) and I sought an analogue to Storybook.  The bad news was that I didn't find one; the great news is that Flutter's built-in hot reload and tooling support makes rolling your own storyboards simple.  As simple as it is, I've published my version as a Flutter dart package, making it very fast to begin incorporating storyboards into your own dev cycle.

      
## What is a Storyboard

The goal of a storyboard is to stage a collection of Widgets outside their normal app context.  By repeating the same widget in various configurations or states in a single view, the effect of code changes to the Widget can quickly be assessed. This is especially useful for cosmetic changes, but also effective for behavior changes.


![Screenshot of Storyboard](/assets/storyboarding-widgets-in-flutter-assets/Screenshot_storyboard.png){:width="33%" .center-image}

## A Simple Storyboard

A simple DIY storyboard can easily be created by making a Material App consisting of a basic scaffold holding a single widget.  

```dart
void main() {
  runApp(
      new MaterialApp(
          home: new Scaffold(
              appBar: new AppBar(title: "My Storyboard"),
              body: new FancyWidget()
          )));
}
```

Even such a simple example has a lot of power when teamed with Flutter's amazing hot-reload.  We can evolve our widget incrementally using this simple app, quickly observing the effect of code changes without the need to run our full app. This exercise is so simple and useful that I suspect many Flutter developers regularly use a similar approach during widget development.

## A Less Tedious Storyboard

Building storyboards as above becomes tedious quickly.  To remedy this, I developed a [small Flutter package](https://pub.dartlang.org/packages/storyboard) to reduce boilerplate and make it easier to build Storyboards consisting of multiple "Stories".

The code is rather trivial, actually; the magic is all built into Flutter itself. So if you'd like to start storyboarding, then [head straight to the Pub](https://pub.dartlang.org/packages/storyboard). Otherwise, I'll give a brief explanation of how the package works.

The basic building block of the package is the abstract Story class.  A developer will create a story by creating concrete implementations of this class.  The only method requiring overriding is ```get storyContent```, which is where the widgets to be observed are defined. 

```dart
abstract class Story extends StatelessWidget {
  const Story({Key key}) : super(key: key);
  List<Widget> get storyContent;
  String get title => new ReCase(runtimeType.toString()).titleCase;
  bool get isFullScreen => false;

  Widget _widgetListItem(Widget w) =>
      new Row(mainAxisAlignment: MainAxisAlignment.center, children: [
        new Container(padding: const EdgeInsets.symmetric(vertical: 8.0), child: w)]);

  Widget _widgetTileLauncher(Widget w, String title, BuildContext context) =>
      new ListTile(
          leading: const Icon(Icons.launch),
          title: new Text(title),
          onTap: () {
            Navigator.push(context,
                new MaterialPageRoute<Null>(builder: (BuildContext context) { return w; }));
          });

  @override
  Widget build(BuildContext context) {
    if (!isFullScreen) {
      return new ExpansionTile(
        leading: const Icon(Icons.list),
        key: new PageStorageKey<Story>(this),
        title: new Text(title),
        children: storyContent.map(_widgetListItem).toList(),
      );
    } else {
      if (storyContent.length == 1) {
        return _widgetTileLauncher(storyContent[0], title, context);
      } else {
        return new ExpansionTile(
          leading: const Icon(Icons.fullscreen),
          key: new PageStorageKey<Story>(this),
          title: new Text(title),
          children: storyContent.map((Widget w) => _widgetTileLauncher(w, title, context)).toList(),
        );
      }
    }
  }
}
```

The Story class includes logic necessary to render itself and its widget children, either in a vertical list or in a separate Scaffold (if the Story overrides ```isFullScreen``` as true).  The ```build``` method itself can be overridden if more precise control of rendering is required.

Here's a simple story that shows off a widget using different configurations.

```dart
/* myapp/storyboard/stories/attribute_bar_story.dart */
import 'package:my_app/attribute_bar.dart';
import 'package:flutter/material.dart';
import 'package:storyboard/storyboard.dart';

class AttributeBarStory extends Story {
  @override
  List<Widget> get storyContent {
    return [
      new AttributeBar(label: [
        const Icon(Icons.ac_unit, color: Colors.white),
        const Icon(Icons.wb_sunny, color: Colors.yellow),
        const Icon(Icons.local_florist, color: Colors.green),
      ]),
      new AttributeBar(textDirection: TextDirection.rtl, label: [
        const Icon(Icons.free_breakfast, color: Colors.blue),
        const Icon(Icons.grain, color: Colors.yellow),
        const Icon(Icons.restaurant, color: Colors.green),
      ]),
    ];
  }
}
```

Multiple stories are bundled together to build a Storyboard, a widget that renders a scaffold with its children in a simple List view.  
```dart
class Storyboard extends StatelessWidget {
  final _kStoryBoardTitle = "Storyboard";

  Storyboard(this.stories)
      : assert(stories != null),
        super();

  final List<Story> stories;

  @override
  Widget build(BuildContext context) {
    return new Scaffold(
        appBar: new AppBar(title: new Text(_kStoryBoardTitle)),
        body: new ListView.builder(
          itemBuilder: (BuildContext context, int index) => stories[index],
          itemCount: stories.length,
        ));
  }
}
```



Here's an example of running a simple Storyboard.
```dart
/* myapp/storyboard/basic_widget_storybook.dart */
import 'package:flutter/material.dart';
import 'package:storyboard/storyboard.dart';
import 'stories/attribute_bar_story.dart';
import 'stories/grid_list_story.dart';
import 'stories/grid_item.dart';

void main() {
  runApp(new StoryboardApp([
    new AttributeBarStory(),
    new GridListStory(),
    new GridItemStory(),
  ]));
}
```

And here's how it looks in action with hot-reload:

{% include youtube.html id=eKj3LI70Vhk %}


## Integration Testing

Storyboards can easily do double duty as scaffolding for integration testing using ```flutter_driver```.  As part of your integration workflow, an integration test can drive a storyboard, generating performance metrics and screenshots along the way.
See [Flutter docs](https://flutter.io/testing/#integration-testing) for more info.


## Conclusion
This small storyboarding technique leverages Flutter's top-tier mobile developer experience to easily build the seed of a valuable dev tool.

Flutter is still young, but as more and more of the usual supporting cast of developers tools is built, Flutter will become something very special.  

## Source

See the [Storyboard package](https://pub.dartlang.org/packages/storyboard) on dart pub.




