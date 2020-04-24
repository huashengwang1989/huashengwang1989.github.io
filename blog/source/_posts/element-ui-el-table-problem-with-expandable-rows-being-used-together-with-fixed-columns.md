---
title: >-
  Element-UI el-table: Problem with Expandable Rows Being Used Together with
  Fixed Columns
date: 2017-08-13 12:22:25
tags:
- Vue
- Element UI
---

The Table component from [Element](http://element.eleme.io)  (`el-table`) is very versatile in displaying tabulated data, with advanced functions such as fixed columns and expandable rows. However, the two don’t seem work well together.

<!-- more -->

### Problem

When a row is expanded, the expanded area takes the width of the whole table content, including the width of the columns partially or fully hidden behind the fixed columns, if the table is too wide for the screen. More over, if contents like a paragraph is displayed, the “head” and “tail” of each row will be fixed inside the fixed columns, whereas you have to scroll left and right to view the majority of the content. I am not sure whether there is any use case for this; but personally I think that it will make more sense if the expanded area is not affected by the fixed columns, taking the width of the actual el-table DOM element, and always display above other elements/ (/i.e./ the borders of the fixed columns should not cut through the expanded area).

By analyzing the compiled HTML, I realized that the `el-table` realizes the fixed columns by duplicating the whole table body. If you have columns fixed at both left and right sides, the table will be duplicated for three times: one for the fixed columns on the left side, one for the main table contents, and one for the fixed columns on the right side. Unused columns for the `<table>` used for fixed columns are given a `is-hidden` class attribute. This is a genius but a rather expensive operation though.

Unsurprisingly, the expanded rows are also duplicated three times in the case with fixed columns at both sides. It is indeed a `<td>` element with `colspan` equal to the total number of columns.  That explains why one expanded row is shown up at 3 places.

I would like to solve this problem (at least making it suitable for my use case) and try not to directly modifying the source code for `el-table` (as a beginner, I have no confidence of doing so).

### Solution

Eventually, I managed to have some way to work around it. This is not a perfect way though.

What I need to do includes the following:

* Try to bring the expanded areas to top levels without being blocked by the fixed columns (directly manipulating `z-index`?)
* Let the width of the expanded area be the same as the `el-table` `<div>`.

The manipulating of `z-index` works (I simply set it to 999 for demo purpose, and this should work for most cases and will not obstruct is a modal is applied, /e.g./, where there is a Dialog triggered); but getting the correct width is not easy. The use of `<td>` inside a `<table>` making it rather difficult to manipulating the width of that `<td>` alone without affecting other elements inside this `<table>` (or rather, these three `<table>` elements).

I have to think of a workaround. Instead of manipulating the `<td>` itself, it would be easier if I create a wrapper `<div>` inside the `<td>`, and do everything to the wrapper `<div>`. In this way, I only need do the following to the CSS of the `<td>` of the expanded row:

{% codeblock lang:css %}
.el-table__expanded-cell {
	z-index: 999;
	padding: 0;
}
{% endcodeblock %}

The `z-index` is what I have mentioned just now to bring it to front; I also override the `padding`, as originally, it have a default `padding: 20px 50px`. I would rather control it in my wrapper later.

Now we have yet to deal with the width. You will notice that the expanded area are too wide, which you may have to scroll left and right to pan through rows of contents. Since the content is now inside the wrapper (here I name it `expand-area-wrapper`), I just need assign a width to the wrapper. As I have set the `padding` of `.el-table__expanded-cell` to 0, I simply need make it equal to the`clientWidth` of  `el-table` `<div>`.

To achieve this, I need know whether there is a row expanded, what rows are expanded /etc./ Therefore, in `<el-table>`, I have included the following:

```
:data="tableData"
:row-key="getRowKeys"
:expand-row-keys="expands"
@expand="handleExpand"
```

The `tableData` is the data Array I feed to the `el-table`. `getRowKeys` is a function to return the `key` of the row with the input of a row, where each element in `tableData` is an Object of the row.

```javascript
getRowKeys (row) {
	return row.key
},
```

I have a `key` prop assigned to each `row` as a unique identifier of  rows. You may use another property, as long as it is unique to each row.

The `expands` is an Array to hold the keys to the rows that are expanded.

However, the prop `expand-row-keys` is only useful to display the expanded rows based on its value `expands` at initialization. Any show / hide of expanded rows thereafter will not have any effect to `expands`. Therefore, we need manually update the `expands` in `handleExpand` method.

The `expand` event passes two arguments: `row`, and the expand status of that `row`. If it is now expanded, `expanded` is `true`, and vice versa.

Here is the `handleExpand` method for `expand` event:

```javascript
handleExpand (row, expanded) {
	if (expanded) {
		if (!this.expands.includes(row.key))
			this.expands.push(row.key)
		} else {
          this.expands.splice(this.expands.indexOf(row.key), 1)
		}
	}
}
```

When adding a new `row.key` to `expands`, it is recommended to check duplication, as in rare cases, duplicates may occur, which will result in bizarre behavior of the table.

Now we are ready to tackle the problem of the width. As the number of expanded rows changes at the time when an `expand` event is triggered, we can do it in `handleExpand` method.

```javascript
handleExpand (row, expanded) {
	... // continue from previous code block
	this.$nextTick(function () {
		if (this.expands.length > 0) {
			let elems = document.getElementsByClassName('el-table__expanded-cell')
			for (let elem of Array.from(elems)) {
				elem.firstChild.style.width = this.$refs.demo2.$el.clientWidth + 'px'
			}
		}
	})
}
```

Vue `$nextTick` is necessary as we need do it asynchronously.  We get the DOM elements of class  `el-table__expanded-cell`, and select its `firstChild` (or you can do it via its class name `expand-area-wrapper`. As this is a customized wrapper `div` , you may have a different class name.)  Please note that the value returned by `getElementsByClassName` is an `HTMLCollection`, and you need use `Array.from` to make it an array. Then you are able to set the width of the wrapper `div` based on the `clientWidth` of `el-table` (I added a `ref` to `el-table` named “demo2”, so that I can reference it via `$refs.demo2` in the code).

Manipulating the width itself is not sufficient. For now, the wrapper `div` will always stay at the very left. You can add a padding to the wrapper so that the contents inside will look normal though:

```css
.el-table__expanded-cell .expand-area-wrapper {
	padding: 20px 50px;
}
```

However, in the case of a horizontal scroll bar appears, the wrapper `div` will move together with its parent (In fact, it’s the first `el-table__body-wrapper`). We can apply a margin to the left, but it needs change dynamically with the scrollbar position. Therefore, we need add a listener to it.

To make life easier, I created a `computed` variable for the DOM element where the scrollbar applies:

```javascript
computed: {
	scrollTargetElem: function () {
		let targetTable = this.$refs.demo2
		if (this.loaded && targetTable)
			return targetTable.$el.getElementsByClassName('el-table__body-wrapper')[0]
		return
      }
},
```

In methods, I have the following listener:

```javascript
scrollEventHandler (ev) {
	if (this.scrollTargetElem) {
		let elems = this.scrollTargetElem.getElementsByClassName('el-table__expanded-cell')
		for (let elem of Array.from(elems)) {
			elem.firstChild.style.marginLeft = this.scrollTargetElem.scrollLeft + 'px'
		}
	}
},
```

The listener needs to be added or removed when there is an `expand` event. Therefore for the `handleExpand` method:

```javascript
handleExpand (row, expanded) {
	if (expanded) {
		if (!this.expands.includes(row.key)) this.expands.push(row.key)
	} else {
		this.expands.splice(this.expands.indexOf(row.key), 1)
	}
	this.$nextTick(function () {
		let target = this.scrollTargetElem
		if (target) {
			if (this.expands.length > 0) {
				let elems = document.getElementsByClassName('el-table__expanded-cell')
				for (let elem of Array.from(elems)) {
					elem.firstChild.style.width = this.$refs.demo2.$el.clientWidth + 'px'
					elem.firstChild.style.marginLeft = target.scrollLeft + 'px'
              }
              target.addEventListener('scroll', this.scrollEventHandler, false)
			} else {
				target.removeEventListener('scroll', this.scrollEventHandler, false)
			}
		}
	})
}
```

In short, the following has been added to `handleExpand`:

* Initialize the `margin-left` for the wrapper `div`:

```javascript
elem.firstChild.style.marginLeft = target.scrollLeft + 'px'
```

*  Add and remove the listeners depending on whether there are any expanded rows (`expands.length`).

Where for the `scrollEventHandler` :

```javascript
scrollEventHandler (ev) {
	if (this.scrollTargetElem) {
		let elems = this.scrollTargetElem.getElementsByClassName('el-table__expanded-cell')
		for (let elem of Array.from(elems)) {
			elem.firstChild.style.marginLeft = this.scrollTargetElem.scrollLeft + 'px'
		}
	}
},
```

Till now, most problems are solved. There is only one issue. If your table is not set as fixed width (/e.g./ `style="width: 100%;"`), when your browser window is resized, the wrapper `div` width needs change as well, but there is no watcher on the window resize event.

I imported [vue-resize-directive](https://www.npmjs.com/package/vue-resize-directive) as a directive, and created the following `onResize` method:

```javascript
onResize (ev) {
	if (this.expands.length > 0) {
		let elems = document.getElementsByClassName('el-table__expanded-cell')
		let width = ev.clientWidth
		for (let elem of Array.from(elems)) {
			elem.firstChild.style.width = width + 'px'
		}
	}
},
```

In `<el-table>`, add the following:
```
v-resize.debounce="onResize"
```

Now the problem solved, though it is not a perfect solution.  Here is the JSFiddle for it:

{% jsfiddle d004x3pf %}
