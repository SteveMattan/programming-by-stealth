# PBS Tidbit 6 of Y — A Real-World Webpack Case Study

In the main series we recently dedicated two instalments ([PBS 138](./pbs138) & [PBS 139](./pbs139)) to using [Webpack](https://webpack.js.org/) to bundle a website or web app. In the instalments we used a very simplistic example to help keep things clear. The example worked, but it left me wondering what it would be like to migrate an existing real-world web app to Webpack. 

I want to make some improvements to [this-ti.me](https://this-ti.me) in the coming months, and I don't want to put any time into a non-webpacked project anymore, so I decided to port this existing app to Webpack as a real-world case study. In the main series we never aim to cover any of our topics exhaustively, instead, we cover the basics in the expectation that that will arm you all with enough knowledge to learn the specific advanced features you need from the documentation and other online resources. 

With that in mind I fully expected to have to learn at least some new Webpack skills to get the site working well, and that's exactly what happened. In this tidbit I'll share my journey, and what I learned along the way.

## Matching Podcast Episode

Listen along to this instalment on [episode 743 of the Chit Chat Across the Pond Podcast](https://www.podfeet.com/blog/2022/09/ccatp-743/).

<audio controls src="https://media.blubrry.com/nosillacast/traffic.libsyn.com/nosillacast/CCATP_2022_09_17.mp3?autoplay=0&loop=0&controls=1">Your browser does not support HTML 5 audio 🙁</audio>

You can also <a href="https://media.blubrry.com/nosillacast/traffic.libsyn.com/nosillacast/CCATP_2022_09_17.mp3" >Download the MP3</a>

## The Original Code

The code before I started the migration was pretty much unchanged since it was developed as my sample solution to the challenge set at the end of [instalment 96](./pbs96), and described in [instalment 100](./pbs100). The code and its entire history is [published on GitHub](https://github.com/bartificer/this-ti.me).

The entire codebase was self-contained within a single `index.html` file. All custom CSS and JavaScript was embedded in `<style>` and `<script>` tags, all the [Mustache templates](https://github.com/janl/mustache.js) embedded in `<script type="html">` tags, and all 3rd-party CSS, JavaScript, and web fonts loaded from CDNs.

Before starting the code was already managed in Git, with the entire repo contents published as a website using GitHub pages.

## Preparation — Re-Factor to Separate Files

To ensure I could always roll back my changes, the very first thing I did was switch to a new branch named `chore-migrateToWebpack`.

To use Webpack I needed to switch the repo from publishing the entire thing as a website to publishing just a single folder, docs, so the first step was to move `index.html` to `src/index.html`.

For Webpack to be able to bundle the code the CSS and the JavaScript needed to come out of the HTML file. To that end I made the following changes:

1. I moved all the custom CSS from `src/index.html` to `src/index.css` and replaced the `<style>` tag with a `<link rel="stylesheet">` tag.
2. I moved all my own JavaScript from `src/index.html` to `src/index.js` and updated the `<script>`  tag to use an `src` attribute to load the code from the newly created file.

So, the starting point for the migration to Webpack was as follows:

1. `src/index.html` containing the HTML markup, the Mustache templates, and importing my own code from `src/index.css` & `src/index.js`, and all third-party CSS, JavaScript and web fonts from various CDNs.
2. `src/index.css` containing my CSS code
3. `src/index.js` containing my JavaScript code

## Initialise NPM & Webpack

To get either Webpack itself, or any of the dependencies currently loaded from CDNs, the first step is to turn the repo into a NodeJS package:

```sh
npm init
```

Next, install Webpack itself:

```sh
npm install --save-dev  webpack webpack-cli copy-webpack-plugin css-loader style-loader
```

When it came to configuring Webpack I initially started with the config from the end of the worked example in [PBS 139](./pbs139), but I very quickly made a very significant change. Before I explain the big change, here's my final config:

```js
// Needed hackery to get __filename and __dirname in ES6 mode
// see: https://stackoverflow.com/questions/46745014/alternative-for-dirname-in-node-js-when-using-es6-modules
import path from 'node:path';
import { fileURLToPath } from 'node:url';

const __filename = fileURLToPath(import.meta.url);
const __dirname = path.dirname(__filename);

// import webpack's standard functionality
import webpack from 'webpack';

// import the Webpack copy plugin
import CopyPlugin from 'copy-webpack-plugin';

// export the Webpack config
export default {
    entry: {
        head: './src/index-head.js', // will be imported inside the header — CSS only
        body: './src/index-body.js', // will be imported at the very bottom of the body — JavaScript only
    },
    output: {
        path: path.resolve(__dirname, 'docs'),
        filename: 'bundle-[name].js',
        clean: true // remove all files from the output not generated by the current build
    },
    module: {
        rules: [
            {
                test: /\.css$/,
                use: [
                    'style-loader',
                    'css-loader'
                ]
            },
            {
                test: /\.mustache$/,
                type: 'asset/source'
            },
            {
                test: /\.(woff|woff2|eot|ttf|otf|svg)$/i,
                type: 'asset/resource',
                generator: {
					filename: 'webfonts/[hash][ext][query]'
				}
            }
        ]
    },
    plugins: [
        new CopyPlugin({
            patterns: [
                { from: "src/index.html", to: "index.html" }
            ],
        }),
        new webpack.ProvidePlugin({
            $: 'jquery',
            jQuery: 'jquery',
            moment: 'moment'
        })
    ]
};
```

For the most part this config does things much like we did in PBS 139, with four notable exceptions, only one of which I want to focus on now, the others I'll explain later.

### Separate Head & Body Bundles

In all our PBS examples up to and including PBS 139 we've used Webpack to bundle all our JavaScript, CSS, and dependencies into a single bundle — one file which we then included in our HTML with a `<script src=''>` tag. This config doesn't do that, it creates **two** bundles, one intended to be imported in the `<head>`, and one to be imported at the bottom of the `<body>`.

Why would I want to do that? If you cast your mind right back to when we stated to use CDNs, you may vaguely recall that we included the CSS from the CDNs in the `<head>`, and the JavaScript at the bottom of the `<body>`. We had a good reason to do that, and it was to give users the best experience on slow network connections.

Anything imported into the `<head>` will be fetched by the browser before it renders and of the page, and anything loaded into the bottom of the `<body>` won't be fetched until the rest of the body has been rendered. The page won't actually function until everything is loaded, so why do we care about the order? It's to do with giving the user the sense that something is actually happening. Think about these three scenarios:

1. We import all our dependencies in the `<head>` — this means the page remains entirely blank until the very end of the load time when suddenly it all appears at once and functions immediately on rendering. Users are really impatient, so they'll probably assume your site/app is broken as they  stare at a totally blank page for a few seconds!
2. We import all our dependencies at the bottom of the `<body>` — this means the HTML will start to render almost instantly, but without any of the styles applied, and with none of the web fonts loaded, it will then flicker and change and jump about as the CSS loads in, and none of the UI will function until the JavaScript loads in. This the most reposoosive option, but it's really jarring, and looks amateurish at best!
3. The compromise solution is to load the CSS in the `<head>`, and the JavaScript in the `<body>` — this will slow down the initial rendering a little, but everything that renders will be rendered with its styles applied, so it shouldn't jump around and change disconcertingly. Again, the UI won't actually function until the JavaScript gets loaded in. 

 The third option is considered best practice, hence my need for two bundles. One hot topic of debate though is where web fonts and glyficons should be loaded, with the CSS, or with the JavaScript? I can see the argument for loading the icons at the end, but I can't see why you would want the fonts changing as the page loads and more than the sizes and the colours! I decided to load the icons, the fonts, and the CSS in the `<head>`, and just the JavaScript at the bottom of the `<body>`.
 
 So, how do we output two bundles? By making two entry points! I chose to use `index-head.js` and `index-body.js`. In our previous configs we defined the entry point as a single string, in this config I define the entry point as a dictionary with two keys `head` and `body`, but I could have named them anything I liked. These keys become the *names* for the entry points elsewhere in the config. This is the code snippet that defines the two entry points:
 
 ```js
 export default {
    entry: {
        head: './src/index-head.js', // will be imported inside the header — CSS only
        body: './src/index-body.js', // will be imported at the very bottom of the body — JavaScript only
    },
    // …
  }
 ```
 
 If we have two entry points, don't we also need two distinct bundle names to map them to? We do indeed, but we don't achieve this with another dictionary, instead, we use the templating functionality supported in some Webpack config values. The placeholders are wrapped in square brackets.
 
 In this case, we need to introduce the template syntax into the `filename` key in the `output` dictionary, instead of outputting to `bundle.js`, we output to `bundle-[name].js`, here's the fill snipped:
 
 ```js
 export default {
    // …
    output: {
        path: path.resolve(__dirname, 'docs'),
        filename: 'bundle-[name].js',
        clean: true // remove all files from the output not generated by the current build
    },
    // …
 ```
 
Since I named my entry points `head` & `body`, this template will produce bundles named `bundle-head.js` & `bundly-body.js`.
 
 ### Automatically Clean the Output Dir
 
 This is a good opportunity to draw your attention to a very useful output option `clean: true` — this tells Webpack to remove everything from the output dir that's not part of the most recent build. This prevents leftovers from past experiments getting accidentally left behind.

## Find & Install Each Dependency

Before I committed myself too much, I made sure each and every dependency I was including via a CDN was available from NPM, thankfully, they all were 😅

### Installing Specific Version of Dependencies

So far, we've always used NPM at the point in time when we first need a module, so we've always been happy with NPM's default behaviour of installing the latest versions of modules.

We know that NPM uses SemVer to ensure you don't automatically update between major versions and risk breaking changes, but the initial install is always the latest released version, and it's that major version that gets locked in as the project ages.

I wrote this site with Bootstrap 4, and re-implementing it in Bootstrap 5 is too big of a task to simply mix in with the migration to Webpack. By default, NPM would give me Bootstrap 5, so how do I tell it Bootstrap 4? Simple, post-fix the package name with an `@` symbol and as much of the version number as you want to specify. I started with simply:

```sh
npm install --save bootstrap@4
```

I needed to do something similar for Font Awesome and JS cookie:

```sh
npm install --save @fortawesome/fontawesome-free@5  js-cookie@2
```

By putting only one number after the `@` I'm in effect saying *'give me the most recent minor and patch version under this major version'*. You can of course be more specific, and specify a major an minor version, or even a major, minor, and patch version. I ended up re-installing Bootstrap to a very specific version because I ran into an odd bug when I let it go to the latest Bootstrap 4:

```sh
npm remove bootstrap
npm install --save bootstrap@4.2.1
```

The reason for the weirdness became clear later — I was moving each dependency one-by-one and re-building and testing in between, and at one point I had the Bootstrap CSS from NPM, and the Bootstrap JS from the CDN, and they were at different versions. As soon as I forced NPM to use the identical version to the one I was getting from the CDN sanity returned!

One last little gotcha in the dependencies is that when using a CDN you use a pair of `<script>` tags to get MomentJS with timezone support, with NPM you get both together in one package:

```sh
npm install --save moment-timezone
```

### Globally Loaded Modules (to Handle Peer Dependencies)

Some third party code is not intended to be used alone, but to augment another piece of code. For example, I used two Bootstrap plugins on this site, [Tempus Dominus](https://getdatepicker.com/5-4/) to provide the date & time pickers, and [Bootstrap 4 Autocomplete](bootstrap-4-autocomplete) to provide the auto-complete functionality on the timezone text box.

When using CDNs you simply add the tag to import the plugin after the tag(s) to import the code its extending. When using pure NodeJS code the plugin will list the thing it extends as a *peer dependency*, meaning it expects you to install the extended code into your package as a dependency. This is a simple and painless process, but things get a little messier when you try to bundle code with peer dependencies!

As soon as I tried to port the first of my plugins from CDN to Webpack I started getting errors in the console about jQuery and MomentJS being missing but required. These modules were installed, so they were available to pure NodeJS code, but there were scoping issues inside the bundled code.

I didn't panic, because I knew this was a common thing to need to do, so off I went to the Googles. Sure enough, this is such a commonly needed feature that Webpack includes the built-in [Provide](https://webpack.js.org/plugins/provide-plugin/) plugin for solving this issue. You use the plugin to map NodeJS packages to specific global variable names that will be available to all code in your bundles. This is the relevant snippet from `webpack.config.js`:

```js
export default {
    // …
    plugins: [
        // …
        new webpack.ProvidePlugin({
            $: 'jquery',
            jQuery: 'jquery',
            moment: 'moment'
        })
    ]
};
```

## Some Simple Refactorings

At this stage the code was working again, but since I had it open and had my head in that space I figured I should make a few additional tweaks to put it on a more stable footing going forward.

The first thing I did was re-factor my Mustache templates from `<script type="text/html">` tags to Webpack resources (like we did in [PBS 139](./pbs139)).

Next, I replaced the abandoned `is.js` type checking library with its actively maintained port `is-it-check`. First, I replaced the NPM packages:

```sh
npm remove is_js
npm install --save is-it-check
```

Then I updated the `import` statement in `src/index-body.js` from:

```js
import is from 'is_js';
```

To:

```js
import is from 'is-it-check';
```

At this stage I'd figured out why I was getting odd behaviour when I first tried to use the latest Bootstrap 4 release, so I used `npm upgrade` to roll all packages forward to the latest release for their major version:

```sh
npm outdated
npm update jquery bootstrap
```

The Bootstrap update did introduce a very minor visual glitch under the tabs, probably because I shouldn't be using `h4` inside a tag, but it was easily fixed by explicitly setting the bottom margin to zero by adding the `mb-0` Bootstrap class.

## Refactor to Shrink the Bundles

When you're using Webpack to distribute a re-usable library (as described in [instalment 137](./pbs137)) getting all your code into a single bundle file is literally the whole point of the exercise! That's not true when you're bundling a web page or app, the point is to produce code that's easy for you to manage, and that doesn't depend on other people's servers to run. Bundling all code into monolithic bundles can be wasteful — if you only use 5 icons from your icon set, having them all stuffed into one massive `.js` file by marking them as inline assets will really slow you page/app down over slow network connections. In this case it would be better to allow Webpack bundle the resources as separate files that will only be fetch if and when they're needed.

When it comes to breaking up your bundles, there is no definable best practice, it's one of those dark arts where each case it different and over time you get a feel for what works well and what doesn't in very specific scenarios. Webpack's documentation dedicates [a page to the various options for splitting bundles](https://webpack.js.org/guides/code-splitting/).

By having multiple entry points I'd already shrunk my bundles a bit, but `bundle-head.js` was still over 4mb, which struck me as very large. I was pretty sure it was the inlined web fonts and glyphicons that were the cause, so I changed them from [inline assets](https://webpack.js.org/guides/asset-modules/#inlining-assets) to [resource assets](https://webpack.js.org/guides/asset-modules/#resource-assets) and specified they should be bundled into a folder named `webfonts`.

To do this I simply needed to update the relevant rule in `webpack.config.js` from:

```js
{
  test: /\.(woff|woff2|eot|ttf|otf|svg)$/i,
  type: 'asset/inline',
}

```

To:

```js
{
  test: /\.(woff|woff2|eot|ttf|otf|svg)$/i,
  type: 'asset/resource',
  generator: {
    filename: 'webfonts/[hash][ext][query]'
  }
}
```

If I'd been OK with the generated files appearing in the root of the output folder named for their hashes I could have omitted the `generator` dictionary, but I'm a stickler for keeping things tidy, so I added the `generator` attribute so I could specify a filename template for the matching file types.

## Final Thoughts

I certainly learned a lot migrating this existing app to Webpack. I learned a few important lessons in the process:

1. It's easier to use Webpack from the start than to retro-fit it later, so I'll be doing that in future.
2. The Webpack docs are very good!
3. Webpack is popular enough and has a big enough community that you'll find lots of helpful [Stack Overflow answers](https://stackoverflow.com/questions/tagged/webpack) and blog posts to help you address your problems.
4. Webpack 5 is new enough that you need to check all the answers/posts to be sure they're not for Webpack 4!

I hope that by sharing this real-world experience I'll help to push you over the edge into pulling the trigger and migrating your existing web apps and sites to Webpack 🙂

 - [← Tidbit 5 — Tips for the Vacationing Programmer](tidbit5)
 - [Index](index)
