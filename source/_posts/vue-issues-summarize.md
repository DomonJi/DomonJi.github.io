---
title: vuejs 踩坑及经验总结
date: 2017-05-22T20:38:11.000Z
tags:
  - vuejs
  - frontend
category: vuejs
---


# 趟坑总结
## 问题描述
在使用 v-for repeat 组件时控制台会出现警告：
![image](https://cloud.githubusercontent.com/assets/12231277/26719687/0f18e08c-47b8-11e7-8641-53f4a58679e1.png)
## 解决方法
在组件标签上使用 v-for ：
* 加 [:key](https://cn.vuejs.org/v2/guide/list.html#key)
* 使用 template 标签包裹该组件，再在 template 标签 上使用 v-for。
---

<!-- more -->

# 趟坑总结
## 问题描述
在 Vue data 属性中定义的变量名如果以 "_" 开头，就不能正确的赋值和渲染
## 问题原因
https://github.com/vuejs/vue/issues/2098
## 解决方法
变量名不要以 "_" 开头
---

# 趟坑总结
## 问题描述
在我们的前端组件库 dao-option 组件中 中使用 [:key](https://cn.vuejs.org/v2/guide/list.html#key) 时，如果 v-for 枚举的值是 Object，但是 :key 中的值是简单类型时，dao-options 选出来的值并不在 dao-select 原本储存的 options 中。
![dao-select-062601](https://user-images.githubusercontent.com/19180212/27534266-ff8756b4-5a98-11e7-8c5f-bd9c8b650d7c.png)

## 问题原因
在组件中使用 [:key](https://cn.vuejs.org/v2/guide/list.html#key) 时，如果 v-for 枚举的值是 Object，但是 :key 中的值是简单类型时，当 Object 地址改变，:key 中解析值没变时，组件会被复用，并不会被销毁。例如： dao-style 的 dao-select 中 dao-option 因为上述的描述导致组件没被销毁，最终 dao-options 选出来的值不在 dao-select 原本储存的 options 中。

## 解决方法
:key 绑定复杂类型
---

# 趟坑总结
## 问题描述
在 Vue Router 中定义路由为 a/:id 形式时，从 a/1 跳转到 a/2 时，不会触发 component 的 created 等方法。
## 解决方法
1.```<router-view :key="$route.path"></router-view>```
2. 在 component 中监听 $route.path 的变化，手动触发 created 内的方法。
P.S. 可以参照 [router-link](http://router.vuejs.org/zh-cn/essentials/dynamic-matching.html) 中的 响应路由参数的变化
---

# 趟坑总结
## 问题描述
不能直接 watch subscriptions 中的变量，回调不会执行。
## 解决方法
要先在 data 中注册该变量


