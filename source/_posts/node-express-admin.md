---
title: node实现后台权限管理系统
author: ciey
categories: web开发
tags:
  - node.js
  - express
date: 2019-08-21 10:22:00
---
本文面向的是node初学者，目标是搭建一个基础的后台权限系统。使用的node框架是上手最简单的express，模板是ejs，这些在node入门的书籍中都有介绍说明，所以应该是难度较低的。

对于node初学者来说，可以先尝试搭建一个blog，单用户的或者多用户的都可以。[cnodejs](https://github.com/cnodejs/nodeclub)论坛是我学的第一个源码，还是很经典的。但是由于开发比较早，基于nodev4版本，很多后面新加的特性都未使用，比如class，async/await语法等。随着node版本的大幅升级，目前至少是基于nodev8，稳定的是nodev10版本开发。所以本系列教程node版本至少是8版本，推荐装LTS 10版本。

ide强烈推荐速度最快，使用最流畅的Visual Studio Code。它也是基于node的electron实现的，一级棒。

数据库用过MS SQLServer、MySQL、mongodb，其中mongodb好多node教程推荐使用才开始流行，但它并非关系型数据库，所以综合考虑还是选用MySQL。数据库客户端推荐Navicat Premium。

以上各环境的安装准备工作大家可以自行教程，不在展开说明了。

说下此项目的目标是搭建一个后台权限管理系统。对于实际项目的业务开发，后台的基础权限框架必不可少。本教程将会带你如何设计用户、角色、菜单、及权限控制，并通过代码示例实现。

如何定义一个好的框架，没有一个标准。主要看每个人的水平及项目的适用性。以前从事.net开发，学设计模式、封装、Ioc注入等，好固然是好，但也提高了入门的门槛，想要让一个初学者快速上手进行业务开发却很难。因为封装后的代码可读性变差，如果没有一定水平框架也很难维护。还有就是如果一个项目比较小，那么在设计框架时分层分个5、6层就有点过头了，一般3层MVC就够用了。当然如果你在大型互联网公司，接触到的用户数量是几万甚至几十万，业务也很多，那框架就势必要考虑到很多方面的问题，那架构设计就不是那么简简单单的了，可能会涉及到分布式、微服务等。

我们可以先从基础的简单的框架入手，等经验丰富了后再不断的重构升级以应对日益增长的业务需求。

下面我们来开始设计并搭建框架。

最基础的权限是用户的登录，通过用户名和密码跟数据库匹配来判断是否登录成功。
![用户登陆](https://user-images.githubusercontent.com/3664948/63405937-f7f91200-c41a-11e9-8179-52d2c8dbdbc7.png)

用户登录并存入session后，下次只需判断session中用户是否存在，可以写在express中间件中。
```
/** 权限判断中间件*/
class authMiddleware {
    /** 需要用户登录*/
    async loginRequired(req, res, next) {
        if (!req.session || !req.session.user || !req.session.user.id) {
            return res.redirect('/login');
        }
        await next();
    }
}

module.exports = new authMiddleware();
```

然后在每次请求的路由中，先判断下用户是否已登录，然后再执行相应controller。
```
const router = require('express').Router(),
    auth = require('./middleware/auth'),
    login = require('./controller/login'),
    main = require('./controller/main');

router.get('/login', login.showLogin);
router.post('/login', login.login);

router.get('/main', auth.loginRequired, main.showMain);
```
其中登录页面及登录post提交是不需要检查session中有无用户的，因为用户这时候还没登录成功。但是像主页/main设定的是需要用户登录才能查看的。

登录后的后台管理系统布局一般都是左侧是菜单树，右侧为内容。
![image](https://user-images.githubusercontent.com/3664948/63408315-4c06f500-c421-11e9-8547-71fc28b8435b.png)

很显然左侧的菜单需要权限控制，用于区分哪些菜单(页面)用户可以访问，哪些菜单(页面)用户不可以访问。一般不能访问的菜单(页面)不显示给用户，这样不同的用户登录后显示的左侧菜单是不相同的。
![image](https://user-images.githubusercontent.com/3664948/63409806-7f974e80-c424-11e9-9df8-52df07353ac5.png)

可以从图上看到，在原来的登录的基础上增加了菜单表和用户菜单表，一个用户可以对应有多个菜单。那么他在登录后就获得了相应的菜单集合，在页面左侧加载即可。

对于那些在界面上没显示的菜单在后台路由中也是要检查下权限的，要不然像用户管理/userList这条路由虽然没有配给某个测试账号，但他可以直接在地址栏输入/userList进行访问。所以在Middleware中需要增加判断用户是否有此page_url的访问权限。
```
/** 需要用户菜单权限*/
    async userPermission(req, res, next) {
       //先判断用户session
        if (!req.session || !req.session.user || !req.session.user.id) {
            return res.redirect('/login');
        }
        let hasPower = false;
        //userMenu指当前用户拥有的菜单集合，请自行db查询
        userMenu.forEach(el => {
            if (el.page_url == req.route.path) {
                hasPower = true;
            }
        });
       if (!hasPower) {
            if (req.xhr) {
                return res.json({
                    state: false,
                    msg: "抱歉，您无此权限！请联系管理员"
                });
            }
            return res.send('抱歉，您无此权限！请联系管理员');
        }
        next();
    }
```
路由中增加中间件的执行
```
router.get('/userList', auth.userPermission, main.showUser);
```
这样当浏览器请求/userList路由时，先执行中间件userPermission方法，判断用户是否拥有此url权限，如果hasPower为false，即没有权限，直接返回输出。如果有权限，再执行相应controller中方法。

至此，基本的用户登录，及用户菜单权限已设计完成。但如果需要进一步权限控制到页面中的按钮、对用户进行分组设置权限等，还得再进行补充完善。

#### 按钮（控件）
上面权限系统控制到了页面，如果页面中不同用户操作按钮（控件）权限需要区分，比如常见的增、删、改操作，还需要进一步权限设计。

页面列表数据查看、新增、修改、删除等各种操作，都需要通过ajax将数据或参数提交给对应的接口，然后接口再将结果返回，所以我们可以通过限制接口的访问来做到对页面按键功能的控制。

可以把各种按钮也看成是菜单项，对前面的菜单表进行扩充，增加（控件地址、是否显示）两个字段即可。

其中控件地址这个字段，就是我们访问接口数据的路由地址，比如说获取单个订单数据是通过指定主键id，它的路由接口一般设计成'/api/user/:id'。这些路由地址是我们自己设计且在整个项目中都是唯一的。所以在开发时，我们可以在后台做个中间件拦截，通过用户是否有这个菜单项对应的接口路由地址，来判断用户是否有访问该接口的权限。

是否显示字段，是为了在输出菜单列表时将这些按钮菜单项隐藏，不在界面中显示出来。
![image](https://user-images.githubusercontent.com/3664948/63414535-a78baf80-c42e-11e9-9b14-1f6ddd3bc77a.png)

#### 角色（职位）
当系统中用户增多，对每一个用户都需要单独设置权限这种重复劳动工作量增大的时候，势必要考虑用户组了，在我们windows系统中早就有用户组的概念了。有了用户组概念，就可以先设置好某个用户组的权限，然后对于新加进来的用户只需加入之前配置好的用户组中，即新加进来的用户就拥有了用户组的权限。

考虑到现实情况，一个公司或企业的组织架构通常是有很多个部门组成，部门下面会有相应的职位，比如部门经理、部门员工等。用职位代替角色或用户组更让人能够容易理解。不同的职位有不同的工作职责，也有不同的操作权限。而部门主要是为了方便对职位进行分组管理，如果职位多时没有对应的分组，查询不方便，如果有相似的职位，也容易混淆。

对于企业来说，人员由于流动关系可能会经常变化，而职位则相对来说是比较固定的，所以权限绑定职位更合适，而不权限直接绑定用户。当一位员工更换岗位时，只需要更改他所绑定的职位，对于新入职的员工，也只需要绑定他所入职岗位，他们就可以拥有该职位的所有权限。

当然也会存在一些特殊的需要，比如说某人与同事都隶属于同一个职位，但他是老员工可以拥有更多的权限，这时可以增加一个新职位（通过制定职位级别）来区分他们的权限。

又比如说，如果权限需要限制部门访问权限，而该部门内的职位只能设置当前部门对应的权限，如果有员工需要跨部门拥有其他权限时，可以通过更改用户账号绑定多职位的方式来实现，也就是说一个员工他只可以绑定一个主部门，但他可以同时拥有多个职位，这样它的权限就是多个职位权限的集合。

这样在数据库表设计中需要调整的是新加入职位表（职位id、职位名称、部门id、菜单权限）和部门表（部门id、部门名称、部门编码、上一级部门id），并在用户表中增加职位id、部门id。

职位表是绑定在部门下的权限角色，菜单权限字段直接与菜单项进行关联，不同职位可以设置不同的权限（设置可查看与操作的菜单项）

职位表还需要存储与部门表的关联项：部门表id。如果为了不关联查询，也可以直接冗余存储部门编码、和部门名称字段（直接存储这个冗余字段，是为在需要显示职位所属部门时，不需要从部门表中关联查询，部门名称设置后更改的机会不大，但查询是每次都固定需要查询），所以冗余字段的设置减少了查询表次数，不过要在程序中确保两边表的数据更新一致。

部门表它相当于权限分组，可以根据企业的部门结构，创建对应的结构记录，这样也方便企业对系统权限关系更加容易理解。当然也可以根据需要设置虚拟部门出来管理。

为了以后扩展需要，需要添加部门编码字段，编码从01开始一直累加到99，当然如果部门超过99个的话，要么增加到3位数，要么当前框架已不能支持业务的发展需要思考新的架构了。

编码每增加一级，在01后面自动增加”0x“，编码的长度跟部门分级深度相关。

综合以上权限设计思路，最终整理出的数据表关系图为：
![image](https://user-images.githubusercontent.com/3664948/63416044-47e2d380-c431-11e9-867a-88539afccfbe.png)

整个权限控制就4张表，如果现实情况不太用得到组织部门，还可以把部门这张表去掉，并把职位表改成角色表，这样最精简的权限控制数据库表就设计完成了。

--------------

上一节主要讲了后台数据库表的设计，这节主要根据4张表来设计界面，主要有用户管理页面、菜单管理页面、部门管理页面、职位管理页面。

前端采用inspinia模板，有些插件略有增减，对话框采用layer，树菜单采用ztree，表格采用bootstrap-table，具体可以看源码。
express中采用ejs-mate模板，是ejs扩展版本，支持layout。

#### 用户管理列表
![image](https://user-images.githubusercontent.com/3664948/63746313-22dddd00-c8d7-11e9-9b6e-c4c8e1aa1499.png)


#### 编辑用户
编辑用户中主要选择用户所属部门和职位，其中职位是可以多个，采用树型结构勾选即可。
![image](https://user-images.githubusercontent.com/3664948/63418001-eae91c80-c434-11e9-8dd4-a6d7db1a6b2e.png)

#### 菜单管理列表
菜单列表采用bootstrap-table，并使用tree-grid插件显示菜单层级关系。
![image](https://user-images.githubusercontent.com/3664948/63417187-5e8a2a00-c433-11e9-92b7-78ff1faac9c0.png)

#### 编辑菜单
编辑菜单主要是设置上下级菜单关系，页面地址和控件地址，如果这个菜单只有控件地址，不在左侧树菜单显示的，需要将是否显示设为隐藏。
![image](https://user-images.githubusercontent.com/3664948/63417265-824d7000-c433-11e9-9dff-4cb9dc973a65.png)

#### 部门管理列表
![image](https://user-images.githubusercontent.com/3664948/63417320-9c874e00-c433-11e9-98a0-9777e5a8cf17.png)

#### 职位管理列表
左侧可以通过点击部门树，来筛选该部门下的职位列表，方便显示。
![image](https://user-images.githubusercontent.com/3664948/63417380-b9bc1c80-c433-11e9-8701-967a2c9dfe05.png)

#### 编辑职位
编辑职位主要设置该职位名称及该职位所拥有的菜单项，菜单项是一个树型结构，打勾即表示该职位拥护此菜单项权限。
![image](https://user-images.githubusercontent.com/3664948/63417422-d193a080-c433-11e9-9c8a-a1e82d290bc2.png)

以上是最终所实现的界面效果。大部分都属于简单的列表和增删改页面，稍微复杂点是有些页面会涉及到树形菜单加载。

用户登录后，就获得了用户所属职位的菜单项集合，在用户每次请求路由中需增加权限判断，我们写在中间件中。
```
/** 用户鉴权*/
async authUserPermission(req, res, next) {
    if (!req.session || !req.session.user || !req.session.user.id) {
        return res.redirect('/login');
    }
    if (!req.session || !req.session.menu || req.session.menu.length == 0) {
        return res.send('抱歉，您无此权限！请联系管理员');
    }
    let targetUrl = req.route.path;
    let hasPower = false;
    req.session.menu.forEach(el => {
        if (el.page_url == targetUrl || el.control_url == targetUrl) {
            hasPower = true;
        }

    });
    if (!hasPower) {
        if (req.xhr) {
            return res.json({
                state: false,
                msg: "抱歉，您无此权限！请联系管理员"
            });
        }

        return res.send('抱歉，您无此权限！请联系管理员');
    }
    next();
}
```

在每次路由请求中先判断用户权限，如有权限则往下执行正常逻辑，如无权限直接返回。
```
router.get('/system/userList', auth.authUserPermission, system.showUserList);
router.get('/system/userEdit/:id', auth.authUserPermission, system.showUserEdit);
```

test账号新增用户报无权限演示：
![image](https://user-images.githubusercontent.com/3664948/63736028-578c6d00-c8b4-11e9-9825-59d232eba992.png)

项目地址：[https://github.com/ciey/NodeExpressAdmin](https://github.com/ciey/NodeExpressAdmin)

