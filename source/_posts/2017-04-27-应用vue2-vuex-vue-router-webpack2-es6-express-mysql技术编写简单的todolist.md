---
title: 应用vue2+vuex+vue-router+webpack2+es6+express+mysql技术编写简单的todolist
date: 2017-04-27 11:36:53
tags: vue vuex vue2 vue-router webpack2 es6 express javascript
---

最近对vue很感兴趣，趁闲暇时间，模仿了wunderlist里面的部分功能，编写了前后端分离的基于vue技术栈和express的todolist小项目。写这篇博文来总结思考下。项目所在[github](https://github.com/mengera88/taskiller),可以自行参考克隆

# 总体概览

整个项目最终做成的样子如下：

![alt todolist界面概览](/images/todolist.png)

大家都看到了，整体实现的功能还是比较简单的。由于对express也很感兴趣，就干脆自己动手做了全栈。另外说一下：这个项目只是自己摸索vue与express过程中，做出的成果，如果有哪个地方不对的，还请大神多多指教。

整个todolist界面分为左侧的目录分类和右侧的list。用户切换不同的目录可以对应到相应的list任务中，并且在该任务中能够添加list和删除list，也能够标记已完成与未完成。这些看上去都很简单，但是里面存在了挺多小细节的，我认为，作为一个新手，尤其是对vue新，对express新又对他们很感兴趣的新手，拿这个项目来练手个人觉得很合适。

不多说废话了，先来看看我的项目目录与大致的介绍吧。
# 项目结构

```javascript
├── README.md                        //这里是readme说明文档
├── node_modules                     //一些依赖在这里
├── build                            
│   ├── build.js
│   └── dev.js
├── db                               //数据库相关的东西放在这里                           
│   └── dbConfig.js                  //数据库配置文件
├── dist                             //webpack编译后的目标文件夹
├── package.json                     //这个就不说了，是个前端都懂
├── server                           //服务器端相关的文件都在这里           
│   └── app.js                       //server服务，后台服务入口文件
├── src                              //前端源文件都在这里
│   ├── App.vue                      //vue顶级组件，包含了vue-router
│   ├── components                   //各个子组件
│   │   ├── common                   //包含了页面的公共模块，比如header，footer等
│   │   ├── menu-item.vue            //左侧菜单栏组件
│   │   ├── search.vue               //搜索框组件
│   │   └── todo-list.vue            //list组件
│   ├── config                       //一些前端配置的东西可以放在这里
│   ├── directives                   //vue的一些指令封装可以放在这里
│   ├── filters                      //vue的一些过滤器可以放在这里
│   ├── images                       //放置图片
│   │   ├── Shapes.jpg
│   │   ├── article.jpeg
│   │   └── avatar.png
│   ├── less                         //公共样式less相关放在这里
│   │   ├── common.less
│   │   ├── fonts
│   │   ├── index.less
│   │   ├── lessfont
│   │   ├── mixin.less
│   │   ├── reset.less
│   │   └── variable.less
│   ├── main.js                      //前端入口文件
│   ├── pages                        //放置不同的页面，本项目只有一个页面，所以暂定只有一个
│   │   └── index.vue
│   ├── route.js                     //路由配置文件
│   └── store                        //vuex相关逻辑放在这里
│       ├── actions.js               //actions相关
│       ├── getters.js               //getter相关
│       ├── index.js                 //顶端vuex设置，入口文件
│       ├── modules                  //放置模块
│       ├── mutations.js             //mutation相关
│       ├── plugins.js               //插件相关
│       └── state.js                 //state相关
├── webpack.config.base.js           //webpack基本配置
├── webpack.config.dev.js            //开发环境配置
└── webpack.config.prod.js           //线上环境配置，但没有测试过也没有怎么研究，可以暂时忽略
```

下面来分前后端两个模块大致讲解一下：

# 后端

首先要准备mysql数据库服务，我用MYSQLWorkbench客户端界面新建数据库，初始化库表信息，当然你也可以不用图形界面，直接用命令行。在`dbConfig.js`中配置好数据库设置

```javascript
module.exports = {
    host     : 'localhost',
    user     : 'your user name default root',
    password : 'your password',
    database : 'taskiller'
}
```

本项目中，数据库数据表的新建，并没有写在server服务中，在实际的项目中应该有个自动化的脚本自动创建你需要的数据表和需要的字段信息。因为是当成练手项目做的，所以一切都从简了，把项目用到的数据库和数据表都事先建立好了。我的数据库名是`taskiller`,需要在这个数据库中，建两个表：`todo_list`和`menu`,`todolist`用来存储list信息，`menu`用来存储目录信息。示例：
**menu表**
```
+----+--------+----------+
| id | name   | selected |
+----+--------+----------+
|  1 | sdd    |        0 |
+----+--------+----------+
```

**todo_list表**
```
+--------------+----+------+---------+
| text         | id | done | menu_id |
+--------------+----+------+---------+
| sdfs         | 46 |    0 |       1 |
| this is life | 47 |    0 |       4 |
+--------------+----+------+---------+
```

接下来就是后端服务文件`app.js`了，看代码说话吧：

```javascript
const path = require('path')
const express = require('express')
const mysql = require('mysql');
const dbConfig = require('../db/dbConfig');
const bodyParser = require('body-parser')

const insertMenu = 'INSERT INTO menu SET ?'
const getMenu = 'SELECT * FROM menu WHERE id = ?'
const getAllMenu = 'SELECT * FROM menu'
const getTodolist = 'SELECT * FROM todo_list WHERE menu_id = ?'
const insertTodolist = 'INSERT INTO todo_list SET ?'
const deleteTodo = 'DELETE from todo_list WHERE id = ?'
const updateTodolist = 'UPDATE todo_list SET done = ? WHERE id = ?'

let menus;  //所有menu列表缓存

const app = express();
app.use(bodyParser());

//连接数据库 
let connection = mysql.createConnection(dbConfig);

app.all('*', function(req, res, next) {
    res.header("Access-Control-Allow-Origin", "*");
    res.header("Access-Control-Allow-Headers", "Origin, X-Requested-With, Content-Type, Accept");
    res.header("Access-Control-Allow-Methods","PUT,POST,GET,DELETE,OPTIONS");
    res.header("X-Powered-By",' 3.2.1')
    res.header("Content-Type", "application/json;charset=utf-8");
    next();
});

//添加目录
app.post('/menu/add', function (req, res, next) {
  let reqParam = req.body;
  connection.query(insertMenu, reqParam, function(error, results, fields) {
      if(error) throw error;
      console.log(results, fields)
  })
  res.sendStatus(200);
  next()
});

//得到所有目录
app.get('/menu/get', function(req, res, next) {
    connection.query(getAllMenu, function(error, results, fields) {
        if(error) throw error;
        menus = results;
        res.json(results);
        next()
    })
})


//得到指定id的目录
app.get('/menu/get/:id', function(req, res, next) {
    console.log('ID:', req.params.id);
   
    connection.query(getMenu, req.params.id, function(error, results, fields) {
        if(error) throw error;
        res.json(results);
    })
})

//根据目录获取todolist
app.get('/todolist/get/:id', function(req, res, next) {
    connection.query(getTodolist, req.params.id, function(error, results, fields) {
        if(error) return error;
        res.json(results);
    })
})

//添加todolist到数据库中
app.post('/todolist/add', function(req, res, next) {
    //text,done, menu_id 
    let reqParam = {
        "text": req.body.data.text,
        "done": false,
        "menu_id": req.body.data.curMenu
    };
    var insertId;

    connection.query(insertTodolist, reqParam, function(error, results, fields) {
      if(error) throw error;

      insertId = results.insertId;
      reqParam.id = insertId;
      res.json(reqParam)
    })
    
})

//删除todolist
app.post('/todolist/delete', function(req, res, next) {
    let reqParam = req.body.id

    connection.query(deleteTodo, reqParam, function(error, results, fields) {
      if(error) throw error;
      console.log(results, fields)
    })
    res.sendStatus(200);
})

//改变todolist状态
app.post('/todolist/toggle', function(req, res, next) {
    let reqParam = req.body;
    console.log(reqParam)

    connection.query(updateTodolist, [!reqParam.done, reqParam.id], function(error, results, fields) {
        if(error) throw error;

        console.log(results)
        res.sendStatus(200);
    })
})

app.listen(3001, function() {
    console.log('listening on port 3001')
})


connection.connect(function(err) {
  if (err) {
    console.error('error connecting: ' + err.stack);
    return;
  }

  console.log('connected as id ' + connection.threadId);
});
```

很简单，所有的接口功能应该都是一目了然。因为笔者主要锻炼的还是vue相关的，express只是有兴趣略微带过，因此没有考虑很复杂很完善的一些逻辑。有几个注意事项：

1. 因为前端的地址端口是3000，后端server的端口又设定了3001，这就涉及到了跨域，因此我加了这段代码，来解决跨域的问题。
```javascript
app.all('*', function(req, res, next) {
    res.header("Access-Control-Allow-Origin", "*");
    res.header("Access-Control-Allow-Headers", "Origin, X-Requested-With, Content-Type, Accept");
    res.header("Access-Control-Allow-Methods","PUT,POST,GET,DELETE,OPTIONS");
    res.header("X-Powered-By",' 3.2.1')
    res.header("Content-Type", "application/json;charset=utf-8");
    next();
});
```
2. 其实这里还是有漏洞的，坐等高手指出来（微笑脸）

后台express没有用express-generator生成一个完整的架构。笔者之前尝试过用这种一键生成的工具快速搭建后台环境，但是这样就都用现成的，好多东西概念就会非常模糊，不太好掌握一些技术细节，也不会很透彻理解这样写的架构到底是为什么，为什么不采用其他架构方式？所以，笔者决定自己纯手写后台的server，这个最初的版本是写的最简单的版本，等后期再深入研究express的时候，再把这个雏形向着可扩展性和模块化发展。不过已经对express很熟的同学完全可以不照着我这个小白的写法写~

# 前端

## webpack配置


前端刚开始我遇到的门槛就是webpack一些配置，网上的教程真的是五花八门，由于本人的目的是学习vue的，并不是捣鼓webpack，所以，在webpack配置方面并没有花很多的时间研究，也是在网上找教程，慢慢摸索倒腾出来的。不过经过这次倒腾我认识到，我有必要研究下webpack的一些东西了。不然配置个东西，太痛苦，并且前端技术日新月异，网上的教程五花八门，有老旧版本的，有新的版本的，很容易让人摸不着头脑，建议还是去官网学习比较好。后期我研究了再写博文总结下经验。

这里先贴一下我的webpack配置，有些地方做了简要的注释。


**webpack.config.base.js**

```javascript
const webpack = require('webpack');
const path = require("path");
const fs = require("fs");
const ExtractTextPlugin = require("extract-text-webpack-plugin");
const HtmlWebpackPlugin = require("html-webpack-plugin");
const autoprefixer = require('autoprefixer');


const PATHS = {
  src: path.resolve(__dirname, './src'),
  dist: path.resolve(__dirname, './dist')
}

module.exports = {
  entry: {
    app: './src/main.js',           // 整个SPA的入口文件, 一切的文件依赖关系从它开始
    vendors: ['vue', 'vue-router']  // 需要进行单独打包的文件
  },
  output: {
    path: PATHS.dist,
    filename: 'js/[name].js',
    publicPath: '/dist/',           // 部署文件 相对于根路由
    chunkFilename: 'js/[name].js'   // chunk文件输出的文件名称 具体格式见webpack文档, 注意区分 hash/chunkhash/contenthash 等内容, 以及存在的潜在的坑
  },
  devtool: '#eval-source-map',      // 开始source-map. 具体的不同配置信息见webpack文档
  module: {
    rules: [{
        test: /\.vue$/,
        loader: 'vue-loader'
      },
      {
        test: /\.js/,
        loader: 'babel-loader',
        exclude: /node_modules/
      },
      {
        test: /\.(png|jpe?g|gif|svg)(\?.*)?$/,
        loader: 'url-loader?limit=10240&name=images/[name].[ext]'
      },
      {
        test: /\.less/,
        use: ExtractTextPlugin.extract({
          fallback: 'style-loader',
          use: [
            'css-loader',
            'less-loader'
          ]
        })
       },
       {
        test: /\.css/,
        use: ExtractTextPlugin.extract({
          fallback: 'style-loader',
          use: [
            'css-loader'
          ]
        })
       },
       {
          test: /\.(eot|svg|ttf|woff|woff2|png)\w*/,
          loader: 'file-loader'
        }
     ]
  },
  resolve: {
    alias: {
      'vue$': 'vue/dist/vue.common.js',
      'components': path.join(__dirname, 'src/components'),   // 定义文件路径， 加速打包过程中webpack路径查找过程
      'lib': path.join(__dirname, 'src/lib'),
      'less': path.join(__dirname, 'src/less'),
      'filters': path.join(__dirname, 'src/filters'),
      'directives': path.join(__dirname, 'src/directives'),
    },
    extensions: ['.js', '.less', '.vue', '*', '.json']        // 可以不加后缀, 直接使用 import xx from 'xx' 的语法
  },
  plugins: [
    new HtmlWebpackPlugin({                                   // html模板输出插件
      title: 'task kill',
      template: `${PATHS.dist}/template/index.ejs`,
      inject: 'body',
      filename: `${PATHS.dist}/pages/index.html`
    }),
    new ExtractTextPlugin({                                   // css抽离插件,单独放到一个style文件当中.
      filename: `css/style.css`,
      allChunks: true,
      disable: false
    }),
    // 将vue等框架/库进行单独打包, 并输入到vendors.js文件当中
    // 这个地方commonChunkPlugin一共会输出2个文件, 第二个文件是webpack的runtime文件
    // runtime文件用以定义一些webpack提供的全局函数及需要异步加载的chunk文件
    // 具体的内容可以看我写的blog 
    // [webpack分包及异步加载套路](https://segmentfault.com/a/1190000007962830)
    // [webpack2异步加载套路](https://segmentfault.com/a/1190000008279471)
    new webpack.optimize.CommonsChunkPlugin({
      names: ['vendors', 'manifest']
    })
  ]
}
```


**webpack.config.dev.js**

```javascript
const merge = require('webpack-merge');

module.exports = merge(require('./webpack.config.base'), {
  devServer: {
    proxy: {
      '/api': {
        target: 'http://localhost:3001',
        changeOrigin: true,
        secure: false,
        pathRewrite: {
          "^/api": ""
        }
      }
    }
  }
})
```
**webpack.config.prd.js**就不贴了，这次我也没有用上，大家有兴趣可以直接去[github](https://github.com/mengera88/taskiller)里看。

## 逻辑设计

### vuex概要

在这里不得不感谢vuex，这个东西对开发效率的提升真的很有帮助，vuex不熟悉的童鞋可以去[官网](https://vuex.vuejs.org/zh-cn/)查阅，建议既然学习了vue了，vuex必不可少，真的会节省你很多的开发时间。这里我就做个简要的介绍：

vuex是一个专为vue开发的状态管理模式，它采用集中式存储管理应用的所有组件的状态，并以相应的规则保证状态以一种可预测的方式变化。
这个状态自管理应用包含以下几个部分：

1. state，驱动应用的数据源；
2. view，以声明方式将state映射到视图；
3. actions，响应在view上的用户输入导致的状态变化。

比如以下的单一数据流示意图：

![alt 单向流数据示意图](/images/flow.png)

但是，有过组件编写经验的童鞋应该知道，当我们的应用遇到多个组件共享状态时，单向数据流的简洁性很容易被破坏：

1. 多个视图依赖于同一状态。
2. 来自不同视图的行为需要变更同一状态。

对于问题一，传参的方法对于多层嵌套的组件将会非常繁琐，并且对于兄弟组件间的状态传递无能为力。对于问题二，我们经常会采用父子组件直接引用或者通过事件来变更和同步状态的多份拷贝。以上的这些模式非常脆弱，通常会导致无法维护的代码。

因此，诞生了vuex，用来把组件的共享状态抽取出来，以一个全局单例模式管理。在这种模式下，组件树构成了一个巨大的“视图”，不管在树的哪个位置，任何组件都能获取状态或者触发行为！

另外，通过定义和隔离状态管理中的各种概念并强制遵守一定的规则，代码也会变得更结构化且易维护。

来看看这张经典的图例：

![alt vuex](/images/vuex.png)

大致的就介绍到这里啦，需要更加深入的童鞋可以移步[官网](https://vuex.vuejs.org/zh-cn/)。下面针对本项目的逻辑，介绍下我设计的vuex。

### state设计

由于目前实现的逻辑还是较为简单，因此，只有涉及了3个state：
```javascript
export const state = {
    todos: [],     //当前选中目录curMenu对应的todolist
    curMenu: {},   //选中的目录
    menus: []      //所有目录列表
}

```

目录对应的todo列表的切换，我采用的方式是：`curMenu`只要一有变动就会向后端发送请求，后台返回该目录下对应的所有todo列表，更新到todos。因此，这里只设计了一个todos状态就行了。

### mutations设计

mutation只能同步，无法异步，因此`mutations.js`只设计状态的改变。此处根据交互有这几处涉及到的状态改变：
1. 向todos中添加todo
2. 在todos中删除选中的todo
3. 更新选中todo的完成状态
4. 设置当前目录对应的todos数组
5. 设置所有的目录列表menus
6. 设置当前选中的目录curMenu

对应的js代码如下：
```javascript
export const mutations = {
    //添加todo
    addTodo(state, {todo}) {
        state.todos.push(todo)
    },
    //删除todos
    deleteTodo(state, {todo}) {
        state.todos.splice(state.todos.indexOf(todo), 1);
    },
    //设置当前的todos
    setTodo(state, {todos}) {
        state.todos = todos
    },
    //切换todo的完成状态
    toggleTodo(state, {todo}) {
        todo.done = !todo.done;
    },

    //左侧menu切换，设置当前menu值
    setCurMenu(state, {menu}) {
        state.curMenu = menu;
    },
    //设置menu值，
    getMenu(state, {menus}) {
        state.menus = menus;
    }
}
```


### acitions设计

由于需要后台存储一些todo列表的状态，需要将一些动作的变动告知后台，后台更新相应的数据库信息，需要添加actions。

这里要注意：`actions.js`主要放置一些与后台交互相关的操作，而`mutations.js`只用作状态的改变。

仔细看看交互，会发现这里存在以下几个动作：

1. 获取所有的目录，即获得menus列表
2. 切换menu时，向后台获取对应的todolist，并更新对应todos列表
3. 用户添加list时，向后台发送请求存储在数据库中
4. 用户删除list时，向后台发送请求将数据库中该条记录删除
5. 用户改变todolist中的某个list的状态时，向后台发送请求更新数据库中该条记录的`done`字段

```javascript
import axios from 'axios';

const host = 'http://localhost:3001'

//获取所有的menu
export const getMenus = ({commit}) => {
    axios.get(host + '/menu/get')
    .then((response) => {
        var data = response.data;
        var initMenu
        data.forEach(function(item) {
            if(item.selected) {
                initMenu = item
            }
        })
        commit('getMenu', {menus: data});       //提交mutation，初始化menus列表
        
        commit('setCurMenu', {menu: initMenu}); //提交mutation，初始化curMenu。

        axios.get(host + '/todolist/get/'+initMenu.id)
        .then((response) => {
            var data = response.data;
            commit('setTodo', {todos: data})     //根据初始化的curMenu获取todolist，并提交mutation更新todos列表
        })
        .catch( error => {
            console.log(error)
        })
    })
    .catch( error => {
        console.log(error)
    })
}

//确切的讲是getCurTodoList，得到当前menu对应的todolist
export const setCurMenu = ({commit}, menuData) => {
    axios.get(host + '/todolist/get/'+menuData.id)
    .then((response) => {
        var data = response.data;
        commit('setTodo', {todos: data})               //用数据库返回的数据提交mutation，更新todo列表
        commit('setCurMenu', {menu: menuData.menu})    //更新当前目录，提交mutation来更新curMenu值
    })
    .catch( error => {
        console.log(error)
    })
}

//添加todo
export const addTodo = ({commit}, data) => {
    axios.post(host + '/todolist/add', {
        data: data,
    })
    .then((response) => {
        commit('addTodo', {todo: response.data})   // 提交mutation，向todos中添加数据库返回的记录
    })
    .catch( error => {
        console.log(error)
    })
}

//删除todo
export const deleteTodo = ({commit}, todo) => {
    axios.post(host + '/todolist/delete', {
        id: todo.todo.id,
    })
    .then((response) => {
        commit('deleteTodo', todo)               //提交mutation，在todos中删除该条记录
    })
    .catch( error => {
        console.log(error)
    })
}

//完成与未完成的切换
export const toggleTodo = ({commit}, todo) => {
    axios.post(host + '/todolist/toggle', {
        id: todo.todo.id,
        done: todo.todo.done
    })
    .then((response) => {
        console.log(response)
        commit('toggleTodo', {todo: todo.todo})    //提交mutation，更新该条todo的完成状态
    })
    .catch( error => {
        console.log(error)
    })
}
```
### getters
细心的同学会发现，我的界面里面分门别类的显示了已完成和未完成的类别，因此需要通过getters来根据todos的数据获得对应的数据

```javascript
export const doneTodos = state => {
    return state.todos.filter(item => item.done)
}

export const undoneTodos = state => {
    return state.todos.filter( item => !item.done)
}
```

## 组件

### 页面组件
这里我就不多赘述了，都是组件相关的概念，要细讲的话概念点就太多了，看不懂的建议大家去[vue官网](https://cn.vuejs.org/)学习相关概念再来看。

**index.vue**
```html
<template>
  <article class="post-page">
    <div class="g-left">
      <menus></menus>
    </div>
    <div class="g-right">
      <input class="adds" type="text" v-model="addlists"  @keyup.enter="addList($event)" placeholder="添加任务...">
      <todo-list class="todo" v-for="(todo, index) in undoneTodos" :todo="todo" :key="index"></todo-list>
      <div class="title" @click="isShowDone = !isShowDone"> {{isShowDone ? '隐藏已完成任务' : '显示已完成任务'}} <span class="subtitle">共{{doneTodos.length}}项</span></div>
      <todo-list v-show="isShowDone" class="todo" v-for="(todo, index) in doneTodos" :todo="todo" :key="index + 'done'"></todo-list>
    </div>
  </article>
</template>

<script>
  import TodoList from 'components/todo-list.vue'
  import Menus from 'components/common/menus.vue'

  import { mapState } from 'vuex'
  import {mapGetters} from 'vuex'

  const COMPONENT_NAME = 'post-page';

  export default {
    name: COMPONENT_NAME,
    data() {
      return {
        addlists: '',
        isShowDone: false
      }
    },
    computed: {
      ...mapState([
        'todos',
        'curMenu'
      ]),
      ...mapGetters([
        'doneTodos',
        'undoneTodos'
      ])
    },
    methods: {
      addList(e) {
        var text = this.addlists
        var curMenu = this.curMenu.id
        this.$store.dispatch('addTodo', {text: text, curMenu: curMenu})
        this.addlists = '';
      },
    },
    components: {
      TodoList,
      Menus
    }
  }
</script>

<style lang="less">
  .post-page {
    width: 100%;
  }
  .adds{
    width: 100%;
    border: 1px solid #dedede;
    margin-bottom: 40px;
    margin-left: auto;
    margin-right: auto;
    height: 32px;
    line-height: 32px;
    border-radius: 3px;
    padding-left: 10px;
  }
  .todo{
    text-align: left;
  }
  .title{
    width: 300px;
    height: 32px;
    line-height: 32px; 
    text-align: left;
    cursor: pointer;
    .subtitle{
      color: #999;
      font-size: 12px;
    }
  }
</style>
```

这里的`mapState`和`mapGetters`等都在vuex官网里面有介绍，前面的三点`...`是`es6`的语法对象展开符，不懂的童鞋可google一下，网上一堆教程。

我们可以看到，这个页面我用到了`menus`组件和`todo-list`组件,这里我遇到过一个坑，vue文件的模板`template`中的元素只能有一个根节点，不能有多个根节点，大家写的时候在只在顶层写一个标签就好了。

### menus组件
**menus.vue**
```html
<template>
    <div class="menu">
        <menu-item-list v-for="menu in menus" :item="menu" ></menu-item-list>
    </div>
</template>
<script>
    import MenuItemList from 'components/menu-item.vue'
    import {mapActions} from 'vuex'
    import {mapState} from 'vuex'

    const COMPONENT_NAME = 'menu';

    export default {
        name: COMPONENT_NAME,
        data() {
            return {
            }
        },
        created() {
            this.getMenus();
        },
        components: {
            MenuItemList
        },
        computed: {
            ...mapState(['menus'])
        },
        methods: {
            ...mapActions(['getMenus'])
        }
    }
</script>
<style lang="less">
    .menu {
        width: 100%;
    }
</style>
```


**menu-item.vue**
```html
<template>
    <div class="menu-list" @click="setCurMenu(item)" :class="{selected: curMenu === item}">
        <div>
            <i class="fa fa-list-ul" aria-hidden="true"></i>
            <span>{{item.name}}</span>
            <span v-show="!item.selected" class="tips"></span>
            <i v-show="curMenu === item" class="fa fa-pencil-square-o tips" aria-hidden="true"></i>
        </div>
    </div>
</template>
<script>
    const COMPONENT_NAME = 'menu-item-list';
    import {mapState} from 'vuex'

    export default {
        name: COMPONENT_NAME,
        data() {
            return {
            }
        },
        computed: {
            ...mapState([
                'curMenu',
            ]),
        },
        props: ['item'],
        methods: {
            setCurMenu(menu) {
                this.$store.dispatch('setCurMenu', {id: this.item.id, menu: menu})
            }
        }
    }
</script>
<style lang="less">
    .menu-list {
        width: 100%;
        padding-left: 20px;
        cursor: pointer;
        &:hover{
            background: #29A9FF;
            div{
                color: #fff;
            }
        }
        &.selected{
            background: #29A9FF;
            div{
                color: #fff;
            }
        }
        div{
            height: 38px;
            line-height: 38px;
            color: #333;
        }
        i{
            display: inline-block;
            height: 38px;
            line-height: 38px;
        }
        .tips{
            float: right;
            margin-right: 20px;
        }
    }
</style>
```

### todo-list组件
**todo-list.vue**

```html
<template>
    <div class="list">
        <input type="checkbox" :checked="todo.done" @change="toggleTodo({todo: todo})">
        <span>{{todo.text}}</span>
        <span class="delete fa fa-times-circle" aria-hidden="true" @click="deleteTodo({todo: todo})"></span>
    </div>
</template>
<script>
    import { mapActions} from 'vuex'

    const COMPONENT_NAME = 'todo-list';

    export default {
        name: COMPONENT_NAME,
        props: ['todo'],
        methods: {
            ...mapActions([
                'deleteTodo',
                'toggleTodo'
            ])
        }
    }
</script>
<style  lang="less" coped>
    .list{
        width: 300px;
        height: 32px;
        line-height: 30px;
        border: 1px solid #dedede;
        margin-bottom: 20px;
    }
    .delete{
        float: right;
        margin-top: 8px;
        margin-right: 10px;
        cursor: pointer;
        color: #999;
    }
</style>
```

# 总结
至此，主要的功能结构都讲的差不多了，有哪里讲的不清楚的地方，还望指出来，互相学习。
不得不说，vue框架用来来真的不错，大家在开发的时候，记住：要从数据的角度思考问题，一切就会变得如此简单。
这里记录下接下来要添加的功能，如果有踩坑会继续带来博文分享互相交流学习下：

1. 顶部路由添加user登录信息
2. 顶部单独辟出一个路由显示已完成任务或其他主题
3. 左侧menu栏目添加新建目录功能，其实后端接口已经写好，前端需要加一下
4. 左侧每个目录需要有右键功能，右键弹出项目暂定： 重命名、删除
5. 右侧添加任务的界面要改的好一点，样式细节需要调整
6. 右侧输入框目前是点击回车之后会自动添加一个项目，但是在中文输入法时直接按回车切换英文时存在bug，会导致list存在空白的情况，这个要处理一下
7. 每个list是否需要可以编辑待定
8. 左侧目录最右端显示有几条todolist功能添加