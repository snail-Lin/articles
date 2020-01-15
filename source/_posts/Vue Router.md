---
layout: post
title:  "Vue Router"
date:   2019-12-30
categories: Experience
excerpt: ""
tag:
- Vue
comments: true
---

### 一、动态路由匹配
#### 1.路由配置

**指定路由**
```javascript
const router = new VueRouter({
  routes: [
    // 动态路径参数 以冒号开头
    { path: '/user/:id', component: User }
  ]
})
```

**捕获所有路由或 `404 not found` 路由**
```javascript
{ path: '*', redirect: '/404', hidden: true },
{
  // 会匹配以 `/user-` 开头的任意路径
  path: '/user-*'
}
```

**高级匹配模式**
[https://github.com/pillarjs/path-to-regexp/tree/v1.7.0](https://github.com/pillarjs/path-to-regexp/tree/v1.7.0)

**嵌套路由（多层嵌套的组件，类似于navigation嵌套）**
```javascript
const router = new VueRouter({
  routes: [
    { path: '/user/:id', component: User,
      children: [
        {
          // 当 /user/:id/profile 匹配成功，
          // UserProfile 会被渲染在 User 的 <router-view> 中
          path: 'profile',
          component: UserProfile
        },
        {
          // 当 /user/:id/posts 匹配成功
          // UserPosts 会被渲染在 User 的 <router-view> 中
          path: 'posts',
          component: UserPosts
        }
      ]
    }
  ]
})
```

#### 2.接收参数：

```javascript
const User = {
  template: '<div>User {{ $route.params.id }}</div>'
}
```

#### 3.响应路由参数变化

**使用`watch`监听**
```javascript
const User = {
  template: '...',
  watch: {
    '$route' (to, from) {
      // 对路由变化作出响应...
    }
  }
}
```

**使用导航守卫方法**
```javascript
const User = {
  template: '...',
  beforeRouteUpdate (to, from, next) {
    // react to route changes...
    // don't forget to call next()
  }
}
```

### 二、导航
1.声明式导航


