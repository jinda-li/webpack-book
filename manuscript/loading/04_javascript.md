# Loading JavaScript

Webpack processes ES2015 module definitions by default and transforms them into code. It does **not** transform specific syntax, such as `const`, though. The resulting code can be problematic especially in the older browsers.

To get a better idea of the default transform, consider the example output below (`npm run build -- --mode none`):

**dist/main.js**

```javascript
...
/***/ (function(module, __webpack_exports__, __webpack_require__) {

"use strict";
__webpack_require__.r(__webpack_exports__);
/* harmony default export */ __webpack_exports__["default"] = ((text = "Hello world") => {
  const element = document.createElement("div");

  element.className = "rounded bg-red-100 border max-w-md m-4 p-4";
  element.innerHTML = text;

  return element;
});
...
```

The problem can be worked around by processing the code through [Babel](https://babeljs.io/), a famous JavaScript compiler that supports ES2015+ features and more. It resembles ESLint in that it's built on top of presets and plugins. Presets are collections of plugins, and you can define your own as well.

T> Given sometimes extending existing presets is not enough, [modify-babel-preset](https://www.npmjs.com/package/modify-babel-preset) allows you to go a step further and configure the base preset in a more flexible way.

## Using Babel with webpack configuration

Even though Babel can be used standalone, as you can see in the _SurviveJS - Maintenance_ book, you can hook it up with webpack as well. During development, it can make sense to skip processing if you are using language features supported by your browser.

Skipping processing is a good option primarily if you don't rely on any custom language features and work using a modern browser. Processing through Babel becomes almost a necessity when you compile your code for production, though.

You can use Babel with webpack through [babel-loader](https://www.npmjs.com/package/babel-loader). It can pick up project level Babel configuration, or you can configure it at the webpack loader itself. [babel-webpack-plugin](https://www.npmjs.com/package/babel-webpack-plugin) is another lesser-known option.

Connecting Babel with a project allows you to process webpack configuration through it. To achieve this, name your webpack configuration using the _webpack.config.babel.js_ convention. [interpret](https://www.npmjs.com/package/interpret) package enables this, and it supports other compilers as well.

T> Given that [Node supports the ES2015 specification well](http://node.green/) these days, you can use a lot of ES2015 features without having to process configuration through Babel.

W> If you use _webpack.config.babel.js_, take care with the `"modules": false,` setting. If you want to use ES2015 modules, you could skip the setting in your global Babel configuration and then configure it per environment as discussed below.

{pagebreak}

### Setting up _babel-loader_

The first step towards configuring Babel to work with webpack is to set up [babel-loader](https://www.npmjs.com/package/babel-loader). It takes the code and turns it into a format older browsers can understand. Install _babel-loader_ and include its peer dependency _@babel/core_:

```bash
npm add babel-loader @babel/core -D
```

As usual, let's define a function for Babel:

**webpack.parts.js**

```javascript
const APP_SOURCE = path.join(__dirname, "src");

exports.loadJavaScript = () => ({
  module: {
    rules: [
      {
        test: /\.js$/,
        include: APP_SOURCE, // Consider extracting as a parameter
        use: "babel-loader",
      },
    ],
  },
});
```

Next, you need to connect this to the main configuration. If you are using a modern browser for development, you can consider processing only the production code through Babel. It's used for both production and development environments in this case. Also, only application code is processed through Babel.

{pagebreak}

Adjust as below:

**webpack.config.js**

```javascript
const commonConfig = merge([
  ...
leanpub-start-insert
  parts.loadJavaScript(),
leanpub-end-insert
]);
```

Even though you have Babel installed and set up, you are still missing one bit: Babel configuration. The configuration can be set up using a _.babelrc_ dotfile as then other tooling can use the same.

W> If you try to import files **outside** of your configuration root directory and then process them through _babel-loader_, this fails. It's [a known issue](https://github.com/babel/babel-loader/issues/313), and there are workarounds including maintaining _.babelrc_ at a higher level in the project and resolving against Babel presets through `require.resolve` at webpack configuration.

### Setting up _.babelrc_

At a minimum, you need [@babel/preset-env](https://www.npmjs.com/package/@babel/preset-env). It's a Babel preset that enables the required plugins based on the optional environment definition you pass to it.

Install the preset first:

```bash
npm add @babel/preset-env -D
```

To make Babel aware of the preset, you need to write a _.babelrc_. Given webpack supports ES2015 modules out of the box, you can tell Babel to skip processing them. Jumping over this step would break webpack's HMR mechanism although the production build would still work. You can also constrain the build output to work only in recent versions of Chrome.

Adjust the target definition as you like. As long as you follow [browserslist](https://www.npmjs.com/package/browserslist), it should work. Here's a sample configuration:

**.babelrc**

```json
{
  "presets": [
    [
      "@babel/preset-env",
      {
        "modules": false
      }
    ]
  ]
}
```

If you execute `npm run build -- --mode none` now and examine _dist/main.js_, you will see something different based on your `.browserslistrc` file.

{pagebreak}

Try to include only a definition like `IE 8` there, and the code should change accordingly:

**dist/main.js**

```javascript
...
/***/ (function(module, __webpack_exports__, __webpack_require__) {

"use strict";
__webpack_require__.r(__webpack_exports__);
/* harmony default export */ __webpack_exports__["default"] = (function () {
  var text = arguments.length > 0 && arguments[0] !== undefined ? arguments[0] : "Hello world";
  var element = document.createElement("div");
  element.className = "rounded bg-red-100 border max-w-md m-4 p-4";
  element.innerHTML = text;
  return element;
});
...
```

Note especially how the function was transformed. You can try out different browser definitions and language features to see how the output changes based on the selection.

T> [@babel/preset-modules](https://www.npmjs.com/package/@babel/preset-modules) goes beyond **@babel/preset-env** by fixing bugs in modern browsers. A part of the work has been ported to **@babel/preset-env** as well and can be enabled by setting the `bugfixes` flag to `true`. The preset is useful only for modern browsers!

## Polyfilling features

_@babel/preset-env_ allows you to polyfill certain language features for older browsers. For this to work, you should enable its `useBuiltIns` option (`"useBuiltIns": true` or `"useBuiltIns": "usage"`) and install [@babel/polyfill](https://babeljs.io/docs/usage/polyfill/). You have to include it in your project either through an import or an entry (`app: ["@babel/polyfill", PATHS.app]`). _@babel/preset-env_ rewrites the import based on your browser definition and loads only the polyfills that are needed.

_@babel/polyfill_ pollutes the global scope with objects like `Promise`. Given this can be problematic for library authors, there's [@babel/plugin-transform-runtime](https://babeljs.io/docs/plugins/transform-runtime/) option. It can be enabled as a Babel plugin, and it avoids the problem of globals by rewriting the code in such way that they aren't be needed.

T> [corejs-upgrade-webpack-plugin](https://www.npmjs.com/package/corejs-upgrade-webpack-plugin) makes sure you are using the newest **core-js** polyfills. Using it can help to reduce the size of the output.

W> Certain webpack features, such as _Code Splitting_, write `Promise` based code to webpack's bootstrap after webpack has processed loaders. The problem can be solved by applying a shim before your application code is executed. Example: `entry: { app: ["core-js/es/promise", PATHS.app] }`.

## Babel tips

There are other possible [_.babelrc_ options](https://babeljs.io/docs/usage/options/) beyond the ones covered here. Like ESLint, _.babelrc_ supports [JSON5](https://www.npmjs.com/package/json5) as its configuration format meaning you can include comments in your source, use single quoted strings, and so on.

Sometimes you want to use experimental features that fit your project. Although you can find a lot of them within so-called stage presets, it's a good idea to enable them one by one and even organize them to a preset of their own unless you are working on a throwaway project. If you expect your project to live a long time, it's better to document the features you are using well.

Babel isn't the only option although it's the most popular one. [Buble](https://buble.surge.sh) by Rich Harris is another compiler worth checking out. There's experimental [buble-loader](https://www.npmjs.com/package/buble-loader) that allows you to use it with webpack. Buble doesn't support ES2015 modules, but that's not a problem as webpack provides that functionality.

{pagebreak}

## Babel plugins

Perhaps the greatest thing about Babel is that it's possible to extend with plugins:

- [babel-plugin-import](https://www.npmjs.com/package/babel-plugin-import) rewrites module imports so that you can use a form such as `import { Button } from "antd";` instead of pointing to the module through an exact path.
- [babel-plugin-import-asserts](https://www.npmjs.com/package/babel-plugin-import-asserts) asserts that your imports have been defined.
- [babel-plugin-jsdoc-to-assert](https://www.npmjs.com/package/babel-plugin-jsdoc-to-assert) converts [JSDoc](http://usejsdoc.org/) annotations to runnable assertions.
- [babel-plugin-log-deprecated](https://www.npmjs.com/package/babel-plugin-log-deprecated) adds `console.warn` to functions that have `@deprecate` annotation in their comment.
- [babel-plugin-annotate-console-log](https://www.npmjs.com/package/babel-plugin-annotate-console-log) annotates `console.log` calls with information about invocation context, so it's easier to see where they logged.
- [babel-plugin-sitrep](https://www.npmjs.com/package/babel-plugin-sitrep) logs all assignments of a function and prints them.
- [babel-plugin-transform-react-remove-prop-types](https://www.npmjs.com/package/babel-plugin-transform-react-remove-prop-types) allows you to remove `propType` related code from your production build. It also allows component authors to generate code that's wrapped so that setting environment at `DefinePlugin` can kick in as discussed in the book.
- [babel-plugin-macros](https://www.npmjs.com/package/babel-plugin-macros) provides a runtime environment for small Babel modifications without requiring additional plugin setup.

T> It's possible to connect Babel with Node through [babel-register](https://www.npmjs.com/package/babel-register) or [babel-cli](https://www.npmjs.com/package/babel-cli). These packages can be handy if you want to execute your code through Babel without using webpack.

## Enabling presets and plugins per environment

Babel allows you to control which presets and plugins are used per environment through its [env option](https://babeljs.io/docs/usage/babelrc/#env-option). You can manage Babel's behavior per build target this way.

`env` checks both `NODE_ENV` and `BABEL_ENV` and adds functionality to your build based on that. If `BABEL_ENV` is set, it overrides any possible `NODE_ENV`.

Consider the example below:

**.babelrc**

```json
{
  ...
  "env": {
    "development": {
      "plugins": [
        "annotate-console-log"
      ]
    }
  }
}
```

Any shared presets and plugins are available to all targets still. `env` allows you to specialize your Babel configuration further.

{pagebreak}

It's possible to pass the webpack environment to Babel with a tweak:

**webpack.config.js**

```javascript
module.exports = mode => {
leanpub-start-insert
  // You could use NODE_ENV here as well
  // for a more generic solution.
  process.env.BABEL_ENV = mode;
leanpub-end-insert

  ...
};
```

T> The way `env` works is subtle. Consider logging `env` and make sure it matches your Babel configuration or otherwise the functionality you expect is not applied to your build.

## Setting up TypeScript

Microsoft's [TypeScript](http://www.typescriptlang.org/) is a compiled language that follows a similar setup as Babel. The neat thing is that in addition to JavaScript, it can emit type definitions. A good editor can pick those up and provide enhanced editing experience. Stronger typing is valuable for development as it becomes easier to state your type contracts.

Compared to Facebook's type checker Flow, TypeScript is a safer option in terms of ecosystem. As a result, you find more premade type definitions for it, and overall, the quality of support should be better.

[ts-loader](https://www.npmjs.com/package/ts-loader) is the recommended option for TypeScript. One option is to leave only compilation to it and then handle type checking either outside of webpack or to use [fork-ts-checker-webpack-plugin](https://www.npmjs.com/package/fork-ts-checker-webpack-plugin) for the purpose to handle checking in a separate process.

You can also compile TypeScript with Babel through [@babel/plugin-transform-typescript](https://www.npmjs.com/package/@babel/plugin-transform-typescript) although this comes with small [caveats](https://babeljs.io/docs/en/next/babel-plugin-transform-typescript.html#caveats).

T> If you have set up TypeScript to your project, you can write your configuration in TypeScript by naming the configuration file as **webpack.config.ts**. Webpack is able to detect this automatically and run it correctly. Remember to install [ts-node](https://www.npmjs.com/package/ts-node) as webpack requires that to execute the configuration.

T> You can find types for webpack behind [@types/webpack](https://www.npmjs.com/package/@types/webpack) and [@types/webpack-env](https://www.npmjs.com/package/@types/webpack-env). Webpack 5 includes TypeScript support out of the box.

T> There's a [TypeScript parser for ESLint](https://www.npmjs.com/package/typescript-eslint-parser). It's also possible to lint it through [tslint](https://www.npmjs.com/package/tslint).

## Setting up Flow

[Flow](https://flow.org/) performs static analysis based on your code and its type annotations. You have to install it as a separate tool and then run it against your code.

If you use React, the React specific Babel preset does most of the work through [babel-plugin-syntax-flow](https://www.npmjs.com/package/babel-plugin-syntax-flow). It can strip Flow annotations and convert your code into a format that is possible to transpile further.

[flow-runtime](https://www.npmjs.com/package/flow-runtime) allows runtime checks based on our Flow annotations. The approaches complement Flow static checker and allow you to catch even more issues.

T> [flow-coverage-report](https://www.npmjs.com/package/flow-coverage-report) shows how much of your code is covered by Flow type annotations.

{pagebreak}

## Conclusion

Babel has become an indispensable tool for developers given it bridges the standard with older browsers. Even if you targeted modern browsers, transforming through Babel is an option.

To recap:

- Babel gives you control over what browsers to support. It can compile ES2015+ features to a form the older browser understand. _@babel/preset-env_ is valuable as it can choose which features to compile and which polyfills to enable based on your browser definition.
- Babel allows you to use experimental language features. You can find numerous plugins that improve development experience and the production build through optimizations.
- Babel functionality can be enabled per development target. This way you can be sure you are using the correct plugins at the right place.
- Besides Babel, webpack supports other solutions like TypeScript or Flow. Flow can complement Babel while TypeScript represents an entire language compiling to JavaScript.
