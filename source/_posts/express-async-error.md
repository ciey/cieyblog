---
title: express全局处理async error
author: ciey
categories: web开发
tags:
  - node.js
  - express
date: 2019-08-22 18:22:00
---
![maxresdefault](https://user-images.githubusercontent.com/3664948/63739876-7ba27b00-c8c1-11e9-8c0a-a55aff282f56.jpg)

express 结合 async/await 可以获得很好的开发体验。一般情况下 async/await 在错误处理方面，在最一开始使用Promise时，都习惯用Promise.catch()处理错误，之后async/await 流行后，大家习惯用 ` try/catch` 来处理。
```
router.get('/users', async (req, res, next) => {
  try {
   const users = await User.findAll();
    res.send(users);
  } catch (err) {
    logger.error(err.message, err);
    res.send([]);
  }
});
```
这样做本身没什么问题，但是代码中会存在大量重复的` try/catch`，能不能将异常全局捕获统一处理呢？

假设在没有加`try/catch`的情况下
```
router.get('/users', async (req, res, next) => {
    const users = await User.findAll();
    res.send(users);
});
```
如果User.findAll()报错了， **express全局的错误处理并不能直接捕获 Promise 错误**。会报一个` UnhandledPromiseRejectionWarning `错误。

如何解决这个问题，我们先来看一下express错误处理的机制。

express中定义的错误处理中间件函数的定义方式与其他中间件函数基本相同，差别在于错误处理函数有四个自变量而不是三个：(err, req, res, next)：
```
app.use(function(err, req, res, next) {
  console.error(err.stack);
  res.status(500).send('Something broke!');
});
```
所以exprees中正常处理与错误处理路径是分开的，多了一个`err`参数。
```
app.use((err, req, res, next)=>{}) 对应走 Layer.handle_error
app.use((req, res, next)=>{}) 对应走 Layer.handle_request
```

要想express中能够全局捕获`Promise`对象错误，需要再封装一层用来处理Promise对象。
```
const asyncHandler = fn => (req, res, next) =>
  Promise.resolve()
    .then(() => fn(req, res, next))
    .catch(next);

router.get('/users', asyncHandler(async (req, res) => {
  const users = await User.findAll();
  res.send(users);
}));
```
`asyncHandler`会将Promise中错误通过catch()捕获并交给 next，这样就会去到 express 全局错误中间件中。

但如果在每个路由请求中都增加这个捕获异常的asyncHandler函数跟在每个中都加` try/catch`本质上没多大区别。而且代码看上去也复杂。

还有一种更简便的方法，使用[express-async-errors](https://github.com/davidbanham/express-async-errors)。原理是：
```
This is a very minimalistic and unintrusive hack. 
Instead of patching all methods on an express Router, 
it wraps the Layer#handle property in one place, leaving all the rest of the express guts intact.
```
翻译：这是一种非常简约而且没有入侵性的hack方式。代替了在每个express路由方法中打补丁的方式，它通过复写express中Layer#handle方法，把每个Routing Function的错误统一用 next(err) 传递。

用法也很简单，在express后引入require('express-async-errors')，就可以在express错误处理中捕获错误了。
```
// error handle
app.use((err, req, res, next) => {
  logger.error(err.message, err);
  if (req.xhr) {
    return res.json({
      state: false,
      msg: err.message
    });
  }
  next(err);
});
```
