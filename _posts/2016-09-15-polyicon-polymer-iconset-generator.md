---
layout: post
title:  "PolyIcon: Polymer iconset generator"
date:   2016-09-14  20:50:00
categories: software polymer
---

I've just published [PolyIcon](http://polyicon.com), a web based utility
for building customized Polymer iconsets.

As [Polymer 1.0](https://www.polymer-project.org/1.0/) is young, its
toolbox is still rather sparse.  While sometimes I just use that screwdriver 
instead of a hammer, often I find myself pursuing a coding detour as I seek to
add to the toolbox.  It's slow, but educational and rewarding, and more
so when I can share a new tool with others. Such is the case with 
[PolyIcon](http://polyicon.com).  


## Polymer Iconsets

Polymer uses [iconsets](https://elements.polymer-project.org/elements/iron-iconset-svg)
to represent scalable icons.  Like most of Polymer, it is a clean encapsulated 
implementation and it makes a lot more sense than the traditional 
webfont/css method for icons.

A typical iconset is defined in an html file:

```html
<iron-iconset-svg size="1000" name="my-fancy-icons">
<svg xmlns="http://www.w3.org/2000/svg">
<defs>
  <g id="alert"><path d="M750 ..."/></g>
  <g id="star"><path d="M750 ..."/></g>
</defs>
</svg>
</iron-iconset-svg>
``` 

and then subsequently used elsewhere 

```html
<iron-icon icon="my-fancy-icons:star"></iron-icon>
```

Pretty nifty.


## Icon Building Tools

There are a lot of great pre-designed icon webfonts as well as tools to 
customize those.  But that is not the case with Polymer iconsets. 

The only extant tool I could find was [PolymerLabs' Polymer Iconset Generator](https://poly-icon.appspot.com/), 
which has a pretty slick UI but unfortunately only allows to build subsets
of the existing default polymer icons. 

When I needed a fully custom iconset for my project, I realized I would have to 
build my own from scratch.  This triggered at least two of my [three programmer virtues](http://threevirtues.com/)
, so I decided to code up a more general solution that I (and hopefully
others) could reuse.

I had previously made use of [Fontello](http://fontello.com) to build 
customized icon webfonts.  It's an excellent and well-written utility that will
produce webfonts in several formats.  

I forked Fontello, made some modifications such that it would produce
Polymer iconsets alongside its existings formats, and [offered it back to the 
Fontello authors via a Merge Request](https://github.com/fontello/fontello/pull/530).
However, the authors did not want to include polymer support, so it was 
rejected.

I understand the point, but I thought it would be unfortunate if 
Polymer authors couldn't take advantage of a similar web UI.  What to do?


## PolyIcon

So I invested a week of evenings making a conversion of Fontello
to produce only Polymer iconsets.  Along the way, I was able to wade ankle-deep in some new
technologies (most notably [Pug](https://pugjs.org/) and [Stylus](http://stylus-lang.com/)).

It's up and running now and I hope others will find it helpful.

Take a look at [polyicon.com](http://www.polyicon.com) and if you have 
any suggestions, issues, etc, let me know on [GitHub](https://github.com/ilikerobots)
or [Twitter](https://twitter.com/ilikerobotz).


 
