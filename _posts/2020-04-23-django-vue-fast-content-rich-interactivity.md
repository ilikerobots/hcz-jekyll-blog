---
layout: post
title:  "Django and Vue - Blazing Content and Rich Interactivity"
date:   2020-04-23  02:00:00
categories: django vue software 
---

_See also [Part 2]({% post_url 2019-12-04-django-vue-vuex %}) of this Django + Vue series_

# Django + Vue - Blazing Content, Rich Interactivity

Albert Einstein discovered in 1905 the curious phenomenon of the _mass defect_, which is that the mass of a molecule is less than the sum of the masses of its constituent atoms.  In other words, he found that when two atoms are combined, the new molecule weighs less than the total weight of the original two atoms.  

Now, Einstein knew that nothing was truly lost and the mass defect could be explained by the interchangeability of mass and energy described by his famous equation E=mc^2. 

But how often do we attempt to combine software frameworks only to find that the result is less than the sum of its parts?  We are forced to compromise, to forgo the most compelling and innovative features to preserve compatibility.  We discover a _quality defect_.

Several months ago I began a project combining Django's frontend templating framework and Vue.  I wanted Django's tight integration to ORM and flexible caching.  I wanted Vue's modern Javascript flexibility and dynamism. I was fully expecting that I would lose something in the mix, limiting my ability to fully leverage one or both in my project.

But, after several months of working to meld these two technologies and, ultimately, launching my website, I am both surprised and pleased to find that I have been able to use every feature of Django and Vue I have wished.  Further, progress on my bipartite project **has been faster, easier, and more enjoyable than had I used Django or Vue exclusively.**

In the remainder of this article, I will demonstrate how Django and Vue can be used to deliver speedy pages that, in some scenarios, may be more performant than either Vue or Django would have delivered alone.

## A Great Weight Upon our Homepage

Consider the following situation.  We've built a blazing fast homepage: a contentful Django template tied to ORM and served directly from memory cache.  There's minimal Javascript but also minimal interactivity. One day, our manager gives us a task: we must introduce an interactive and complex component, one requiring numerous third party libraries, right onto our website's index page.  

"Great" we think, "there go the PageSpeed scores."  We plead and implore, but the only concession gained is that the widget can be displayed below the fold.

Can we preserve our fast load of content and still provide rich Vue interactivity?  

Yes!  We can can have the best of both worlds!


## Introducing Lazy-Loading to a Vue/Django Project 

This article will demonstrate how to lazy-load Vue components into a Django template view.  For those seeking fast pageloads (and high PageSpeed scores), this is an essential consideration.  With the techniques in this article, we can deliver a nearly Javascript-free fast-loading Django template with meaningful content, delaying the loading and processing required for weighty Vue components (or even the entire Vue framework itself) until our desired conditions are met.

![Vue + Django](/assets/django-and-vue-multipage-assets/vue_plus_django_opt.svg){:width="90%" .center-image}

I'll continue from where I left in Part 2: a working Django App integrated with a couple of Vue components using Vuex.  The companion example application also continues from the [same repository](https://github.com/ilikerobots/django-vue-mpa/tree/part_2) used in prior articles in this series.  The starting point for this article is tagged [part_2](https://github.com/ilikerobots/django-vue-mpa/tree/part_2).   

Additionally, at the end of the article I'll share links to a real world example.


### Adding a Component with Dependencies

To simulate our new weighty component, we'll use the Stopwatch component below. In truth, this component is rather trivial and has only a single external dependency (Moment.js), but the techniques in this article would apply to a very complex component with numerous additional dependencies.

First, we add Moment.js to our dependencies.

```sh
yarn add momentjs
```

And here is our Stopwatch component, which simply displays the number of seconds since it was loaded.

```vue
<template>
  <div class="stopwatch">
    <h3>This component was loaded {{ formattedTime }}</h3>
  </div>
</template>

<script>
  import * as moment from "moment";

  export default {
    name: 'Stopwatch',
    data() {
      return {
        elapsedTime: 0,
        timer: undefined
      };
    },
    created: function () {
      this.timer = setInterval(() => {
        this.elapsedTime += 1000;
        }, 1000);
      },
    computed: {
      formattedTime() {
        let secs = this.elapsedTime / 1000;
        return moment.duration(0 - secs, "second").humanize(true) + ` (${secs}s)`;
      }
    },
  }
</script>

<style scoped>
  .stopwatch { text-align: center; }
  .stopwatch h3 { color: #454545; }
</style>
```

Normally, to add this component to our index page, we would need only modify the existing entry point `index.js` as such:

```js
import Stopwatch from "./components/Stopwatch";

[...]

new Vue({
  el: "#stopwatch",
  components: {Stopwatch}
});
```

and place the appropriate container div into our Django template:   

```html 
<div id="stopwatch">
   <stopwatch></stopwatch>
</div>
```

If we did so, however, the component and all dependencies would be loaded immediately onto our page.  Let's avoid that.

### Splitting Chunks for Dependencies

But first, our Stopwatch's dependency on Moment.js has caused us another problem.  The `chunk-vendor` chunk, which contains all of our third party dependencies, now also includes Moment.js.  But since Moment.js is used only in a single component in our application, it's inefficient to include it in our main vendor chunk, which is used throughout our application.  


Currently, in our `vue.config.js` we define only a single chunk:

```js
config.optimization.splitChunks({
  cacheGroups: {
  vendor: {
    test: /[\\/]node_modules[\\/]/,
      name: "chunk-vendors",
      chunks: "all",
      priority: 1
    },
  },
});
```

The problem can be plainly seen by visualizing the bundle generated for our app.  With the configuration above, our bundle looks like this:

![Bundle before](/assets/django-and-vue-multipage-assets/Screenshot_bundles_before.png){:width="50%" .center-image}



We see that `chunk-vendors` is now dominated by Moment.js.  

We can remedy this by splitting the Moment.js dependency into a separate chunk. To do so, we add a separate cacheGroup to include only the code for the Moment.js dependency.

```js
config.optimization.splitChunks({
  cacheGroups: {
    moment: {
      test: /[\\/]node_modules[\\/]moment/,
      name: "chunk-moment",
      chunks: "all",
      priority: 5
    },
    vendor: {
      test: /[\\/]node_modules[\\/]/,
      name: "chunk-vendors",
      chunks: "all",
      priority: 1
    },
  },
});
```

Note the `test` for our new chunk is a regular expression that will match on the entire Moment.js library.  Also, we give this new chunk a higher priority, ensuring any modules unmatched will be collected into the base vendor chunk.

With this change, we can inspect our new bundles, and see that Moment.js has been isolated snugly into its own chunk.

![Bundle After](/assets/django-and-vue-multipage-assets/Screenshot_bundles_after.png){:width="50%" .center-image}

If we were to visit our application now with our web browser, our component would now fail to load because of the missing Moment.js dependency. We could explicitly load it (`{%raw%}{% render_bundle 'chunk-moment' %}{%endraw%}`), but we'll skip this for now, as we'll see later it won't be necessary.


### Multiple Entry Points per Page

To lazy-load our component, we will actually be lazy-loading an entire Vue entry point. Usually in a Vue multi-page app (MPA), there is one-to-one correspondence between entry point and physical page.  This is reflected in the Vue configuration, where entry points are defined in a list called `pages`.  But, this is not a formal restriction; we can in fact load as many entry points as we like on each physical html page.

In our case, our index page will load two entry points: 
 * `index` which loads immediately and contains the minimum js/css necessary to render our initial page load
 * `stopwatch` which loads lazily and contains only js/css necessary to build Stopwatch

Let's add our new `stopwatch` entry point in `vue.config.js`:

```js
const pages = {
 "stopwatch": {
    entry: "./src/stopwatch.js",
    chunks: ["chunk-moment", "chunk-vendors"],
  },
  [...]
}
```

Note that unlike our other entry points, `stopwatch` is dependent not only on the vendors chunk, but on our newly created `chunk-moment`.  

And here's the code for the new entry point `src/stopwatch.js`, identical in structure to our other entry points:

```js
import Vue from "vue/dist/vue.js";
const Stopwatch = () => import( /* webpackChunkName: "chunk-stopwatch" */ "./components/Stopwatch.vue");

Vue.config.productionTip = false;

new Vue({
  el: "#stopwatch",
  components: {Stopwatch}
});
```

The only notable variation is that we are now dynamically loading the Stopwatch component instead of using a static import.  While this is not strictly necessary, doing so will eliminate the need to manually render dependent chunks (in this case `chunk-moment`) in our Django template.  Instead, our component will dynamically load its own dependencies.



### Lazy-Loading a Vue Component


Now that our dependencies are isolated into chunks, we can begin work on lazy-loading our Stopwatch component. 

As a building block, let's first implement a vanilla Javascript function `load_bundle_file` to dynamically load an arbitrary Javascript or CSS resource.  Note for simplicity of the example, I inlined this script directly in the example app's base template, but of course you could use your Django foo to put this wherever is appropriate (and speedy).

```js
function load_bundle_file(url, filetype) {
  let fileref;
  if (filetype === "js") { // build js script tag
    fileref = document.createElement('script');
    fileref.setAttribute("type", "text/javascript");
    fileref.setAttribute("src", url);
  } else if (filetype === "css") { // build css link tag
    fileref = document.createElement("link");
    fileref.setAttribute("rel", "stylesheet");
    fileref.setAttribute("type", "text/css");
    fileref.setAttribute("href", url);
  }
  if (typeof fileref != "undefined")
    document.getElementsByTagName("head")[0].appendChild(fileref);
} 
```

The function accepts two arguments, a url to load and the type of that resource (i.e. js or css).  The script creates a corresponding link or script tag and appends it to the DOM's head.   

Next, we'll use this Javascript function to dynamically load our bundles.  To do so, I created a Django template tag to mimic `render_bundle` from `django-webpack-loader`.  I called this new template tag `lazy_render_bundle` and placed it in a custom template tag library `lazy_webpack_loader`.  

```py
from typing import Dict
from django import template

register = template.Library()

@register.inclusion_tag('lazy_render_bundle.html', takes_context=False)
def lazy_render_bundle(bundle: Dict[str, str]) -> Dict[str, Dict[str, str]]:
    return {'bundle': bundle}
```

And the corresponding template html:

```
{% raw %}{% load get_files from webpack_loader %}

{% get_files bundle as bundle_files %}
{% for f in bundle_files %}
    {% if f.url|default:""|slice:"-2:" == "js" %}
        load_bundle_file("{{ f.url }}", "js");
    {% else %}
        load_bundle_file("{{ f.url }}", "css");
    {% endif %}
{% endfor %}{% endraw %}
```

The template tag accepts a bundle name, but instead of directly rendering the bundle, it instead obtains all associated files for that bundle.  For each, it then calls our `load_bundle_file` Javascript function to dynamically load each js/css resource in the bundle.



Our wiring is complete. All that is left is to modify our Django template to use this wiring.

We'll need to choose some event to trigger the lazy-loading of our component. For simplicity, we'll create a button that, when tapped, will load our Stopwatch component.  With vanilla Javascript:

```html
{% raw %}<div>
 <input id="button_loader" type="button" value="Load stopwatch">
</div>

<div id="stopwatch">
  <stopwatch></stopwatch>
</div>

<script>
  let stopwatchBtn = document.getElementById('button_loader');
  stopwatchBtn.addEventListener('click', function (e) {
    stopwatchBtn.style.visibility = "hidden";
    {% lazy_render_bundle 'stopwatch' %}
  });
</script>{% endraw %}
```

In the code, we listen for a click on our button, and when it occurs we load our new single chunk `stopwatch` using the `lazy_render_bundle` template tag we created early.  The appropriate entry point code is loaded and Vue mounts our Stopwatch component to the element `#stopwatch`. Note there is no need to render `chunk-moment` because our `stopwatch.js` entry point uses dynamic imports.  Had we instead used a static import, we would then also need to explicitly render the moment chunk.



### The Results

With our both our Django development server running and our Vue front end serving (`yarn serve`), we can now view our results.

![Lazy loaded stopwatch](/assets/django-and-vue-multipage-assets/django_vue_mpa_component_lazy_load.gif){:.center-image}

Our index page loads a relatively lightweight set of resources (those required by the `index.js` entry point) on initial page load.  The weightier resources required to build and render our Stopwatch component remain unloaded.  When we click our load button, those are fetched and Vue mounts and renders our component.




### Going further

![Lazy loaded stopwatch](/assets/django-and-vue-multipage-assets/meme_lazy_load_all_the_vues.jpg){:width="75%" .center-image}

I mentioned earlier in this article it's possible to lazy-load the entire Vue framework.  I've included an example of this in the example repository.  The `fully_lazy.html` page uses `lazy_render_bundle` to defer loading of all chunks.  After a short timer expires, the chunks are loaded and all components are mounted, including restoration of Vuex state.

![Lazy-loaded stopwatch](/assets/django-and-vue-multipage-assets/django_vue_mpa_fully_lazy_load.gif){:.center-image}

In real-life scenarios, a hybrid approach could be used, loading a bare minimum of initial chunks, and lazy-loading additional entry points if/when needed. If particular Vue components are not required above the fold and/or immediately, then delaying the load of Vue until a scroll event or after a short timer can yield big gains in perceived speed or time to interactivity.  


### A Real World Example

If you'd like see these techniques utilized in a real world example, then take a look at our [Sidetrip Tours](https://www.sidetriptours.com) website. 

Our tour information (descriptions, images, locations, etc) are stored in Django's excellent ORM.  Because our general product information changes infrequently, we utilize Django's templating system to easily create contentful product views that can be cached in-memory server-side and rendered quickly by the client.

However, while our product metadata is relatively static, our pricing and availability data is highly dynamic.  Further, the user interactions involved in viewing and selecting availabilities and pricing options are more complex.  For both these reasons, a Vue implementation a better choice than Django templating.


You can observe in our website many of the strategies described in this article, particularly on mobile views of tour product pages such as our [Munich to Prague Sightseeing Tour](https://www.sidetriptours.com/tour/munich-to-prague-sightseeing-transfer-tour/).  Here, on the first pageview from a mobile device, loading of the Vue framework and pricing/availability components is deferred until the user scrolls the view.  On subsequent pageviews or when viewing from a desktop device (which has more horsepower), the resources are loaded immediately after DOM load.














### Acknowledgements 

Propers to [@owais](https://github.com/owais) and his [django-webpack-loader](https://github.com/owais/django-webpack-loader); absolutely essential to the techniques in this article.


## Source

See [django-vue-mpa](https://github.com/ilikerobots/django-vue-mpa) on github.





