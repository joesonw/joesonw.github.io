---
layout: post
title: "一个用于前端api请求缓存的npm库"
---


介绍一个新开源的一个小项目. 用来在前端项目上, 缓存请求结果在内存中. 适用于比较大型的应用, 存在多个地方可能重复请求同一个ID, 或者批量中会包含重复的ID. 为了节省资源以及时间, 写了这么一个小小的工具.

## 安装
> npm i bulkcache

## 实例

```
import { Cache, FilterFunction } from 'bulkcache';

interface User {
    id: number;
    name: string;
    email: string;
}

function idGetter(user: User) {
    return user.id;
}

async function fetch(id: number) {
    return await YourAPI.fetchUser(id);
}

async function bulkFetch(ids: number[]) {
    return await YourAPI.fetchALotOfUsers(ids);
}

function allIWantIsANameFilter(user: User) {
    return user.name;
}

const cache = new Cache(idGetter, fetch, bulkFetch);
cache.addFilter(FilterFunction(allIWantIsANameFilter));

console.log(await cache.fetch(1)); 
console.log(await cache.fetch(1)); 

cache.fetch(2)
console.log(await cache.fetch(2));

console.log(await cache.batchGet([1, 2]))
console.log(await cache.batchGet([1, 2, 3, 4]))
```


先来看看初始化的代码:
```
function idGetter(user: User) {
    return user.id;
}

async function fetch(id: number) {
    return await YourAPI.fetchUser(id);
}

async function bulkFetch(ids: number[]) {
    return await YourAPI.fetchALotOfUsers(ids);
}

function allIWantIsANameFilter(user: User) {
    return user.name;
}

const cache = new Cache(idGetter, fetch, bulkFetch);
cache.addFilter(FilterFunction(allIWantIsANameFilter));
```	

这里`idGetter`是为了给每条记录返回一个去重的值, 一般来说就是id.

`fetch`和`bulkcache`都是调用的部分, 可以是api, 也可以是任意请求, 只要返回合适的数据. `fetch`是用于按id获取一条数据, `bulkfetch`是用于批量获取, 也可以传空, 这样就会适用`fetch`来一条条的拿批量的数据

`allIWantIsANameFilter`是一个`Filter`. 这里示例说我们只要取`user.name`, 而不需要整个所有的数据.


然后就是使用的部分了.
```
await cache.fetch(1); 
await cache.fetch(1);
```
这里只会产生一次网络请求, 因为第二次请求的时候, 数据已经在内存里了.


```
cache.fetch(2)
console.log(await cache.fetch(2));
```
这里我们并没有`await`第一次的调用, 也就是第二次调用时, 第一次的结果还没返回, 但是还是只会产生一次网络请求.

原来是, 在第一次调用时, 已经创建了一个`Q.Deferred`. 第二次调用时, 发现了`Q.Deferred`, 就会直接等待第一次调用的请求完成, 然后`resolve`这个`defer`之后, 第二次调用也会拿到数据了.


最后
```
console.log(await cache.batchGet([1, 2]))
console.log(await cache.batchGet([1, 2, 3, 4]))
```
这里我们使用到了批量请求, 因为`1`和`2`已经存在于内存中了, 所以并不会产生任何网络请求, 直接返回.

第二次调用会产生一个批量网络请求, 但是请求中只会包含`3`和`4`, 因为`1`和`2`会直接从内存中拿出来一起返回给调用者


## 结语
这是最近游戏项目中算是比较常用的一个小功能, 这里拿出来分享给大家, 希望工作中能更轻快更好的完成需求.

最近快过年了, 放假了有时间了, 整理一些比较小巧好用的小包出来.

最后, 欢迎PR. 当然, 也欢迎Issue!

Till next time.
