---
title: 'vue动态添加的路由页面刷新时失效'
date: 2021-02-24 11:57:47
tags:
- 前端学习
- 工作积累
- Vue
nav:
- Vue
categories:
- Vue积累
image: img/xhcat/cat4.jpg
---
 

{% asset_img me.png This is an example image %}

## 问题描述
昨天在做vue后台管理系统有关权限页面动态添加到路由的功能时，遇到一个问题：动态添加的路由页面，在页面刷新时出现了404的情况。

## 场景
后台管理系统的权限控制是通过在前端页面定义权限code， 把code给后台同学保存配置到表中，之后根据后台返回的权限code列表与前端页面配置的code菜单列表做筛选匹配，code相等的页面就是有权限的页面，再通过router.addRoute()动态添加到路由中，有权限的路由才可以被访问，否则会提示无权限。

固定路由一开始就会放在new Router中，比如登录页面login

### 接口返回
{% asset_img api.png api image %}

### 前端菜单定义
{% asset_img menu.png menu image %}

### vuex中的方法
{% asset_img vuex.png menu image %}
{% asset_img addRoute.png menu image %}


## 出现的问题
登录后，通过调用vuex中的方法，完成获取权限code，动态筛选权限路由页面操作，然后通过router.addRoute()将有权限菜单添加到路由中，进入动态添加的路由页面，刷新页面出现404
 
## 原因分析
页面刷新时，路由重新初始化，动态添加的路由此时已不存在，只有一些固定路由（比如登录页面）还在，所以出现了404的情况

## 解决方案
VUEX store中存储的数据会在页面刷新时清空。
在路由的全局导航router.beforeEach处做个判断，根据VUEX中存放的list是否有值来判断页面是否是刷新，如果不为0，则是第一次登陆，登录后会走匹配路由的方法，不会有问题;如果list.length为0，就为刷新页面，需要重新执行路由匹配，重新添加动态路由即可。

### 实现代码 route/index.js的导航守卫中添加逻辑判断

#### ---------router.js-------------
```javascript
const constantRoutes = [
    {
        path: '/',
        redirect: '/login'
    },
    {
        path: '/login',
        name: 'login',
        meta: {
            auth: false
        },
        component: () => import('@/views/login')
    },
    {
        path: '/layout',
        name: 'layout',
        meta: {
            auth: true
        },
        component: () => import('@/views/layout/index'),
        children: [
            {
                path: '/index',
                name: 'index',
                component: () => import('@/views/home')
            }
        ]
    },
    {
        path: '*',
        component: () => import('@/views/error/404')
    }
]

Vue.use(VueRouter)
const createRouter = () =>
    new VueRouter({
        routes: constantRoutes
    })
export function resetRouter() {
    const newRouter = createRouter()
    router.matcher = newRouter.matcher // reset router
}
const router = createRouter()
 
//页面刷新后重新设置权限页面动态路由，防止出现动态路由404问题
const reSetPermissionList = to => {
    return new Promise((resolve, reject) => {
        if (to.path !== '/login' && store.state.permission.permissionList.length === 0) {
            store
                .dispatch('permission/getPermissionList')
                .then(() => {
                    resolve('permCode')
                })
                .catch(error => {
                    resolve('permCode')
                })
        } else {
            resolve()
        }
    })
}
router.beforeEach((to, from, next) => {
     
    const accessToken = localStorage.getItem('accessToken')
    if (_.isEmpty(accessToken)) {//是否已经登录 否 去登陆页面
        next({
            path: '/login',
            query: {
                redirect: to.fullPath
            }
        })
    } else { //已登录用户进入页面
        if (to.path === '/login') {
            next({ path: '/index' })
        } else {
             reSetPermissionList(to).then(data => {
                    data === 'permCode' ? next({ path: to.path, query: to.query }) : next()
                })
        }
    }
   
})
```
## 总结 
主要通过在全局导航处判断VUEX中的数据是否存在，判断页面是否刷新，是的话重新走一遍权限路由匹配的方法。

## 结尾小彩蛋
第一次遇到这个问题，比较懵缺......一时找不到原因，后来通过查找，看到一篇文章，里面的情况跟我遇到的差不多，在此贴上链接，感谢这位网友的分享 ^3^
{% link 参考文章链接--前往 https://blog.csdn.net/qq_31906983/article/details/88942965?utm_medium=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-8.add_param_isCf&depth_1-utm_source=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-8.add_param_isCf   %}