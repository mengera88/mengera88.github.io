---
title: 用regular编写带有checkbox的树形组件
date: 2017-03-14 10:01:25
tags: regular javascript 前端
---

最近由于项目需求，要写一个多层级类似树状目录结构，并且每一个层级都有checkbox勾选,类似下图:

![alt 层级目录](/images/treeList.png)

这个层级图和RegularUI的树状图需求上还是有差别的，RegularUI的树形组件只支持单选，并且选择形式仅仅是高亮某个选中层，而本篇文章所说的树形组件需要支持多选，多选的形式可以是加checkbox，也可以在层级后面显示勾勾表示选择。

## 设计思路

大家都知道，类似这种树形组件，最好的办法就是递归了，其实单纯的不带checkbox树形组件递归并不是很难，首先设计一个单层结构的，作为一个子组件，再设计父组件，用简单的逻辑来判断是否递归自己，由于业务需要，我自己又添加了一个顶层的组件，用来综合处理所有的逻辑。

千言万语不如几行代码来的清楚：

**treeListItem.html** 模板文件

```html
<div class="m-checklist-item" r-hide={parent && !parent.open}>
    <i class="icon {source.open ? 'icon-chevron-down' : 'icon-right'}" on-click={this.openChange($event)}></i>
    <div class="u-checkbox u-checkbox-inline">
        <input id={source.key} type="checkbox" r-model={source.isChecked} on-change={this.check($event, parent)} disabled={source.isDisabled}>
            <label for={source.key} r-hide={!source.ischeckboxShow}></label>
            <i class={source.iconClass}></i>
            <span title={source.name} class="f-toe" style="display: inline-block;width: 100%;vertical-align: middle;">{source.name}</span>
    </div>

</div>
```

该模板对应的js文件 **treeListItem.js**
```javascript
/**
 * data:source,parent(对应的父亲),source
 * source = {
 *   isChecked: false,     //默认没有check
 *   isDisabled: false,    //默认不禁止
 *   name: '',             //显示的内容
 *   key: {Number} or {String}         //唯一标识,
 *   open: true,           //是否展开
 *   iconClass: 'icon icon-table', //checkbox和name之间插入的图标
 *   disabledNum: 0,           //无法选择的项目个数
 *   children: [{}],    //该条checklist对应的孩子,可无限循环下去
 * }
 */
define([
    'base/util',
    'base/event',
    'regular!./treeListWithCheckItem.html',
    'text!./treeListWithCheck.css',
    'util/template/tpl',
    '{pro}global/util.js',
    '{components}tooltip/tooltip.js'
], function(_u, _v, _tpl, _css, _t, _util) {
    _t._$parseUITemplate('<textarea name="css" id="css-box">' + _css + '</textarea>');

    var TreeListWithCheckItem = Regular.extend({
        name: 'treelist-item',
        template: _tpl,
        config: function(data) {
            this.$ancestor = this.$parent.$ancestor;
            this.data = _u._$merge({
                source: {
                    isChecked: false,
                    isDisabled: false,
                    name: 'list-item',
                    key: 'list-item',
                    iconClass: "",
                    open: true,
                    disabledNum: 0,
                    children: null
                }
            }, data);

            this.openChange = this.openChange._$bind(this);
            this.check = this.check._$bind(this);
        },
        init: function() {
            //init disabledNum
            var source = this.data.source;
            var parent = this.data.parent;
            this.initDisabledNum(source);
            parent && parent.isChecked && this.checkChildren(source);
     
            this.$watch('source.isChecked', function(checked) {
                var source = this.data.source;

                parent && (parent.isChecked = (parent.children.filter(function(item) {
                    return !!item.isChecked;
                }).length === (parent.children.length - parent.disabledNum)) && !parent.isDisabled);
            })
            this.$watch('source.isDisabled', function(isDisabled) {
                var source = this.data.source;

                parent && (parent.isDisabled = parent.children.filter(function(item) {
                    return !!item.isDisabled;
                }).length === parent.children.length);

            })

            this.data.parent && this.openCtrl(this.data.parent)
        },
        initDisabledNum: function(source) {
            var self = this;
            if(source.children && source.children.length) {
                var children = source.children;
                for(var k = 0, klen = children.length; k < klen; k++) {
                    if(children[k].isDisabled) {
                        source.disabledNum++;
                    }
                    self.initDisabledNum(children[k]);
                }
            } else {
                source.disabledNum = 0;
            }
        },
        openChange: function(e) {
            var source = this.data.source;
            source.open = !source.open;
            this.openCtrl(source);
        },
        openCtrl: function(source) {
            var self = this;
            if(source.children && source.children.length) {
                var children = source.children;
                 for(var i = 0, len = children.length; i < len; i++) {
                     if(!source.open) {
                         children[i].open = false;
                         self.openCtrl(children[i]);
                     }
                 }
            }
        },
        checkChildren: function(source) {
            var self = this;
            if(source.children && source.children.length) {
                 var children = source.children;
                 for(var i = 0, len = children.length; i < len; i++) {
                     if(!children[i].isDisabled)
                         children[i].isChecked = source.isChecked;
                     self.checkChildren(children[i]);
                 }
            }
        },
        check: function(e, parent) {
            var source = this.data.source;
            this.$ancestor.check.call(this.$ancestor, source, source.isChecked, this.data.parent);
            this.checkChildren(source);
        }
    })

    return TreeListWithCheckItem;
})
```

这里的重点就是事件抛出,在config时:
```javascript
this.$ancestor = this.$parent.$ancestor;
```

在用户勾选checkbox时，抛出事件到父组件上：
```javascript
this.$ancestor.check.call(this.$ancestor, source, source.isChecked, this.data.parent);
```
这样就简单实现了把所有的checkbox事件都集中在顶级父组件中，简化了处理逻辑。把这个事件机制搞清楚了，剩下的就很好办了。

**treeList**组件，是**treeListItem**组件的父ji组件

**treeList.html**
```html
<ul class="m-treelist-check">
    {#list source as item}
        <li>
            <treelist-item source={item} parent={parent}></treelist-item>
        </li>
        {#if item.children && item.children.length}
        <li class="recursion">
            <tree-list-with-check source={item.children} parent={item}></tree-list-with-check>
        </li>
        {/if}
    {/list}
</ul>
```
**treeList.js**
```javascript
/* 使用示例：
 * <tree-list source={source}></tree-list>
 */

define([
    'base/util',
    'base/event',
    'regular!./treeListWithCheck.html',
    'text!./treeListWithCheck.css',
    'util/template/tpl',
    '{pro}global/util.js',
], function(_u, _v, _tpl, _css, _t, _util) {

    _t._$parseUITemplate('<textarea name="css" id="css-box">' + _css + '</textarea>');
    
    var dom = Regular.dom;

    var TreeListWithCheck = Regular.extend({
        name: 'tree-list',
        template: _tpl,
        config: function(data) {
            this.$ancestor = this.$parent.$ancestor;
        },
        init: function() {
            
        },
        check: function(source, isChecked, parent) {
            this.$ancestor.check.call(this.$ancestor, source, source.isChecked, parent);
        }
    })

    return TreeListWithCheck;
})
```


顶级**treeView**组件

**treeView.html**

```html
<tree-list source={source}></tree-list>
```

**treeView.js**
```javascript
/**
 * 带有checkbox的树组件顶级视图，调用子视图
 * 使用示例：
 * <tree-view source={source} on-check={check时事件处理函数}></tree-view>
 * 
 */

define([
    'base/util',
    'base/event',
    'regular!./treeViewWithCheck.html',
    'text!./treeListWithCheck.css',
    'util/template/tpl',
    '{pro}global/util.js',
], function(_u, _v, _tpl, _css, _t, _util) {

    _t._$parseUITemplate('<textarea name="css" id="css-box">' + _css + '</textarea>');
    
    var dom = Regular.dom;
    
    var TreeViewWithCheck = Regular.extend({
        name: 'tree-view',
        template: _tpl,
        config: function(data) {
            this.$ancestor = this;
        },
        check: function(source, isChecked, parent) {
            this.$emit('check', {
                source: source,
                isChecked: isChecked,
                parent: parent
            })
        }
    })
})
```
相信大家看到了，顶级组件把所有的子组件抛出来的时间都统一$emit出一个事件，这样在后期处理业务逻辑相关的代码时会方便很多，比如我现在要做的一个需求就是将这个树形组件应用在穿梭框中，类似于这种：

![alt 层级目录](/images/transfer.png)

需要实现左右数据类似于联动的绑定。

后期会研究下用vue实现相同的组件。这篇文章先到这里啦~~