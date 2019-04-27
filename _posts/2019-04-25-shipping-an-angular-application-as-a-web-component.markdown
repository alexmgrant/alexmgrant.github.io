---
layout: post
title: "Shipping an Angular Application as a Web Component"
date: 2019-04-25 12:01:05
categories:
  [
    AngularJs Upgrade,
    Web Components,
    Angular Elements,
    Angular,
    AngularJs,
    ngUpgrade,
  ]
description: "Ship your Angular application as a native web component. Even within another AngularJs app."
---

<ul class="post-tag">
  {% for tag in page.categories %}
  <li>
    <a href="{{ site.url }}/tags#{{ tag }}" class="tag">
      <span class="term">{{ tag }}</span>
    </a>
  </li>
  {% unless forloop.last %}{% endunless %}
  {% endfor %}
</ul>

Last updated: April 25, 2019

At [Xello](https://xello.world/en/careers/) our host application is written in AngularJs. We have many different teams shipping code to our host application of varying sizes. Today we want to be shipping Angular, not AngularJs, and the following is what has worked best for us when integrating Angular with an AngularJs host. Keep in mind that we could also ship our app to any web page or framework like this.

My local environment  
`node -v` = v11.6.0  
`npm -v` = 6.9.0

Installation (from scratch)  
`npm install -g @angular/cli`  
`ng new angular-elements`  
`cd angular-elements`  
`ng serve --open`

Let’s set up our app to build as a native web component.

Add the **@angular/elements** package to your project.  
`ng add @angular/elements`  

Next, we tell Angular which components we would like to build as custom elements (web components) using **enteryComponents** within `app.module.ts`.

Now we tell angular to create and define our custom element on boot within the constructor called with the **ngDoBootstrap** method. Your `app.module.ts` file should look like the following.

{% highlight JavaScript %}
// src/app/app.module.ts
import { BrowserModule } from "@angular/platform-browser";
import { NgModule, Injector } from "@angular/core";
import { createCustomElement } from "@angular/elements";

import { AppRoutingModule } from "./app-routing.module";
import { AppComponent } from "./app.component";

@NgModule({
  declarations: [AppComponent],
  imports: [BrowserModule, AppRoutingModule],
  providers: [],
  entryComponents: [AppComponent] // notice we remove bootstrap in favour of entryComponents
})
export class AppModule {
  constructor(private injector: Injector) {
    const appElement = createCustomElement(AppComponent, {
      injector: this.injector
    }); 

    customElements.define("app-element", appElement);
  }

  ngDoBootstrap() {}
}
{% endhighlight %}

You should now see a blank screen on `http://localhost:4200/`. No worries, we need to replace `<app-root></app-root>` with our new web component within `src/index.html`.
{% highlight html %}
<!-- src/index.html -->

<!doctype html>
<html lang="en">
    <head>
        <meta charset="utf-8">
        <title>AngularElements</title>
        <base href="/">

        <meta
            name="viewport"
            content="width=device-width, initial-scale=1">
        <link rel="icon" type="image/x-icon" href="favicon.ico">
    </head>
    <body>
        <app-element></app-element> <!-- 👈 our new web component -->
    </body>
</html>
{% endhighlight %}

We also need to add some polyfills to help browsers support web components.  
`npm install @webcomponents/webcomponentsjs --save`  
{% highlight javascript %}
// src/pollyfills.json

/***************************************************************************************************
* APPLICATION IMPORTS
*/
import "@webcomponents/webcomponentsjs/custom-elements-es5-adapter.js";
{% endhighlight %}

No more blank screen and now we're serving the app via a web component in our development environment 🥳  

Now let's take a look at routing because currently there are some [issues/feature](https://github.com/angular/angular/issues/23740) 🤷‍♀️ which stop the Angular router from working correctly within a web component.

Add a component and define a path, so it loads when we init our app.
`ng g component home`  

{% highlight javascript %}
// src/app/app.routing.module.ts

import { NgModule } from "@angular/core";
import { Routes, RouterModule } from "@angular/router";
import { HomeComponent } from "./home/home.component";

const routes: Routes = [
  { path: "", redirectTo: "/home", pathMatch: "full" },
  { path: "home", component: HomeComponent }
];

@NgModule({
  imports: [RouterModule.forRoot(routes)],
  exports: [RouterModule]
})
export class AppRoutingModule {}
{% endhighlight %}

You'll notice the app does not load the new HomeComponent. We can fix that by explicitly passing the Angular router the current path. `app.module.ts` should look like the following. 

{% highlight javascript %}
// src/app/app.module.ts

import { BrowserModule } from "@angular/platform-browser";
import { NgModule, Injector } from "@angular/core";
import { createCustomElement } from "@angular/elements";
import { Router } from "@angular/router";
import { Location } from "@angular/common";

import { AppRoutingModule } from "./app-routing.module";
import { AppComponent } from "./app.component";
import { HomeComponent } from "./home/home.component";

@NgModule({
  declarations: [AppComponent, HomeComponent],
  imports: [BrowserModule, AppRoutingModule],
  providers: [],
  entryComponents: [AppComponent] // notice we remove bootstrap in favour of entryComponents
})
export class AppModule {
  constructor(
    private injector: Injector,
    private router: Router,
    private location: Location
  ) {
    const appElement = createCustomElement(AppComponent, {
      injector: this.injector
    });

    customElements.define("app-element", appElement);

    //init router with starting path
    this.router.navigateByUrl(this.location.path(true));

    //on every route change tell router to navigate to defined route
    this.location.subscribe(data => {
      this.router.navigateByUrl(data.url);
    });
  }

  ngDoBootstrap() {}
}
{% endhighlight %}

Now our app inits with our `homeComponent` displaying. Let’s set up another route to test routing changes. We’ll make a lazy loaded module now so we can see everything works.  
`ng g module lazy`  
`ng g component lazy`

{% highlight javascript %}
// src/app/lazy/lazy.module.ts

import { NgModule } from "@angular/core";
import { CommonModule } from "@angular/common";
import { Routes, RouterModule } from "@angular/router";

import { LazyComponent } from "./lazy.component";

const routes: Routes = [
  {
    path: "",
    component: LazyComponent
  }
];

@NgModule({
  declarations: [LazyComponent],
  imports: [CommonModule, RouterModule.forChild(routes)]
})
export class LazyModule {}
{% endhighlight %}

{% highlight html %}
// src/app/home/home.component.html

<p>home works!</p>

<a [routerLink]="['/lazy']">Lazy link</a>
{% endhighlight %}

{% highlight javascript %}
// src/app/app-routing.module.ts

import { NgModule } from "@angular/core";
import { Routes, RouterModule } from "@angular/router";
import { HomeComponent } from "./home/home.component";

const routes: Routes = [
  { path: "", redirectTo: "home", pathMatch: "full" },
  { path: "home", component: HomeComponent },
  { path: "lazy", loadChildren: "./lazy/lazy.module#LazyModule" }
];

@NgModule({
  imports: [RouterModule.forRoot(routes)],
  exports: [RouterModule]
})
export class AppRoutingModule {}
{% endhighlight %}

Now we can see everything works with the Angular router within a web component. 🎉

Next we optimize our build process for shipping our app.  
First we change the build config within `angular.json`.

{% highlight javascript %}
  "projects": {
    "angular-elements": {
      ...
      "architect": {
        "build": {
          ...
          "options": {
            ...
            "scripts": [
              {
                // this is needed for the elements to get registered properly
                // note that the "ng add" command adds this automatically
                "input": "node_modules/document-register-element/build/document-register-element.js"
              }
            ]
          },
          "configurations": {
            "production": {
              ...
              // no hashing makes the file names easy to concatenate
              "outputHashing": "none",
{% endhighlight %}

Next, we create a Node.js script to concat our build files into a single file so we can quickly ship our application. Create a file called `build-angular-elements.js` in the root of the project. 

{% highlight javascript %}
// build-angular-elements.js

const fs = require('fs-extra');
const concat = require('concat');
(async function build() {
  const files = [
    './dist/angular-elements/runtime.js',
    './dist/angular-elements/polyfills.js',
    './dist/angular-elements/scripts.js',
    './dist/angular-elements/main.js'
  ];
  await fs.ensureDir('angular-elements-build');
  await concat(files, 'angular-elements-build/angular-elements.js');
  await fs.copy('./dist/angular-elements/styles.css', 'angular-elements-build/styles.css');
  await fs.copy('./dist/angular-elements/assets/', 'angular-elements/assets/');
})();
{% endhighlight %}

Install concat and fs-extra  
`npm i concat fs-extra --save-dev`  

Create an npm script to build our angular app and run our concat script.
{% highlight javascript %}
// package.json

"scripts": {
  ...
  "build:angular-elements": "ng build angular-elements --prod && node build-angular-elements.js"
{% endhighlight %}

Now run our new build script.  
`npm run build:angular-elements`

Success, we now have our app built as a web component available in the root of our project under the `angular-elements-build` directory. Include these files into any website and enjoy! 