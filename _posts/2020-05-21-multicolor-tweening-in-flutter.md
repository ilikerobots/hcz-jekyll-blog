---
layout: post
title:  "Multicolor Tweens in Flutter"
date:   2020-05-21  02:00:00
categories: flutter dart software
---

# Multicolor Tweens in Flutter

Flutter's built-in `ColorTween` is the standard way to animate a color transition between two colors.  With `ColorTween` we can _lerp_ (linearly interpolate) between two colors along the lifetime of a controller's animation.

But how can we smoothly transition between multiple colors?  

This article discusses three methods, each which may suit different situations.

![Three methods of multicolor tweens](/assets/multicolor-tweening-in-flutter/multicolor_tween_flutter.mp4.gif){:width="75%" .center-image}


# Tween Sequence Approach

First, we recognize that transitioning between multiple colors is equivalent to transitioning between pairs of colors in sequence.  For example, the transition of `Red → Blue → Green → Yellow → Red` is simply the seamless succession of transitions:

* Red → Blue
* Blue → Green
* Green → Yellow
* Yellow → Red

In Flutter, we can mirror this concept exactly by building a composite tween, `TweenSequence`, consisting of individual ColorTweens.

```dart
Animatable<Color> bgColor = TweenSequence<Color>([
  TweenSequenceItem(
    weight: 1.0,
    tween: ColorTween(begin: Colors.red, end: Colors.blue),
  ),
  TweenSequenceItem(
    weight: 1.0,
    tween: ColorTween(begin: Colors.blue, end: Colors.green),
  ),
  TweenSequenceItem(
    weight: 1.0,
    tween: ColorTween(begin: Colors.green, end: Colors.yellow),
  ),
  TweenSequenceItem(
    weight: 1.0,
    tween: ColorTween(begin: Colors.yellow, end: Colors.red),
  ),
])
``` 

Each item in the sequence is a two-color transition, and each `SequenceItem` has equal weight, meaning each occupy equal parts of the total animation duration (i.e. one-fourth).

This tween can now be used to form an animation in a widget's state.  In the example below, a `TweenSequence` _bgColor_ is driven by a 5-second looping controller to build an `Animation`.

```dart
class _TweenSequenceBoxState extends State<TweenSequenceBox>
    with SingleTickerProviderStateMixin {
  AnimationController _controller;
  Animation<Color> _colorAnim;

  @override
  void initState() {
    super.initState();

    _controller = AnimationController(
      duration: const Duration(seconds: 5),
      vsync: this,
    )..forward()..repeat();

    _colorAnim = bgColor.animate(_controller);
  }

 // Our build method will go here

 @override
  void dispose() { // don't forget to clean up!
    _controller.dispose();
    super.dispose();
  }

}
```

With the animation prepared, the build method makes use of an `AnimatedBuilder` to draw a `Container` with a background color.  Each frame, the widget is rebuilt with a background color as defined by the `_colorAnim` value.

```dart
  Widget build(BuildContext context) {
    return AnimatedBuilder(
        animation: _controller,
        builder: (context, child) {
          return Container(
            padding: EdgeInsets.all(24.0),
            decoration: BoxDecoration(color: _colorAnim.value),
            child: Text('Tween Sequence'),
          );
        });
  }
```



## Rainbow Color Tween Approach

The [rainbow_color](https://pub.dev/packages/rainbow_color) package provides a simpler interface to build multicolor tween: `RainbowColorTween`.  This may be used as a drop-in replacement for an equal-weighted `TweenSequence<Color>`.  For example, the _bgColor_ sequence above can be replaced with the more direct

```dart
 Animatable<Color> bgColor = RainbowColorTween([
    Colors.red,
    Colors.blue,
    Colors.green,
    Colors.yellow,
    Colors.red,
  ])
```

All other code remains identical.

## Rainbow Interpolation Approach


`RainbowColorTween` utilizes the `Rainbow` multicolor interpolation class behind the scenes.  If a widget involves numerous derived multicolor animations, especially those dependent on the spectrum itself, then it may be useful to directly use Rainbow spectrum interpolation.  For example, the following widget shows the text foreground color "lagging" one color behind the background in the sequence:

```dart
class _RainbowBoxState extends State<RainbowBox>
    with SingleTickerProviderStateMixin {
  AnimationController _controller;
  Animation<double> _anim;

  @override
  void initState() {
    super.initState();
    _controller = AnimationController(
      duration: const Duration(seconds: 5),
      vsync: this,
    )..forward()..repeat();
    _anim = bgValue.animate(_controller); 
  }

  // animate a double from 0 to 10
  Animatable<double> bgValue = Tween<double>(begin: 0.0, end: 10.0);

  // build a rainbow spectrum that blends across the same numerical domain
  Rainbow rb = Rainbow(rangeStart: 0.0, rangeEnd: 10.0, spectrum: [
    Colors.red,
    Colors.blue,
    Colors.green,
    Colors.yellow,
    Colors.red,
  ]);

  Widget build(BuildContext context) {
    return AnimatedBuilder(
        animation: _controller,
        builder: (context, child) {
          return Container(
              padding: EdgeInsets.all(24.0),
              decoration: BoxDecoration(color: rb[_anim.value]),  // lerp for background color
              child: Text('Rainbow'),
                style: TextStyle(color: rb[(_anim.value - 2) % 10]), //shift one color backward
              );
        });
  }
}

// dispose() omitted
}
```

## Conclusion

For simple equal-weighted multicolor tweens without curves, `RainbowColorTween` is the simplest solution.  For more complex use-cases, combine `ColorTween` items into a `TweenSquence`.

Finally, if animations involve choreography within a multicolor spectrum or across several spectra, then utilizing `Rainbow` directly to interpolate may prove simplest.

## Source

See [multicolor_tweening_flutter_example](https://github.com/ilikerobots/multicolor_tweening_flutter_example) on Github.





