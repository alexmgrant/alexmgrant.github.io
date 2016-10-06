---
layout: post
title:  "Ionic rc0 & Firebase"
date:   2016-10-6 12:01:05
categories: Ionic
---
So Ionic rc0 is here. Wooop! RIP Webpack. Wait no Webpack? Nope, we're using [rollupjs](http://rollupjs.org/) now. 
This comes with a few problems, also known as changes. Let's go over how we can get some [third party libraries](http://ionicframework.com/docs/v2/resources/third-party-libs/) loaded up 
into our shiny new Ionic project.  

TLDR: Here is a blank tabs starter project with everything discussed below: [Ionic rc0 & Firebase](https://github.com/alexmgrant/ionic_rc0_firebase)  
`git clone git@github.com:alexmgrant/ionic_rc0_firebase.git`  
`cd ionic_rc0_firebase`  
`npm i`  
`ionic serve`

### Installation (from scratch) 
`mkdir your_project_name`  
`cd your_project_name`    
`ionic start blank --v2`

For this project we're going to install lodash as well as Firebase  
`npm i lodash firebase --save`  

Typescript will want the static types for our newly added libraries   
`npm install @types/lodash --save`  

Install your packages  
`npm i`  

Apparently @types ecosystem should come through for us "most of the time", however in the case of Firebase they have not updated their static types to their latest api 3.0. No sweat.  
Add Firebase types to `/tsconfig.json`  
{% highlight json %}
{
  "compilerOptions": {
    "allowSyntheticDefaultImports": true,
    "declaration": true,
    "emitDecoratorMetadata": true,
    "experimentalDecorators": true,
    "lib": [
      "dom",
      "es2015"
    ],
    "module": "es2015",
    "moduleResolution": "node",
    "target": "es5"
  },
  "types": [
    "firebase"
  ],
  "exclude": [
    "node_modules"
  ],
  "compileOnSave": false,
  "atom": {
    "rewriteTsconfig": false
  }
}
{% endhighlight %}  

Next we need to tell rollupjs to use our commonjs modules ie. Firebase. So we don't have to [dig into ionic's implementation](https://github.com/driftyco/ionic-app-scripts/) we can use `package.json` to build our own custom rollupjs script.  

Setup our `/package.json`  
{% highlight Json %}
{
  "name": "ionic-hello-world",
  "author": "Ionic Framework",
  "homepage": "http://ionicframework.com/",
  "private": true,
  "scripts": {
    "build": "ionic-app-scripts build --rollup ./config/rollup.config.js", // now when build is called we replace Ionic's config with our own
    "watch": "ionic-app-scripts watch",
    "serve:before": "watch",
    "emulate:before": "build",
    "deploy:before": "build",
    "build:before": "build",
    "run:before": "build"
  },
  "config": { // here we use config to call our custom rollup.js 
    "ionic_rollup": "./config/rollup.config.js"
  },
  "dependencies": {
    "@ionic/storage": "^1.0.3",
    "@types/lodash": "^4.14.37",
    "firebase": "^3.4.1",
    "ionic-angular": "^2.0.0-rc.0",
    "ionic-native": "^2.0.3",
    "ionicons": "^3.0.0",
    "lodash": "^4.16.4"
  },
  "devDependencies": {
    "@ionic/app-scripts": "latest",
    "typescript": "^2.0.3"
  },
  "cordovaPlugins": [
    "cordova-plugin-device",
    "cordova-plugin-console",
    "cordova-plugin-whitelist",
    "cordova-plugin-splashscreen",
    "cordova-plugin-statusbar",
    "ionic-plugin-keyboard"
  ],
  "cordovaPlatforms": [
    "ios",
    {
      "platform": "ios",
      "version": "",
      "locator": "ios"
    }
  ],
  "description": "ionic_rc0_firebase: An Ionic project"
}

{% endhighlight %}

Let's create our custom `rollup.config.js`. Create a `/config` directory  
`mkdir config`  

Inside of our new `/config` ceate a file named `rollup.config.js` and place the following setup into that file. 

{% highlight JavaScript %}
var nodeResolve = require('rollup-plugin-node-resolve');
var commonjs = require('rollup-plugin-commonjs');
var globals = require('rollup-plugin-node-globals');
var builtins = require('rollup-plugin-node-builtins');
var json = require('rollup-plugin-json');

// https://github.com/rollup/rollup/wiki/JavaScript-API

var rollupConfig = {
  useStrict: false,
  /**
   * entry: The bundle's starting point. This file will
   * be included, along with the minimum necessary code
   * from its dependencies
   */
  entry: 'src/app/main.dev.ts',

  /**
   * sourceMap: If true, a separate sourcemap file will
   * be created.
   */
  sourceMap: true,

  /**
   * format: The format of the generated bundle
   */
  format: 'iife',

  /**
   * dest: the output filename for the bundle in the buildDir
   */
  dest: 'main.js',

  /**
   * plugins: Array of plugin objects, or a single plugin object.
   * See https://github.com/rollup/rollup/wiki/Plugins for more info.
   */
  plugins: [
    builtins(),
    commonjs({
      include: [
        'node_modules/rxjs/**', // firebase depends on rxjs and needs this to avoid build errors
        'node_modules/firebase/**' // here is firebase
      ],
      namedExports: {
        'node_modules/firebase/firebase.js': ['initializeApp', 'auth', 'database']
      }
    }),
    nodeResolve({
      module: true,
      jsnext: true,
      main: true,
      browser: true,
      extensions: ['.js']
    }),
    globals(),
    json()
  ]
};

if (process.env.IONIC_ENV == 'prod') {
  // production mode
  rollupConfig.entry = '{{TMP}}/app/main.prod.ts';
  rollupConfig.sourceMap = false;
}

module.exports = rollupConfig;

{% endhighlight %}

Now let's include Firebase into our project within `/src/app/app.component.ts`  

{% highlight JavaScript %}
import { Component } from '@angular/core';
import { Platform } from 'ionic-angular';
import { StatusBar } from 'ionic-native';

import { HomePage } from '../pages/home/home';
import firebase from 'firebase'; // Firebase does not support named exports, but we can use the default syntax

const firebaseconfig = { // setup your Firebase config
  apiKey: '',
  authDomain: '',
  databaseURL: '',
  storageBucket: ''
};

@Component({
  template: `<ion-nav [root]="rootPage"></ion-nav>`
})
export class MyApp {
  rootPage = HomePage;

  constructor(platform: Platform) {
    firebase.initializeApp(firebaseconfig); // init Firebase

    platform.ready().then(() => {
      StatusBar.styleDefault();
    });
  }
}
{% endhighlight %}

Start it up  
`ionic serve`  

So it works, but until Firebase updates this setup is not ideal. You'll get a couple errors in the console.  
`$ rollup: Use of 'eval' (in /your_path/your_project_name/node_modules/firebase/auth.js) is strongly discouraged, as it poses security risks and may cause issues with minification. See https://github.com/rollup/rollup/wiki/Troubleshooting#avoiding-eval for more details`

Fortunately you can ignore this. When the Firebase team updates I'm sure they will update their use of eval.

There you go, you now have Ionic rc0, Firebase & Lodash in your project. Any questions?