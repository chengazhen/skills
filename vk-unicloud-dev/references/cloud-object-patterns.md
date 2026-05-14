# 云对象完整指南

本文档涵盖 vk-unicloud-router 云对象路由模式的完整用法，包括标准模板、内置 API、权限体系、拦截器和常见问题。

## 目录

- [云对象 vs 云函数的关键差异](#云对象-vs-云函数的关键差异)
- [云对象标准模板](#云对象标准模板)
- [this 内置方法完整列表](#this-内置方法完整列表)
- [内置权限体系](#内置权限体系)
- [_before 和 _after 拦截器](#_before-和-_after-拦截器)
- [前端调用云对象](#前端调用云对象)
- [两种模式混合使用](#两种模式混合使用)
- [常见问题](#常见问题)

---

## 云对象 vs 云函数的关键差异

| 特性 | 云函数 | 云对象 |
|---|---|---|
| 文件结构 | 一个文件 = 一个接口 | 一个文件 = 一组接口 |
| 标记 | 无需标记 | `isCloudObject: true` |
| 获取参数 | `event` 解构 | `data` 参数 + `this` 方法 |
| 获取 uid | `let { uid } = data;` | `let { uid } = this.getClientInfo();` |
| 获取 vk | `let { vk } = util;` | `vk = uniCloud.vk;`（顶部或 `_before` 中） |
| 权限控制 | 目录名 `pub/kh/sys` | 函数名前缀 `pub_`/`kh_`/`sys_` |
| 拦截器 | 全局中间件 | 内置 `_before`/`_after` |
| 内部调用 | 需要 vk.callFunction | 可直接 `await this.otherMethod()` |

---

## 云对象标准模板

```js
'use strict';
let vk = uniCloud.vk; // 全局 vk 实例
// 涉及的表名
const dbName = {
  //test: "vk-test", // 测试表
};

const db = uniCloud.database(); // 全局数据库引用
const _ = db.command; // 数据库操作符
const $ = _.aggregate; // 聚合查询操作符

const cloudObject = {
  isCloudObject: true, // 必须标记为云对象模式

  /**
   * 请求前处理
   * 文档: https://vkdoc.fsq.pub/client/uniCloud/cloudfunctions/cloudObject.html#before-预处理
   */
  _before: async function() {
    vk = uniCloud.vk; // 将 vk 定义为全局对象
    // let { customUtil, uniID, config, pubFun } = this.getUtil(); // 获取工具包
  },

  /**
   * 请求后处理
   * 文档: https://vkdoc.fsq.pub/client/uniCloud/cloudfunctions/cloudObject.html#after-后处理
   */
  _after: async function(options) {
    let { err, res } = options;
    if (err) {
      if (err instanceof Error) {
        return; // 系统错误，直接 return 不处理
      }
      return err;
    }
    return res;
  },

  /**
   * 获取列表（kh 权限，默认需登录）
   * @url client/moduleName.getList 前端调用的url参数地址
   */
  getList: async function(data) {
    let res = { code: 0, msg: '' };
    let { uid } = this.getClientInfo();
    // 业务逻辑开始-----------------------------------------------------------

    // 业务逻辑结束-----------------------------------------------------------
    return res;
  },

  /**
   * 公开接口（pub_ 前缀，无需登录）
   * @url client/moduleName.pub_getPublicInfo
   */
  pub_getPublicInfo: async function(data) {
    let res = { code: 0, msg: '' };
    // 无需登录即可访问
    return res;
  },

  /**
   * 管理接口（sys_ 前缀，需管理员权限）
   * @url client/moduleName.sys_deleteAll
   */
  sys_deleteAll: async function(data) {
    let res = { code: 0, msg: '' };
    let { uid } = this.getClientInfo();
    // 仅管理员可操作
    return res;
  },
};

module.exports = cloudObject;
```

### 简易模板

如果只需要基础结构：

```js
'use strict';
let vk;
const cloudObject = {
  isCloudObject: true,
  _before: async function() {
    vk = uniCloud.vk;
  },
  _after: async function(options) {
    let { err, res } = options;
    if (err) {
      if (err instanceof Error) return;
      return err;
    }
    return res;
  },

  getList: async function(data) {
    let res = { code: 0, msg: '' };
    let { uid } = this.getClientInfo();
    // 业务逻辑
    return res;
  },
};
module.exports = cloudObject;
```

---

## this 内置方法完整列表

**特别注意**：云对象内置 API 无法通过重写覆盖，业务函数命名需要规避这些名称。

### this.getClientInfo() - 获取客户端信息

```js
let { uid, os, appId, clientIP, platform, deviceId } = this.getClientInfo();
```

**返回值（常用字段）：**

| 参数 | 类型 | 说明 |
|---|---|---|
| uid | String | 框架通过 token 解析出来的 uid（可信任） |
| os | String | 客户端系统 |
| appId | String | 客户端 DCloud AppId |
| clientIP | String | 客户端 IP |
| platform | String | 客户端平台：h5、mp-weixin 等 |
| deviceId | String | 客户端设备标识 |
| locale | String | 客户端语言 |
| userAgent | String | 客户端 UA |
| uniIdToken | String | 客户端用户 token |
| source | String | 调用来源：client / function / http / timing / server |
| filterResponse | Object | 框架中间件返回值（middleware/modules 内的中间件） |
| originalParam | Object | 原始请求参数 |

### this.getUserInfo() - 获取当前登录用户信息

```js
let userInfo = await this.getUserInfo(); // 注意：需要加 await
```

返回 `uni-id-users` 表中除 password 和 token 之外的全部字段。

**缓存机制：** 同一次请求中多次调用 `await this.getUserInfo()` 只执行一次数据库查询。如果在本次请求中修改了用户信息并需要获取最新数据，应改用：

```js
let { uid } = this.getClientInfo();
// 方式一：直接查库
let newUserInfo = await vk.daoCenter.userDao.findById(uid);
// 方式二：修改并返回
let newUserInfo = await vk.baseDao.updateAndReturn({
  dbName: 'uni-id-users',
  whereJson: { _id: uid },
  dataJson: { nickname: '新昵称' },
});
```

### this.getUtil() - 获取工具包

```js
let { customUtil, uniID, config, pubFun, db, _, $ } = this.getUtil();
```

| 参数 | 说明 |
|---|---|
| customUtil | 自定义工具包 |
| uniID | uni-id 实例 |
| config | 全局配置信息 |
| pubFun | 自定义公共函数（`router/util/pubFunction.js`） |
| db | 数据库实例 |
| _ | 数据库操作符 = db.command |
| $ | 聚合查询操作符 = _.aggregate |

### this.getMethodName() - 获取当前调用的方法名

主要用于 `_before` 中判断当前调用的是哪个方法：

```js
_before: async function() {
  let methodName = this.getMethodName(); // 如 "getList"、"pub_getInfo"
},
```

### this.getParams() - 获取当前请求参数

主要用于 `_before` 中获取前端传来的参数：

```js
_before: async function() {
  let { a, b, c } = this.getParams();
},
```

### this.getUniIdToken() - 获取用户 token

```js
const token = this.getUniIdToken();
```

### this.getCustomClientInfo() - 获取自定义客户端信息

> vk-unicloud 版本 >= 2.19.4

需要先在前端调用 `vk.setCustomClientInfo` 设置，才能在云对象内获取到。

```js
const customClientInfo = this.getCustomClientInfo();
```

### this.getCloudInfo() - 获取云端信息

```js
const cloudInfo = this.getCloudInfo();
// { provider, spaceId, functionName, functionType: 'cloudobject', runtimeEnv }
```

### this.getUniCloudRequestId() - 获取请求 ID

```js
const requestId = this.getUniCloudRequestId();
```

### this.getHttpInfo() - 获取 HTTP 信息

仅在云对象 URL 化时可用：

```js
const httpInfo = this.getHttpInfo();
```

---

## 内置权限体系

### pub（无需登录即可访问）

满足以下**任意一条**规则即为 pub：

| 规则 | 示例 | 权重 |
|---|---|---|
| 函数名以 `pub_` 开头 | `pub_getList` | 3 |
| 文件名以 `pub` 命名 | `pub.js` 或 `pub.user.js` | 2 |
| 文件在 `pub/` 目录下 | `pub/user.js` | 1 |

### kh（需要登录才能访问）

满足以下**任意一条**规则即为 kh：

| 规则 | 示例 | 权重 |
|---|---|---|
| 默认（函数名无任何前缀） | `getList`、`getInfo` | 0 |
| 函数名以 `kh_` 开头 | `kh_getList` | 3 |
| 文件名以 `kh` 命名 | `kh.js` 或 `kh.user.js` | 2 |
| 文件在 `kh/` 目录下 | `kh/user.js` | 1 |

### sys（需要角色授权才能访问）

满足以下**任意一条**规则即为 sys：

| 规则 | 示例 | 权重 |
|---|---|---|
| 函数名以 `sys_` 开头 | `sys_getList` | 3 |
| 文件名以 `sys` 命名 | `sys.js` 或 `sys.user.js` | 2 |
| 文件在 `sys/` 目录下 | `sys/user.js` | 1 |

sys 类型函数通常用于 admin 端。框架会通过用户角色权限自动判断拦截请求。

### _（私有函数，禁止前端访问）

函数名以 `_` 开头则禁止前端访问，如 `_before`、`_after`、`_myHelper`（权重 99）。

### 权重优先级

当函数同时满足多个类型时，取权重大的一方：

- `pub.js` 文件中的 `kh_getList` → **kh** 权限（函数名权重 3 > 文件名权重 2）
- `user.js` 文件中的 `pub_getList` → **pub** 权限（函数名权重 3 > 默认权重 0）
- `pub/` 目录下 `user.js` 的 `sys_delete` → **sys** 权限（函数名权重 3 > 目录权重 1）

---

## _before 和 _after 拦截器

### _before（预处理）

在调用常规方法之前执行，一般用于拦截器、身份验证、参数校验、定义全局对象。

```js
_before: async function() {
  vk = uniCloud.vk;
  let methodName = this.getMethodName();
  let params = this.getParams();

  // 跳过公开接口
  if (methodName.indexOf('pub_') === 0) return;

  // 自定义权限检查
  let { shop_id } = params;
  let userInfo = await this.getUserInfo();
  let { shop_ids = [] } = userInfo;

  if (vk.pubfn.isNull(shop_id)) {
    return { code: -1, msg: '店铺id不能为空' };
  }
  if (shop_ids.indexOf(shop_id) === -1) {
    return { code: -1, msg: `无权限操作店铺【${shop_id}】` };
  }
},
```

**注意：** 用户登录检测框架已内置，无需在 `_before` 中再写登录判断。

**传递数据到后续方法：** 在 `_before` 中通过 `this.xxx = value` 挂载数据，后续方法通过 `this.xxx` 获取（注意不要与方法名重复，建议加前缀）：

```js
_before: async function() {
  vk = uniCloud.vk;
  this.startTime = Date.now();
  this._shopInfo = await vk.baseDao.findByWhereJson({
    dbName: 'shops',
    whereJson: { owner_id: this.getClientInfo().uid },
  });
},

getList: async function(data) {
  let shopInfo = this._shopInfo; // 获取 _before 中挂载的数据
  let timeCost = Date.now() - this.startTime;
  // ...
},
```

### _after（后处理）

在方法执行后运行，用于再加工返回结果或处理错误：

```js
_after: async function(options) {
  let { err, res } = options;
  if (err) {
    if (err instanceof Error) {
      return; // 系统错误，直接 return 不处理
    }
    return err; // 业务错误，返回给前端
  }
  // 可在此加工 res
  res.timeCost = Date.now() - this.startTime;
  return res;
},
```

---

## 前端调用云对象

### 方式一：vk.callFunction（通用，URL 用点号分隔方法名）

URL 格式：`service 内的目录名 + 文件名.方法名`

```js
// 回调形式
vk.callFunction({
  url: 'client/order.getList',
  title: '请求中...',
  data: { pageIndex: 1, pageSize: 10 },
  success: (data) => { console.log(data.rows); },
});

// async/await 形式
let data = await vk.callFunction({
  url: 'client/order.getList',
  data: { pageIndex: 1, pageSize: 10 },
});
```

### 方式二：uni.vk.importObject（云对象独有，面向对象调用）

分两步：先导入云对象，再调用方法。

```js
// 第1步：导入云对象（路径对应 service/ 下的文件，不含 .js 后缀）
const orderObj = uni.vk.importObject('client/order');

// 第2步：调用方法（回调形式）
orderObj.getList({
  title: '加载中...',
  data: { pageIndex: 1, pageSize: 10 },
  success: (data) => { console.log(data.rows); },
});

// 第2步：调用方法（async/await 形式）
let data = await orderObj.getList({
  title: '加载中...',
  data: { pageIndex: 1, pageSize: 10 },
});
```

**Vue3 App 模式注意事项：** 不可直接在页面生命周期外导入云对象，需延迟执行：

```js
var orderObj;
setTimeout(() => {
  orderObj = uni.vk.importObject('client/order');
}, 10);
```

### importObject 高级选项

```js
// 指定其他 router 云函数
const obj = uni.vk.importObject('client/order', { name: 'router2' });

// 设置默认 title
const obj = uni.vk.importObject('client/order', { title: '请求中' });

// 设置默认请求参数（每次调用自动合并）
const obj = uni.vk.importObject('client/order', { data: { shop_id: '123' } });
// 调用时: obj.getList({ data: { a: 1 } })
// 最终发送: { shop_id: '123', a: 1 }

// 开启简易传参模式（省略 data 包裹）
const obj = uni.vk.importObject('client/order', { title: '请求中', easy: true });
let data = await obj.getList({ pageIndex: 1, pageSize: 10 });

// 开启加密通信
const obj = uni.vk.importObject('client/order', { encrypt: true });
```

---

## 两种模式混合使用

云函数和云对象可在同一个 router 下共存，互不影响：

```
service/
├── client/
│   ├── order.js                   # 云对象模式
│   └── product/                   # 云函数模式
│       ├── pub/
│       │   └── getList.js
│       └── kh/
│           └── add.js
├── user/                          # 云函数模式
│   ├── pub/login.js
│   └── kh/getMyUserInfo.js
```

### 跨模式调用

```js
// 在云对象中调用其他云函数或云对象（同一个 router 下，name 可不传）
let callRes = await vk.callFunction({
  name: 'router',
  url: 'client/user.test',
  clientInfo: this.getClientInfo(),
  data: { a: 1 },
});
```

---

## 常见问题

### 同一个云对象内 A 函数调用 B 函数

通过 `await this.xxx()` 直接调用，xxx 为函数名：

```js
getList: async function(data) {
  // 调用同一云对象内的 getTotal 方法
  let total = await this.getTotal(data);
  // ...
},
getTotal: async function(data) {
  return await vk.baseDao.count({ dbName: 'orders', whereJson: {} });
},
```

### A 云对象调用 B 云对象

**不建议** - 每个云对象之间业务逻辑隔离，跨对象调用会导致耦合度高、不易维护。

如果涉及全局公共函数，写在 `router/util/pubFunction.js`，通过以下方式调用：

```js
let { pubFun } = this.getUtil();
let xxxRes = await pubFun.xxx();
```

如果确实需要跨对象调用：

```js
// 方式一（推荐，vk-unicloud >= 2.9.0）
let callRes = await vk.callFunction({
  name: 'router',
  url: 'client/user.test',
  clientInfo: this.getClientInfo(),
  data: { a: 1 },
});

// 方式二（通用）
let callRes = await uniCloud.callFunction({
  name: 'router',
  data: {
    $url: 'client/user.test',
    data: { a: 1 },
  },
});
```

### 云对象本地运行

右键 router 目录 → 配置运行测试参数，在 `router.param.json` 中设置：

```json
{
  "uni_id_token": "",
  "$url": "client/order.getList",
  "data": { "a": 1 }
}
```

然后右键 router → 运行本地云函数（快捷键 Ctrl+R）。
