---
title: Global Custom Filters for Vue
date: 2017-07-19 21:01:26
tags: Vue
---

It is very convenient to use filters in Vue. Custom filters can be defined in the `filters`, as below in the `export default` of a Vue single file component:

```javascript
export default{
	data () {
		return {
		}
	},

	filters: {
		//my filters
	},

	methods: {
	},
}
```

<!-- more -->

However, `filters` defined in this way can only be easily accessible by the component containing these filters (*i.e.*, in the same .vue file). In many cases, we would like to have some filters available globally to be used by all Vue components inside this project. (And also, I do not wish to write `import myFilters from xxx` inside every Vue component.)

In short, I would like to create a single `filters.js` file to host all my filters, ideally with a similar structure to the familiar Vue single file component, and then import it into `main.js` .

It is easy to create a custom filter with the `Vue.filter()` method. The following post in the Vue Forum has provided a simple solution:
[Custom filters organization - Vue Forum](https://forum.vuejs.org/t/custom-filters-organization/3738/3)

```javascript
export default{
	create: function(Vue) {
		Object.keys(this.filters).forEach(function (filter,k){
			Vue.filter(filter, this.filters[filter])
		}.bind(this));
	},

	filters: {
		filterBar: function (val) {
			return 'Bar_' + val
		},
		filterFoo: function (val) {
			return 'Foo_' + val
		},
	},

}
```

Inside `main.js`:
```javascript
//import filters
import filters from 'yourLocation/filters.js'
filters.create (Vue);
```

Please be noted that the `.bind(this)` is necessary for the function inside the `forEach` loop, and with `.bind(this)`,  `this` will be pointed to the Vue component that has called these filters. However,  `this` will no longer be usable by filter functions to call data or methods, even they are in the same file. To solve this problem, we have to record `this` inside the `create` function:

```javascript
var _this = this;
var _methods = {};
var _data = {};

export default{

	create: function(Vue) {

		Object.keys(this.filters).forEach(function (filter,k){
			Vue.filter(filter, this.filters[filter])
		}.bind(this));

		_this = this;
		_methods = this.methods;
		_data = this.data();

	},

	data () {
		return {
			strBar : 'Bar_',
			strFoo : 'Foo_',
		}
	},

	filters: {

		filterBar: function (val) {
			return _methods.foobar(_data.strBar , val)
		},

		filterFoo: function (val) {
			return _methods.foobar(_data.strFoo , val)
		},

	},

	methods: {

		foobar (a, b){
			return a + b;
		}

	},
},
```

Now, we can use `_this`, `_methods`, `_data` to write our filter functions.
