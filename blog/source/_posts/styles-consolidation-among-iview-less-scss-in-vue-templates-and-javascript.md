---
title: 'Styles Consolidation among iView less, scss in Vue templates and JavaScript'
date: 2018-09-06 00:32:50
tags:
- iView
- less
- sass
- scss
- Webpack
- loader
- stylesheets
- Vue
---
### 0x00: SCSS Variables

In a Vue template file, say `sample.vue`, I can define variables in the `scss` styles. In this way, I can reuse it across the `scss` in this template file:

```html
<template>
    <!-- my virtual DOMs -->
</template>
<style lang="scss">
    $my-color: #fcfcfc;
    .my-class{
    	color: $my-color;
    }
</style>
<script>
    // my JavaScript
</script>
```
<!-- more -->

If I want to have some style variables shared across all my template files, I can create an `scss` file, say `common.styles.scss`, and put it somewhere. Then at the style section in every template file, I can `@import` this file so that the variables there are usable.

A sample `common.styles.scss`:

```scss
$my-color-a: #fcfcfc;
$my-color-b: #f8f8f9;
$my-color-c: #ffffff;
```

### 0x01: Global SCSS Variables to Vue Template Styles

This sounds good, but has one big problem: I have already had almost 100 Vue template files. It is quite silly that I have to write the same line for existing and any future `.vue` files.

Yes, there is an existing webpack loader that does exactly the job: [`sass-resources-loader`](https://github.com/shakacode/sass-resources-loader) (webpack 4 OK!). You may install it via `npm`:

```bash
$ npm i sass-resources-loader --save-dev
```

and then in your webpack config file (*e.g.* `webpack.config.js`):

```javascript
var style_loaders = {
	'css': 'vue-style-loader!css-loader',
	'scss': [
		'vue-style-loader',
		'css-loader',
		'sass-loader',
		{
			loader: 'sass-resources-loader',
			options: {
				resources: path.resolve(__dirname, 'my-path-to-styles/common.styles.scss'),
			},
		},
	],
}


module.exports = {
	// ...
	module: {
		rules: [
		    {
	            test: /.vue$/,
	            use: [
	                {
	                	loader: 'vue-loader',
	                	options: {
	                		extractCSS: true,
	                		loaders: style_loaders
	                	}
	                }
	            ]
	        },
	        {
				test: /\.scss$/,
                use: [ MiniCssExtractPlugin.loader ].concat(style_loaders.scss),
			},
	        { 
	        	// other rules...
	        }
		]
	},

	// ... 
}


```

`sass-resources-loader` will directly take whatever text inside the specified `scss` file (`common.styles.scss` here) to the top of all the `<style lang="scss"></style>`, even non-scss scripts or random texts. In this way, if you have done the `scss` file properly, you will be able to use this global `scss` style variables across Vue template file. If you have any "illegal" contents in your `scss` file, it would be other loader's job to throw an `Error`.


### 0x02: I Want Them in LESS and JS Too

However, the above `sccs` globals cannot be accessed by the JavaScript:  

```javascript
<template>
    <my-component :style="myInlineStyle"></my-component>
</template>
<script>
    export default {
    	//...
    	computed: {
    		myInlineStyle: function(){
    			return {
 				    'backgroundColor': ??? // I want to access to the global style variables here too
    			} 
    		}
    	}
    }
</script>
```

After searching, I found another webpack loader that seemed to do the job: [`node-sass-json-importer`](https://www.npmjs.com/package/node-sass-json-importer). It can import `.json` or `.json5` to `scss` files via the `@import` syntax.

`JSON` would be quite easy to be imported to JavaScript, *e.g.* store the values as a global object in `main.js`. However, writing `JSON` is troublesome; you may wish to have another level above, such as a `.js` file output to `.json`.

PS: by reading the source code, `node-sass-json-importer` seems possible to parse exported Javascript object; but it will have a check of the filetype at the beginning, and will not proceed if the file url at `@import` does not match `/.json5?$/`. This not only makes it impossible to import a `.js` file, also it will have problem for styles in vue template, which will be detailed in next part.

Meanwhile, if some other UI Frameworks are used such as [iView](https://www.iviewui.com), they may use `less` for the styles. iView has supplied a list of `less` variables, which you may customize them to overwrite iView's default styles. It would be best if there is something that can kill both with one stone.

[`js-to-styles-var-loader`](https://www.npmjs.com/package/js-to-styles-var-loader) seems to be able to do the job based on its name and docs. By first look, it have two good points:

- It can directly get the output values from JavaScript modules;
- and it can add these values to both `less` and `sass`.

With this loader, a line at the top of the `less` and 'sass' files will do: 

```
require('my-path-to/shared.styles.js')
```

However, it doesn't work in real life.

Looking through the source code, I discovered that there are several places in the code causing the problem:

- This loader gets the file path by looking for the `require` line, parsing it with a `regex` to get the relative path, and using `path.join` to join this relative path to `webpackContext.context`. This means, `'./'`, `'../'` in the path would not work, unless you put the style files you wish to import the js module at the root folder. Moreover, the `regex` itself prevents symbols like `@`, which many may use it for webpack path alias.

- This loader will also check the file extension. If it is not `.scss`, `.sass` or `.less`, an `Error` will throw. As `sass-resources-loader` will only blindly copy whatever in these style files to Vue template styles, the `require` line will also be copied to all Vue templates -- unless you do the translation at its `options.resources` (too challenging...). Having `require` line in Vue templates means that first, it would be a `.vue?type=styles...` and would not pass the file type check; second, the relative path in the `require` line will not hold as now it is relative to each `.vue` file and they are in different nested folders.

### 0x03: Rewrite the Loader

1. Change the `requireReg` to the following so that it can supports various path formats and the restriction on file types is lifted:

```javascript
var requireReg = /require\s*\((["'])([@\/]?[\w.\-\/]+)(?:\1)\)((?:\.[\w_-]+)*);?/igm
```

PS: I only updated it to my own needs. If you need support for `'~'` *etc.*, please update it by yourself.

2. Update the `getPreprocessorType` function with a param named `'enforceType'`. Previously the file type is checked here. Now if `'enforceType'` is specified, the check on the file extension will be skipped.

```javascript
getPreprocessorType ( { resource, enforceType } ={}) {
    if (['sass','scss','less'].includes(enforceType)){
        return enforceType === 'less' ? 'less' : 'sass';
    }
    // ...
}, 
```

3. Use `webpackContext.resolve` instead of `path.join` in `mergeVarsToContent` function. As `webpackContext.resolve` is async, `'string-replace-async'` is used in place of `String.prototype.replace`. You need `npm install` this module first. This function will then need return a `Promise` instead.

__Update__: Found that `webpackContext.resolve` is not able to handle the case when `relativePath` is only the filename. It will have the error in the callback. In this case, I will simply use the original `path.join` to handle `modulePath`.

```bash
$ npm install --save string-replace-async
```

And here's the code part:

```javascript
const stringReplaceAsync = require('string-replace-async');
//...
const operator = {
	// ...
	mergeVarsToContent (content, webpackContext, preprocessorType){
	    const replacer = function(match_require_string, quote, relativePath){
	        return new Promise((resolve,reject) => {
	            function handlePathResolve(err, modulePath){
	                if (!!err){
                        // If only the file name, webpackContext.resolve would throw an error.
                        // in this case, just join the path as the original.
                        modulePath = path.join(webpackContext.context, relativePath);
                    }
	                const varData = this.getVarData(modulePath, this.propDeDot(property));
	                webpackContext.addDependency(modulePath);
	                const style_vars = this.transformToStyleVars({
	                    type: preprocessorType,
	                    varData
	                });
	                resolve(style_vars)
	            }
	            webpackContext.resolve(webpackContext.context, relativePath, handlePathResolve.bind(this))
	        })
	    };
	    return stringReplaceAsync(content, requireReg, replacer.bind(this))
	},
	// ...
}
// ...
```

4. Tell webpack that this is an async loader. Allow it to pass the processor you wish to use via the options with `options.useProcessor`, which will become the `'enforceType'` for `operator.getPreprocessorType` function.

```javascript
const loader = function (content) {
    const webpackContext = this;
    const resource = operator.getResource(webpackContext);
    const options = loaderUtils.getOptions(webpackContext);
    const enforce_preprocessor_type = !!options && !!options.useProcessor ? options.useProcessor : null;
    const preprocessorType = operator.getPreprocessorType({ 
    	resource, 
    	'enforceType': enforce_preprocessor_type 
    });
    var callback = webpackContext.async(); // Async loader
    operator.mergeVarsToContent(content, webpackContext, preprocessorType)
    .then(res => {
        callback(null, res);
    })
    .catch(err => {
        callback(err);
    })
};

exports.default = loader;
```

I've forked `js-to-styles-var-loader` and included these updates temporarily in the [dev branch](https://github.com/huashengwang1989/js-to-styles-var-loader/tree/dev) in my channel. You may find the `index.js` file there. Nonetheless I have not tested yet, nor the test scripts and demos are updated. So use it at your own risk.


### 0x04: Use Custom Loader in Webpack

Webpack can add custom loader. You may add the following line: 

```javascript
module.exports = {
	mode: 'development', // or 'production'
	// ...
	resolveLoader: {
		alias: {
		    'my-custom-loader': path.join(__dirname, 'path-to-loader/my-custom-loader.js')
		}
	},
	// ...
}
```

Then you may use this loader in the same way as other loaders:

```javascript
{
	loader: 'my-custom-loader',	
	options: {
		useProcessor: 'scss',
	}
}
```

If `sass-resources-loader` is used, this should be put above `sass-resources-loader`, as loaders are loaded from the last to the first. For the `style_loaders` mentioned before, now becomes:

```javascript
var style_loaders = {
	'css': 'vue-style-loader!css-loader',
	'scss': [
		'vue-style-loader',
		'css-loader',
		'sass-loader',
		{
			loader: 'my-custom-loader',	// put here!
			options: {
				useProcessor: 'scss',
			}
		},
		{
			loader: 'sass-resources-loader',
			options: {
				resources: path.resolve(__dirname, 'my-path-to-styles/common.styles.scss'),
			},
		},
	],
}
```

Hope this post helps.
