---
title: 自制简易前端MVC框架
date: 2018-02-05 09:35:28
tags: 前端
---
周末花了大概7小时写了一个简易的响应式blog，原意是练习css的，写着写着却去实现了一套前端路由并渲染的东西，这里写一点心得体会

#### 基本思路与涉及技术
1. 使用url hash进行路由跳转
2. js监听hashchange事件，根据当前hash去决定界面如何渲染
3. 调用 addHandler(hash, func) 这个api来映射hash与handler
4. gulp，scss， es6，模板引擎
5. 需要一些es6的知识，需要理解this
6. 整个工程在 https://github.com/MoonShining/front-end-hero/tree/master/blog， front-end-hero是我自己写的模板代码生成器，用它来练习CSS， 使用**ruby create.rb -n name -s url**来快速创建目录结构，免去重复的工作

#### 例子

![web.gif](http://upload-images.jianshu.io/upload_images/4073552-dabf8b4dce1f48e9.gif?imageMogr2/auto-orient/strip)

#### 核心代码

```js
(()=>{

    let blog = new Blog()
    // add simple router
    blog.addHandler(['', '#programming'], function(){
        let data = {articles: [{title: 'stories to be continue', date: '2017-04-09'}]}
        this.compile('#article-template', data) 
    })

    blog.addHandler('#about', function(){
        let data = {avatar: 'http://7xqlni.com1.z0.glb.clouddn.com/IMG_0337.JPG?imageView2/1/w/100/h/100', name: 'Jack Zhou'}
        this.compile('#about-template', data)
    })

    // initialize the page
    blog.init()

})()
```

调用blog.addHandler来自定义路由改变之后触发的动作

```js
class Blog {

    constructor(){
        this.content = '#content'
        this.router = {}
    }

    init(){
        this.dispatch()
        $(window).on('hashchange',()=>{ 
            this.dispatch()
        });
    }

    dispatch(){
        this.handle(window.location.hash)
    }

    addHandler(hash, func){
        if(Array.isArray(hash)){
            for(let item of hash){
                this.router[item] = func
            }
        }else{
            this.router[hash] = func
        }
    }

    handle(hash){
        if(this.routeMissing(hash)){
            this.handle404()
            return
        }
        this.router[hash].call(this)
    }

    routeMissing(hash){
        if(this.router[hash])
            return false
        else
            return true
    }

    handle404(){
        console.log('handler not found for this hash')
    }

    compile(templateSelector, data, element=this.content){
        let source   = $(templateSelector).html()
        let template = Handlebars.compile(source)
        $(element).html(template(data))
    }

}

```

```this.router```是个是核心，其实也参考了一点Rails的设计，通过一个对象去保存 路由＝》动作 的关系， 并且把核心逻辑都封装在Blog这个类中。
