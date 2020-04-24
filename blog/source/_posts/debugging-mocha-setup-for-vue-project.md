---
title: Debugging Mocha Setup for Vue Project
date: 2018-09-08 20:04:56
tags: 
- Vue
- Webpack
- Babel
- Mocha
- unit test
- debug
---

I am working on a Vue project with Webpack 4 and Babel 7.

At the point of writing this post, they are Webpack 4.12.0 and Babel 7.0.0-rc.1.

When setting up Mocha with `@vue/test-util`, I have encountered a number of bugs.

After a number of rounds try, my test script in `package.json` becomes:

```
"scripts": {
	"test": "cross-env NODE_ENV=test nyc mocha-webpack --full-trace --recursive --require @babel/register --require ignore-styles --require test/setup.js test/**/*.spec.js"
}
```

Is it necessary or overkill? Will all the additional flags be the solution to problems, or the cause of new errors?

Probably yes.

<!-- more -->

### Debug 1: Filename

```
Cannot find module 'my-path/main.[hash].js?[hash]'
    at Function.Module._resolveFilename (module.js)
```

This is because of the hash after the output filename. Check `webpack.config.js` config `output.filename`. If the hash is within the filename, it works fine.

### Debug 2: Styles

```
.my-class-name[data-v-hash] {
^

SyntaxError: Unexpected token .
    at createScript (vm.js)
```

Check whether you have extract CSS plugins *e.g.* `MiniCssExtractPlugin.loader` enabled for `test` environment. If yes, this may be the cause of the error.

Check `webpack.config.js` config `module.rules` for related settings.

For my project, I uses Vue templates with scss.

- For `vue-loader` (`test: /\.vue$/`), its `options.extractCSS` is always `true` and not causing the problem;
- For `test: /\.css$/`, having `MiniCssExtractPlugin.loader` is not causing the problem;
- For `test: /\.scss$/`, having `MiniCssExtractPlugin.loader` leads to the problem; (More related problems see below)
- For `test: /\.less$/` which I only used a `less` file to overwrite iView's default theme, having `MiniCssExtractPlugin.loader` is not causing the problem.

### Debug 3: No Error but No Tests 1

There are tests but "`0 passing (0ms)`" and "`Tests completed successfully`" with no Error.

Strangely, if have [`ignore-styles`](https://www.npmjs.com/package/ignore-styles) installed and included a `--require ignore-styles` flag, and having `MiniCssExtractPlugin.loader` enabled for `test: /\.scss$/`, Error will not appear but nor the tests can be carried out. `ignore-styles` seems not needed but it would result in non-debuggable result when meets `MiniCssExtractPlugin.loader`.

I have stuck on this problem for quite a long time, as there is no hint on this issue.

### Debug 4: No Error but No Tests 2

It only shows: 

```
 WEBPACK  Compiled successfully in 2459ms

----------|----------|----------|----------|----------|-------------------|
File      |  % Stmts | % Branch |  % Funcs |  % Lines | Uncovered Line #s |
----------|----------|----------|----------|----------|-------------------|
All files |        0 |        0 |        0 |        0 |                   |
----------|----------|----------|----------|----------|-------------------|
```

The latest stable version of `mocha-webpack` which only for Webpack 2 and 3. (at the point of writing, its 1.1.0). If you are using Webpack 4, need install the next version (2.0.0+).

```
npm i mocha-webpack@next --save-dev
```

This is regardless whether you are using `@vue/test-utils` or `vue-test-utils`.


### Debug 5: `TypeError` for All Tests

Tests are properly detected, but all having the following `TypeError`.

```
TypeError: Object(...) is not a function
```

__Solution:__ Please use `@vue/test-utils` instead of `vue-test-utils`.


### Debug 5: `Unexpected token import` 1

```
(function (exports, require, module, __filename, __dirname) { import { shallowMount } from '@vue/test-utils';
                                                              ^^^^^^

SyntaxError: Unexpected token import
```

This may becuase of an additional `--require` flag in you test script:

```
"scripts": {
	"test": "cross-env NODE_ENV=test nyc mocha-webpack --require test/setup.js --require test/**/*.spec.js"
}
```
The second `--require` before the `test/**/*.spec.js` should be removed.

### Debug 6: `Unexpected token import` 2

Based on some people's suggestion, `babel` may be added to the test script as `--require @babel/register`:

```
"scripts": {
	"test": "cross-env NODE_ENV=test nyc mocha-webpack --require @babel/register --require test/setup.js test/**/*.spec.js"
}
```

Sometimes there may be warning for `import` again, which is clearly from `@babel`.

```
(function (exports, require, module, __filename, __dirname) { import _Promise from "@babel/runtime-corejs2/core-js/promise";
                                                                ^^^^^^
  
SyntaxError: Unexpected token import
```

This would need specify the `modules` as `"commonjs"` in `.babelrc` or `babel.config.js`:

```
"env":{
	"test": {
		"presets": {
			/* other presets */
			"modules": "commonjs"
		},
		"plugins": ["istanbul", ...]
	}
}
```

Alternatively, `@babel/plugin-transform-modules-commonjs` plugin can be included:

```
"env":{
	"test": {
		"presets": {
			/* other presets */
			"modules": false
		},
		"plugins": ["istanbul","@babel/plugin-transform-modules-commonjs", ...]
	}
}
```

It should also work if you do both.

### Debug 6: `Cannot find module` or `Unexpected token <`

```
Error: Cannot find module '@components/myVueComponent.vue'
    at Function.Module._resolveFilename (module.js:536:15)
```

`'@components'` is an alias I set in `webpack.config.js` pointing to the folder cotaining my Vue component files.

This error may result from the double `--require` flags (See Debug 5) when `@babel/register` is also required.

```
"scripts": {
	"test": "cross-env NODE_ENV=test nyc mocha-webpack --require @babel/register --require test/setup.js --require test/**/*.spec.js"
}
```

After seeing above error, one natually would change the path for the Vue components path from `@components/myVueComponent.vue'` to something like `'../my-path-to-components/myVueComponent.vue'`. The file you wish to test can be found, but whatever in the `webpack.config.js` would not take effect such as `vue-loader`, and would result in the following error:


```
(function (exports, require, module, __filename, __dirname) { <template>	
                                                              ^

SyntaxError: Unexpected token <
    at createScript (vm.js:80:10)
```


### Final

Eventually, I found out that even I don't have all these fancy flags, everything still works:

```
"scripts": {
    "test": "cross-env NODE_ENV=test nyc mocha-webpack --require test/setup.js test/**/*.spec.js",
}
```

As long as:

- I am using `mocha-webpack` next version (2.0.0) instead of the latest 1.1.0.

- I am importing `@vue/test-utils` instead of `vue-test-utils`.

- Make sure I don't include `MiniCssExtractPlugin.loader` for my `scss` loaders in test environment.

- I should not have an additional `--require` flag before `test/**/*.spec.js` in the test script.

- `--require @babel/register` seems not necessary for my case. However if it is included, make sure the `module` is specified as `"commonjs"` in `.babelrc` or `babel.config.js`, or `@babel/plugin-transform-modules-commonjs` is included as one of the plugins for test `env`.