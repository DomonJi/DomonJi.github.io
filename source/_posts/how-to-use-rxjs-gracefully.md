---
title: 如何更优雅地使用 Rxjs
date: 2017-08-18T14:32:13.000Z
tags:
  - rxjs
  - frontend
  - vue-rx
category: Rxjs
---

## 引言

我们的 DCE 项目使用 Rxjs 作为数据流动的框架，这种响应式的编程思想很适合 DCE 这样的数据流动复杂的应用，却也带来不小的使用门槛。近日，在审阅我们代码的过程中，我发现了很多不规范甚至是不正确的 Rxjs 使用方法，不仅造成了代码丑陋，冗余，甚至会引起意想不到的bug。

我找了几处经典的案例，看看我们如何更优雅地使用 Rxjs 。

<!--more-->

## 案例

### 案例 1
我们先来看一处简单的案例
```js
userInfo$.subscribe(v => {
  settingApi.getAccessKeys(v.name)
    .then(accessKeys => {
      this.accessKeys = accessKeys;
      this.updateRemoteBuildCommand(this.rules.remoteBuildConfig);
      this.updateWebhook();
    });
});
```
这样的代码在项目中很常见，乍一看似乎没有什么问题，但我认为暗藏玄机：
1. `Promise` 和 `Observable` 混用，造成代码可读性低。
2. 在 `subscribe` 中发起请求之后修改本地数据，此处操作相当于 `mergeMap`，这样一来就会有一个问题，先发出的请求由于网络延迟返回的比后发出的请求返回的还慢，这样子当先发出的请求到达了之后就用旧的数据把新的数据覆盖，造成不可预期的后果。

经过一番重构，我认为这样子会更加优雅：
```js
userInfo$
  .pluck('name')
  .switchMap(settingApi.getAccessKeys)
  .subscribe(accessKeys => {
    this.accessKeys = accessKeys;
    this.updateRemoteBuildCommand(this.rules.remoteBuildConfig);
    this.updateWebhook();
  });
```
使用 `switchMap` 便解决了上述两个问题。关于 'switchMap' 以及各种 map的区别，请看：
[mergeMap](http://cn.rx.js.org/class/es6/Observable.js~Observable.html#instance-method-mergeMap)
[concatMap](http://cn.rx.js.org/class/es6/Observable.js~Observable.html#instance-method-concatMap)
[switchMap](http://cn.rx.js.org/class/es6/Observable.js~Observable.html#instance-method-switchMap)
[exhaustMap](http://cn.rx.js.org/class/es6/Observable.js~Observable.html#instance-method-exhaustMap)

这段代码第三行等价于：
```js
.switchMap(name => Rx.Observable.fromPromise(settingApi.getAccessKeys(name)))
```

---
### 案例 2

```js
appApi.create(params)
  .then(res => {
    switch (res.status) {
      case 201:
        // 应用详情的数据是从列表数据中获取的,
        // 而列表数据依赖整个 Stream 系统,
        // 因此这里只需要等待列表更新然后再跳转即可。
        hub.hub$$.next('app');
        appsVm$$.subscribe(apps => {
          if (_.find(apps, { name: this.appName })) {
            this.$router.push({ name: 'AppDetail', params: { name: this.appName } });
          }
        });
        break;

      case 207:
        res.data.Errors.forEach(err => {
          new Noty({
            text: err,
            type: 'error',
          }).show();
        });
        break;

      default:
    }
  });
```
此处代码来自 `new-from-registry.js`

可以看到，这段代码的目的是发起一个创建应用的请求，成功后去订阅 `appsVm$$` 等到应用列表中确实存在新建的应用便跳转到应用详情页中。
这段代码有两个问题：
1. `Promise` 和 `Observable` 混用，造成代码可读性低。
2. 在请求成功后订阅了 `appsVm$$` ，却并没有任何地方去取消这个订阅，也就导致了一个非常严重的bug，即创建应用后，在容器组列表中点击刷新按钮也会跳转到应用详情页中，因为执行了 `hub.hub$$.next('pod')`。

经过重构一番之后，我认为合理并且优雅的代码应该如下

```js
const [succeed$, error$] = Rx.Observable.fromPromise(appApi.create(params))
                             .partition(res => res.status === 201);

  succeed$.do(() => hub.hub$$.next('app'))
    .concatMapTo(appsVm$$)
    .first(apps => _.some(apps, app => app.name === this.appName))
    .subscribe(() => {
      this.$router.push({ name: 'AppDetail', params: { name: this.appName } });
    });

  error$.subscribe(res => {
    res.data.Errors.forEach(err => {
      new Noty({
        text: err,
        type: 'error',
      }).show();
    });
  });
```
将 `Promise` 转化为 `Observable` ，通过 `partition` 将其分为成功和错误的两条流。
我们关注成功的这个 `Observable`，在 `do` 中做副作用，之后将其映射为 `appsVm$$`，取第一个符合条件的数据对其订阅。这边 `first` 操作符会在得到第一个符合的数据之后立刻发出 `complete`，订阅也随即停止。

关于 `partition`： [partition](http://cn.rx.js.org/class/es6/Observable.js~Observable.html#instance-method-partition)

---


### 案例 3

```js
networkApi.create(params, params.nodeAddress, tenant, userName)
  .then(res => {
    const networksNumber = this.networks.all.length;
    const interval = setInterval(() => {
      hub.hub$$.next('network');
    }, 2000);
    const temp_ = networksVm$$.subscribe(networks => {
      if (networks.all.length > networksNumber) {
        new Noty({
          text: '网络创建成功。',
          type: 'success',
        }).show();
        // 把参数恢复到默认
        this.data = _.clone(DEFAULT);
        this.isShow = false;
        this.creating = false;
        temp_.unsubscribe();
        clearInterval(interval);
        this.$router.push({
          path: `/network-detail/${res.Id}`,
        });
      }
    });
  })
  .catch(rej => {
    this.creating = false;
    new Noty({
      text: `网络 ${this.data.name} 创建失败，请重试！<br>错误信息：${rej.data.message}`,
      type: 'error',
    }).show();
  };
```
这段代码来自 `create-network.js`

它的目的是请求成功后不断地去轮询，直到网络真的创建成功，再跳转路由。
这段代码同时存在以下几个问题：
1. `Promise` 和 `Observable` 混用，造成代码可读性低。
2. 同时还有 `setInterval`混入。
3. 自己手动的取消 Interval 和取消订阅，造成了不必要的麻烦和代码冗余。

这段逻辑使用Rxjs重构的难点便是在这个轮询上。于是考虑使用 `Observable.interval` 来构造一个定时产生数据的 `Observable`。
```js
Rx.Observable
  .fromPromise(networkApi.create(params, params.nodeAddress, tenant, userName))
  .concatMap(res => Rx.Observable.interval(1000).mapTo(res))
  .do(() => hub.hub$$.next('network'))
  .combineLatest(networksVm$$)
  .first(([res, networks]) => _.some(networks.all, network => network.id === res.Id))
  .subscribe(this.onSuccess, this.onError);
```
将 `Promise` 得到的结果映射为每隔1秒发送一遍的 `Observable`，在 `do`中做 `hub.hub$$.next('network')`，同时结合 `networksVm$$`，订阅第一个符合条件（也就是网络成功被创建）的数据，展示创建成功的信息，在异常的时候调用`onError`展示错误信息。这边把成功和失败的信息展示抽离出来成为单独的方法。

---

### 案例 4
```js
nodeApi.removeK8s(node).then(res => {
  nodeApi.down(node)
  .then(() => {
    const timer = setInterval(() => {
      const promise = nodeApi.detail(node.id)
        .then(res => {
          if (res.Status.State === 'down') {
            clearInterval(timer);
            return nodeApi.remove(node)
              .then(() => {
                hub.hub$$.next('all');
                new Noty({
                  text: '成功删除主机',
                  type: 'success',
                }).show();
              });
          }
          return false;
        });
      return promise;
    }, 3 * 1000);
  })
  .catch(rej => {
    new Noty({
      text: `删除主机失败，错误信息: ${rej.data.message}`,
      type: 'error',
    }).show();
  });
}).catch(rej => {
  new Noty({
    text: `删除主机失败，错误信息: ${rej.data.message}`,
    type: 'error',
  }).show();
});
```
这段代码已经是惨不忍睹，嵌套了4层`then`加一层`setInterval`。
它的目的是，发送两个请求之后去轮询主机详情，直到状态为 down 时再发起请求移除这个主机节点，之后提示成功或者失败的消息。

首先我们可以用 `Promise.all` 把连续的两个 `Promise`合并
再转换为每隔3秒轮询一次的 `Observable`
取第一个状态为 `down` 的数据，调用api移除这个节点，
然后通知 hub，最后订阅这个流，输出成功和错误的提示消息。

重构后的代码如下。
```js
Observable.fromPromise(Promise.all([
  nodeApi.removeK8s(node),
  nodeApi.down(node),
]))
  .concatMapTo(Observable.interval(3000))
  .switchMap(() => nodeApi.detail(node.id))
  .first(res => res.Status.State === 'down')
  .switchMap(() => nodeApi.remove(node))
  .do(() => hub.hub$$.next('all'))
  .subscribe(successNoty, errorNoty);
```

---

### 案例 5

```js
socket$$.subscribe(job => {
  // 因为 job 推送来的数据并不是 build 的数据
  // 所以我还得根据 jobId 查到具体的 build 的数据
  // 索性每次拿到 job 就重新拿列表
  if (!job) return;
  const tempName = `${this.repo.namespace.name}/${this.repo.name}`;
  if (job.entity.name === tempName) {
    const getRepoBuildsPromise = this.getRepoBuilds();
    if (job.state.type === 'Succeed') {
      getRepoBuildsPromise.then(() => {
        if (this.builds.length) {
          this.updateTagList(this.builds[0].ImageTag);
        }
      });
    }
  }
});
```

这段代码也是我们的项目中很常见的，把所有的逻辑都放在 `subscribe` 中，
这样就让 `subscribe` 中逻辑很臃肿。

我们先来看一下这段代码的逻辑：
订阅了`socket$$`，当 `job.entity.name === tempName` 时去调用 `this.getRepoBuilds`这个函数，这个函数里发起了一个请求之后修改了本地数据（实际上这样的函数很不纯，我觉得不应该在这个函数里修改本地数据），并返回一个 `Promise`，如果 `job.state.type === 'Succeed'`，那么在这个 `Promise` resolve 之后去修改本地数据。

这段代码的难点就在于如何等到这个 `Promise` resolve 之后再去修改本地数据。
实际上Rx是很强大的，你能想到的任何对流的操作几乎都有对应的方法，比如这个例子，我们就可以使用 `delayWhen`这个操作符。它接收的是一个方法。

重构之后的代码如下：
```js
socket$$
  .filter(job => job && job.entity.name === tempName)
  .delayWhen(this.getRepoBuilds)
  .filter(job => job.state.type === 'Succeed')
  .subscribe(() => {
    if (this.builds.length) {
      this.updateTagList(this.builds[0].ImageTag);
    }
  });
```
上面第三行代码等价于：
```js
.delayWhen(() => Rx.Observable.fromPromise(this.getRepoBuilds()))
```
关于 `delayWhen`： [delayWhen](http://cn.rx.js.org/class/es6/Observable.js~Observable.html#instance-method-delayWhen)

---
## 更重要的一点

我们DCE代码中很多地方，比如创建镜像，都会给某个按钮点击事件绑定一个方法，在这个方法里面去`subscribe`一个流，然后去做一些操作，我觉得这样是不合理的，因为如果没有对点击加以限制，用户不断的点击，就会不断的去`subcribe`，会引起一些不可预料的结果，甚至引起性能问题，尤其是在一些点击之后需要轮询操作的地方。实际上我们应该在点击的时候做 `next` 操作而不是 `subcribe`操作。

**所以上面所有案例的重构并没有解决根本的问题！**
我认为所有流的订阅都应该放在组件初始化的时候，组件的方法只对流做 `next` 操作和修改本地数据。

比如对于一个点击事件，更合理的做法是利用 `vue-rx`的一个指令 `v-stream:click` 把点击事件转换成一个流，与你需要订阅的流合并，在组件`mounted`的时候去`subcribe`这个点击事件的流。举例如下：

```js
<template>
  <button v-stream:click="{ subject: click$, data: someData }"></button>
</template>

<script>
import someApi from '...somePath'
import someStream$$ from '...somePath'

export default {
  data() {
    return {
      someData: 'someData',
    };
  },
  subscriptions() {
    this.click$ = new Rx.Subject();
  },
  mounted() {
    this.$subscribeTo(
      this.click$.switchMap(someApi.someMethod)
        .concatMapTo(Rx.Observable.interval(1000))
        .combineLatest(someStream$$)
        .first(/** some logic */),
      onNext, onError, OnComplete,
    );
  },
}
</script>
```

---

值得注意的是：`vue-rx` 中提供了一个 `$subscribeTo`方法，可以自动的在组件销毁时 `unsubscribe` 所有的订阅，很多情况下可以代替 `subscribe`方法。

于是我们可以给 `Observable` 的原型链上添加一个方法，便于我们链式调用。

```js
// 给 Observable 添加的方法
// 在 Vue 实例中使用 observable.$subscribeTo(this, onNext, onError, onComplete)
// 等价于使用 this.$subscribeTo(observable, onNext, onError, onComplete)
Rx.Observable.prototype.$subscribeTo = function $subscribeTo(_vm, onNext, onError, onComplete) {
  Vue.prototype.$subscribeTo.call(_vm, this, onNext, onError, onComplete);
}
```

上述 `mounted` 函数便可更加简洁：
```js
mounted() {
  this.click$.switchMap(someApi.someMethod)
    .concatMapTo(Rx.Observable.interval(1000))
    .combineLatest(someStream$$)
    .first(/** some logic */)
    .$subscribeTo(this, onNext, onError, onComplete);
},
```

现在我们项目中对 `vue-rx` 的使用还只停留在给vue实例上加一个subscriptions对象，实际上 `vue-rx`还提供了很多很便利的方法和指令：
`$watchAsObservable`可以观察一个`data`数据产生一个流。
`$createObservableMethod`可以创建一个流，在一个方法每次调用的时候发出它的参数。
`v-stream:click`可以从一个点击事件上创建一个流，每次点击可以发出你想要发出的数据。
还有`$eventToObservable`、`$fromDomEvent`等，大家可以参阅 [vue-rx 文档](https://github.com/vuejs/vue-rx/blob/master/README-CN.md)


