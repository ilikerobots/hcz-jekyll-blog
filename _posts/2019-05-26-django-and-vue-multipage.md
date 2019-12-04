---
layout: post
title:  "Django and Vue - Best of Both Frontends"
date:   2019-05-26  02:00:00
categories: django vue software 
---

_See also [Part 2]({% post_url 2019-12-04-django-vue-vuex %})_

# Django and Vue - Best of Both Frontends

Django and Vue both have unique frontend strengths.  Django's context-driven templates offer rapid development of pages directly from backend model content. Vue's modern reactive components provide powerful tools for building complex UIs within the rich Javascript ecosystem.   

However, typical solutions to integrating Django and Vue forgo much of the strengths of one lieu of the other.  For example, a common approach is to use Django Rest Framework as backend and writing the entire front end in Vue, making it difficult to utilize Django templates in places it could be expedient.  A second approach is to use Vue within Django templates using browser `<script>` includes, but then lost is the ability to use Vue's Single File Components.

I recently started a project where I wanted to reserve the option to utilize both Django Templates and Vue, without compromising either.  Most of my site's pages were relatively static listings of backend data with limited interactivity, exactly what Django excels at it in a few lines of code.   But I had small number of pages that contained specific areas of more complex data-driven interactivity, Vue's forté.   Could I find a way to rely on Django templates for my simpler pages while enriching parts of pages with Vue, all in a easy-to-manage configuration?

Could I have the best of both frontends?  

Of course!  This article would have been spectacularly inessential otherwise.

![Vue + Django](/assets/django-and-vue-multipage-assets/vue_plus_django_opt.svg){:width="90%" .center-image}

## Django Webpack Loader and the Vue Multi-Page App

The key pieces in this solution are the `django-webpack-loader` Django app and Vue's [ability to support Multi-Page Apps](https://cli.vuejs.org/guide/html-and-static-assets.html#building-a-multi-page-app).  By combining these tools, one can build a standard template-driven Django application that incorporates one or more individual webpack bundles in Django views.  Each of these bundles is a separate Vue "page" that can be built using all the standard Vue/Javascript/Webpack tooling.

## Adapting a Django project to use Vue

In the remaining sections of this article, I'll illustrate the modification of a basic template-driven Django app to utilize Vue components authored as Single File Components.  I omit the steps of building a simple Django site as it is [well-documented](https://docs.djangoproject.com/en/2.2/intro/tutorial01/) already, but if you would like a simplified starter point, you can use this article's example repo at tag [example_start](https://github.com/ilikerobots/django-vue-mpa/tree/example_start).   

For the purposes of this article, I assume you have two working templates into which you would like to add separate Vue components. 

In my example repo, these are `templates/vue_app_01.html` and `templates/vue_app_02.html`.

![Starting with no Vue](/assets/django-and-vue-multipage-assets/Screenshot_vue_app_placeholders.png){:width="50%" .center-image}


### Adding a simple Vue project

We'll start by embedding an entire simple Vue project in a subdirectory of our Django project.  First, ensure the Vue CLI is installed.

```sh
sudo npm install -g @vue/cli
```

Then we'll use the CLI to build the stock Vue starting project, placing it in the `vue_frontend` directory

```sh
vue create vue_frontend
```

You can simply accept all the defaults offered by the CLI.  You should see the CLI report that the project was successfully created.  We can verify our starter project is working by serving it:
```sh
cd vue_frontend
yarn serve
```
Point your browser to the URL reported to you (probably `http://localhost:8080`), and you should see the Vue sample app.

![Vue Start Page](/assets/django-and-vue-multipage-assets/Screenshot_vue_starter_page.png){:width="90%" .center-image}


### Convert the Vue Single Page App to a Multi-Page App

The starter project has a single point of entry at `index.js`.  We'll convert this project to use multiple "pages" as points of entry.

First off, we'll need a helper package `webpack-bundle-tracker` to help us track the bundles created by these multiple pages.   Add it to the project by running the following from your `vue_frontend` directory.

```sh
yarn add webpack-bundle-tracker --dev
```

Next we'll reconfigure the Vue project.  Create `vue_fontend/vue.config.js` with the contents

```js
const BundleTracker = require("webpack-bundle-tracker");

const pages = {
    'vue_app_01': {
        entry: './src/main.js',
        chunks: ['chunk-vendors']
    },
    'vue_app_02': {
        entry: './src/newhampshir.js',
        chunks: ['chunk-vendors']
    },
}

module.exports = {
    pages: pages,
    filenameHashing: false,
    productionSourceMap: false,
    publicPath: process.env.NODE_ENV === 'production'
        ? ''
        : 'http://localhost:8080/',
    outputDir: '../django_vue_mpa/static/vue/',

    chainWebpack: config => {

        config.optimization
            .splitChunks({
                cacheGroups: {
                    vendor: {
                        test: /[\\/]node_modules[\\/]/,
                        name: "chunk-vendors",
                        chunks: "all",
                        priority: 1
                    },
                },
            });

        Object.keys(pages).forEach(page => {
            config.plugins.delete(`html-${page}`);
            config.plugins.delete(`preload-${page}`);
            config.plugins.delete(`prefetch-${page}`);
        })

        config
            .plugin('BundleTracker')
            .use(BundleTracker, [{filename: '../vue_frontend/webpack-stats.json'}]);

        config.resolve.alias
            .set('__STATIC__', 'static')

        config.devServer
            .public('http://localhost:8080')
            .host('localhost')
            .port(8080)
            .hotOnly(true)
            .watchOptions({poll: 1000})
            .https(false)
            .headers({"Access-Control-Allow-Origin": ["*"]})

    }
};
```

Let's explore this configuration.

Upfront we declare a list of our pages `vue_app_01` and `vue_app_02`, which will become the names of our bundles, and define points of entry for each.  Note that `main.js` came with our starter app, but we haven't build `newhampshir.js` yet.  We will shortly.     

The `publicPath` section defines how Django will locate our bundles.  We have two variations which can be used, one for production and one for non-production.  In production, we set the `publicPath` empty, as this signals to `django-webpack-loader` to fall back to Django's standard static finder behavior.  However, in non-production mode we override this to point to our own webpack development server.

The `outputDir` defines where the production ready Javascript/CSS will be placed.  This most likely should be one of your Django application's static file locations.

Next we optimize our build using the splitChunks plugin, configuring it to extract any vendor javascripts into a single shared bundle.  This allows us to keep our individual component javascript files small and allow browsers to cache our common javascript while moving among pages.

By default, Vue constructs corresponding html files for our bundles, but we have no need for them since we'll be serving our bundles from Django templates.  By deleting the appropriate plugins from our config we can prevent these from being generated.

The BundleTracker plugin will create a file `vue_frontend/webpack-stats.json` that will describe the bundles produced by this build process.  This file will be used eventually by `django-webpack-loader` to identify and serve our bundles.  More on that later.

The `__STATIC__` alias is a neat bit of trickery [first described](https://ariera.github.io/2017/09/26/django-webpack-vue-js-setting-up-a-new-project-that-s-easy-to-develop-and-deploy-part-1.html) (I believe) by Alejandro Riera Mainar.  This will allows us to reference paths to static files within our Vue component as 
```html
<img src="~__STATIC__/logo.png">
```

And, finally, configure a development server for use in non-production modes, allowing us to hot reload our Vue components during front-end development.


Now that Vue is configured, we need to create our components.  These could be any Vue apps, but for this article I'll just adapt the starter project.  Let's just convert the starter project's `main.js` and `App.vue` into a second slightly different app.  

```sh
cp ./src/main.js ./src/newhampshir.js
cp ./src/App.vue ./src/App02.vue
```

Next, use vim or your favorite inferior editor to update `src/newhampshir.js` to point to our new App02 component, or, if you have sed, simply run the following:
```sh
sed -i 's/App/App02/g' src/newhampshir.js
```

Likewise, make a simple change to the App02 component so we'll be able to recognize it from our browser.  Again use an editor or simply run:

```sh
sed -i 's/pp/pp02/g' src/App02.vue
```

Now our Vue frontend is ready.  We can confirm everything builds correctly by issuing `yarn build`.  You should see some results similar to:
```
⠏  Building for production...

 DONE  Compiled successfully in 1430ms                                                                                       11:32:50 AM

  File                                      Size             Gzipped

  ../django_vue_mpa/static/vue/js/chunk-    82.77 KiB        29.90 KiB
  vendors.js
  ../django_vue_mpa/static/vue/js/vue_ap    4.63 KiB         1.62 KiB
  p_02.js
  ../django_vue_mpa/static/vue/js/vue_ap    4.63 KiB         1.63 KiB
  p_01.js
  ../django_vue_mpa/static/vue/css/vue_a    0.34 KiB         0.23 KiB
  pp_02.css
  ../django_vue_mpa/static/vue/css/vue_a    0.33 KiB         0.23 KiB
  pp_01.css

  Images and other types of assets omitted.

 DONE  Build complete. The ../django_vue_mpa/static/vue directory is ready to be deployed.
```


### Configuring Django to use webpack bundles

Now that our Vue application is building bundles correctly, we need to instruct Django how to locate and render these.  The `django-wepack-loader` project takes care of most of this for us.  Add this package to your Django app, by adding the following line to your ```requirements.txt```:

```
django-webpack-loader==0.6.0
```

then installing the new requirement:

```sh
pip install -r requirements.txt
```

In your Django settings file (e.g. `settings.py`) add `webpack_loader` to the list of `INSTALLED_APPS`.

Elsewhere in that same settings file, add the following lines:

```py
VUE_FRONTEND_DIR = os.path.join(BASE_DIR, 'vue_frontend')

WEBPACK_LOADER = {
    'DEFAULT': {
        'CACHE': not DEBUG,
        'BUNDLE_DIR_NAME': 'vue/',  # must end with slash
        'STATS_FILE': os.path.join(VUE_FRONTEND_DIR, 'webpack-stats.json'),
        'POLL_INTERVAL': 0.1,
        'TIMEOUT': None,
        'IGNORE': [r'.+\.hot-update.js', r'.+\.map']
    }
}
```

This configuration points `django-webpack-loader` to the stats file we generate during our Vue build.  

We're almost there.  The final step is to alter our templates to include our new Vue apps.   Choose one of your existing Django template files (in my case `vue_app_01.html`) and add the following:

{% raw  %}
```
{% load render_bundle from webpack_loader %}

<div id="app">
  <app></app>
</div>
{% render_bundle 'chunk-vendors' %}
{% render_bundle 'vue_app_01' %}
```
{% endraw %}

This code sets aside a container for our app, then includes all the necessary Javascript and CSS from our vendor bundle and our `vue_app_01` bundle.   

Pick a second template and include the same content, except substituting the `vue_app_02` bundle.

### Running the App

To run our app in development mode, we'll need to serve both Django's dev server *and* the webpack development server.  From the vue frontend directory, run

```sh
yarn serve
```

And, in a separate terminal in the Django root directory, run Django development server, e.g.
```sh
./manage.py runserver
```

Point your browser to your Django app (e.g. `http://localhost:8000`) and check out the two pages you modified.  You should see your templates, but now with each running a separate Vue components.



![Starting with no Vue](/assets/django-and-vue-multipage-assets/Screenshot_vue_django_two_apps.png){:width="90%" .center-image}


Inspecting the dev console, you can see that the Vue JS/CSS is being served from our webpack development server.  Also, both components are sharing the same `chunk-vendors.js` bundle.  Further, we can demonstrate that hot-reloading is working correctly by making an alteration to one of our components.  Without requiring a reload, the updates should take effect directly in the browser.


When it's time to deploy, or when we simply want to omit running our Vue dev server, we can build our Vue project in production mode.  Cancel the `yarn serve` process if it's running and instead run `yarn build`.  The optimized bundles will be built in and placed into our Django static file location, and `webpack-stats.json` will be updated to reflect this new configuration.  Go back to your web browser, reload the page, and you'll see that the Vue JS/CSS bundles are now loaded from your standard static files location.  

## Conclusion

Combining Django templates and Vue doesn't require you to compromise on the strengths of either.   Using the techniques described in this article, you are free to leverage both whenever and wherever is appropriate in your project.



### Additional Notes

The Vue components used in this example are very simple, but you are free to make them as complex as you wish, incorporating technologies like Vuex and Vuetify, third-party node modules, or additional webpack configurations.  Just modify your Vue/Webpack configuration as with any standard Vue project.

Using this technique, you are not limited to a single Vue component per page.  If you want to mount multiple components in separate containers on a single page, just mount each accordingly using a separate selector. 

Regarding editors, I have my Python IDE (PyCharm) opened to my Django application root, and in a separate window I keep my Javascript IDE (Webstorm) opened in the `vue_frontend` directory as a separate project.   This helps keeps a nice separation between the two.


I don't advocate any specific approach to production configuration.  You may wish to commit a production webpack-stats.json, maintain two separate versions for dev/production, or incorporate the building of bundles and `webpack-stats.json` into your delivery process. 

A [follow-up to this article]({% post_url 2019-12-04-django-vue-vuex %}) explains how to integrate Vuex, sharing state across components in a single page or between page loads.

### Acknowledgements 

The terrific article [Integrating Django and VueJs with Vue CLI 3 and Webpack Loader](https://medium.com/@rodrigosmaniotto/integrating-django-and-vuejs-with-vue-cli-3-and-webpack-loader-145c3b98501a) by [Rodrigo Smaniotto](https://medium.com/@rodrigosmaniotto) taught me how to use `django-webpack-loader` to integrate a Vue Single Page App.  Much of the `django-webpack-loader` configuration described in this article is based on his work.

Rodrigo's and this article also make use of work from [Django + webpack + Vue.js - setting up a new project that's easy to develop and deploy (part 1)](https://ariera.github.io/2017/09/26/django-webpack-vue-js-setting-up-a-new-project-that-s-easy-to-develop-and-deploy-part-1.html) by [Alejandro Riera Mainar](https://ariera.github.io/).  






## Source

See [django-vue-mpa](https://github.com/ilikerobots/django-vue-mpa) on github.





