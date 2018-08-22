---
title: 基于TypeScript装饰器定义Express RESTful 服务
categories: ['Node.js']
tags: ['typescript', 'express']
---

### 前言
本文主要讲解如何使用TypeScript装饰器定义Express路由。文中出现的代码经过简化不能直接运行，完整代码的请戳：https://github.com/WinfredWang/express-decorator

### 1 为什么使用装饰器

当我们在使用Express时，经常要暴露RESTful服务，代码如下：

```
var express = require('express');
var app = express();
app.get('/users', function(req, res) {
  res.send([{name:'xx'}]);
});

// 路由模块化写法
var router = express.Router();
app.get('/users', function(req, res) {
  res.send([{name:'xx'}]);
});
```
熟悉Java WEB童鞋知道[`jax-rs`](https://baike.baidu.com/item/JAX-RS/10914743?fr=aladdin)可以使用标注(annotation)声明服务。例：
```
@Path("/myResource")
public class SomeResource {
    @GET
    public String doGetAsPlainText() {
        ...
    }
 
    @GET
    public String doGetAsHtml() {
        ...
    }
}
```
使用这种方式声明的服务非常简洁方便，免去了写一坨重复代码之苦，而且看起来更加清晰，那我们看看在Node.js中如何做。

---
### 2 需求

参照[`jax-rs`](https://docs.oracle.com/javaee/6/tutorial/doc/gijqy.html)规范，我们列出如下需求：

- 使用`@Path`声明RESTful服务路由
- 使用`@GET/@POST/@DELETE/@PUT`声明子路由
- 使用`@PathParam，@QueryParam，@HeaderParam，@CookieParam，@FormParam,`来接受服务参数

---

### 3 实现思路

在ES6和TypeScript中有新特性:`装饰器(Decorator)`，正好我们可以借助它实现我们的需求。至于装饰器用法，可以参考我的[上一篇文章](http://www.cnblogs.com/winfred/p/8216650.html)。

![20180107195916](https://user-images.githubusercontent.com/2776992/34775124-ffb5ef42-f64c-11e7-8438-28fb4646e2f5.png)


上图中左边是Java中定义RESTful代码，右边是Express代码，其实他们本质上是一一对应的。我们只要在装饰器的定义中实现Express 路由即可。

继续思考，我们Express 路由到底是放到那个注解中实现呢？
我们知道不同装饰器(类/方法/参数)执行顺序不同：

> 参数装饰器先执行，然后方法最后类装饰器

根据这个特性我们应该将核心实现放到类装饰器`Path`中执行是不是就可以了呢？

其实不是，我们看如下代码，我们在`user-service.ts`中定义了`UserService`服务。

```
@Path("/user")
 class UserService {
    @GET("/{id}")
    public getUsers(@PathParam("id") id： string) {
       // TODO
    }
 }
```
我们定义好了服务，然后想让Node.js模块加载，我们必须在工程入口模块(main.ts)中导入上述文件
main.ts代码：
```
import { HelloService } from './hello-service'

// TODO
```
上述服务代码会执行吗？也就是说
如果仅仅导入模块，而没有使用该模块的话，Node.js是否会加载这个模块呢，换句话说这个模块会执行吗？答案是NO。
为啥呀？因为Node.js对其做了优化，只有一个模块被真正用到才会加载。

上有政策，下有对策。我们就在模块引用一下。
```
import { HelloService } from './hello-service'

HelloService; // 就是为了让Node加载它
```
这样好吗，当然不好。谁知道这是干嘛的。

所以我们应该换了思路，将Express 注册路由代码拿到装饰器外部，额外提供注册服务的入口，通过该注册服务入口，用户可以显式看到有哪些服务。
```
import { HelloService } from './hello-service';
import {RegisterService } from 'xxx';

RegisterService([HelloService]);//注册服务
```

---

### 4 装饰器核心代码

基于上面的思考，我们在装饰器的实现中只是单纯地存储RESTful url以及参数即可，剩下服务注册工作交给`RegisterService`去做。

##### Path装饰器实现

```
 function Path(baseUrl: string) {
    return function (target) {
        target.prototype.$Meta = {
            baseUrl: baseUrl
        }
    }
}
```
这里我们将RESTful路由存储到类的原型中，以便服务实例化时能获取到。

##### GET/POST/DELETE/PUT

```
function GET (url: string) => {
    return (target, methodName: string, descriptor: PropertyDescriptor) => {
        let meta = getMethod(target, methodName);
        meta.subUrl = url;
        meta.httpMethod = httpMehod;
    }
}
```

##### QueryParam/PathParam等实现

```
function PahtParam(paramType: string) {
    return function (target, methodName: string, paramIndex: number) {
        let meta = getMethod(target, methodName);
        meta.params.push({
            name: paramName ? paramName : paramType,
            index: paramIndex,
            type: paramType
        });
    }
}
```
上述就装饰自身代码，本质上就是讲路由、http请求方法和参数存储到类的原型对象中，以便后续可以去到。

### 5 注册服务核心代码
 
##### 路由实现

经过上面的分析，我们可知注册服务主要将Express中注册路由交由我们框架处理,核心代码如下：
```
function RegisterService(app, service) {
    let router = Router();

    // 1. 获取存储在原型对象中的http请求信息()
    let meta = getClazz(service.prototype);

    // 2. 实例化服务类
    let serviceInstance = new service();
    let routes = meta.routes;

    for (const methodName in routes) {
        let methodMeta = routes[methodName];
        let httpMethod = methodMeta.httpMethod;

        // 3. 回调函数
        let fn = (req, res, next) => {
            let result = service.prototype[methodName].apply(serviceInstance, params);
            res.send(result);
        };

        // 4. 注册路由
        router[httpMethod].apply(router, methodMeta.subUrl);
    }
    // 5. 路由中间件
    app.use.apply(app, [meta.baseUrl]);
}
```
![image 6](https://user-images.githubusercontent.com/2776992/34775138-0b038fbc-f64d-11e7-95db-e72548da8064.png)


##### http请求参数处理

```
 @GET('/:id', [ testMidware1 ])
 list( @PathParam('id') id: string, @QueryParam('name') name: string) {
    return {name:"tom", age: 10}
 }
```
用户编码时我们期望回调函数中的参数框架自动注入，而不是让用户自己从`request`中取，所以在注册服务代码中第3处，框架需要出更加参数装饰器中信息，从request中取值后注入回调函数中

```javascript
// 3. 回调函数
let params = extractParameters(req, res, methodMeta['params']);
let fn = (req, res, next) => {
    let result = service.prototype[methodName].apply(serviceInstance, params);
    res.send(result);
};

// 根据参数类型，从request取出对应的值
function extractParameters(req, paramMeta) {
    let paramHandlerTpe = {
        'query': (paramName: string) => req.query[paramName],
        'path': (paramName: string) => req.params[paramName],
        'form': (paramName: string) => req.body[paramName],
        'cookie': (paramName: string) => req.cookies && req.cookies[paramName],
        'header': (paramName) => req.get(paramName),
        'request': () => req, // 获取request/response对象，做一些特别操作
        'response': () => res,
    }
    let args = [];
    params.forEach(param => {
        args.push(paramHandlerTpe[param.type](param.name))
    })
    
    return args;
}
```

##### response处理

```javascript
 @GET('/:id', [ testMidware1 ])
 list( @PathParam('id') id: string, @QueryParam('name') name: string) {
    return {name:"tom", age: 10}
 }
 ```
一个服务处理完成后，总是要向浏览器返回值的，在回调函数中直接使用`return`语句，而不是自己调用response.send方法， 如下代码：


```javascript
// 3. 回调函数
let fn = (req, res, next) => {
    let result = service.prototype[methodName].apply(serviceInstance, params);

    // 支持promise处理
    if (result instanceof Promise) {
        result.then(value => {
            !res.headersSent && res.send(value);
        }).catch(err => {
            next(err);
        });
    } else if (result !== undefined) {
        !res.headersSent && res.send(result);
    }
};
```

### 6 总结

以上就是我们框架处理核心代码，核心实现主要有两步：
- 装饰器本身用来存在路由信息
- 注册机制实现express路由注册（回调函数参数处理，返回值处理等）