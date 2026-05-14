---
name: vk-unicloud-dev
description: vk-unicloud-router 后端开发规范与前端调用方法。涵盖云函数路由模式和云对象路由模式两种开发方式、vk.baseDao 数据库操作、过滤器中间件、Dao 2.0、约定式路由映射、kh/pub/sys 权限模型、前端 vk.callFunction 和 uni.vk.importObject 调用。当用户提到以下任何内容时使用此 skill：vk-unicloud、vk-unicloud-router、编写云函数、云对象、cloudObject、isCloudObject、importObject、云函数路由、vk.baseDao、vk.callFunction、数据库增删改查、分页查询 select/selects、连表查询 foreignDB、过滤器中间件、kh/pub/sys 目录权限、Dao 2.0 数据访问层、用户权限校验 userInfo/uid、事务操作、数据库 schema 设计、封装 API 接口、_before/_after 拦截器。无论是创建新业务模块、编写后端接口、优化数据库查询，还是封装前端调用方法，都应使用此 skill 确保代码符合 vk-unicloud 框架规范。
---

# vk-unicloud-router 开发规范

基于 vk-unicloud-router 云开发框架的项目开发规范。vk-unicloud 封装了路由管理、权限控制、数据库操作（vk.baseDao）等能力，所有开发必须遵循框架约定，禁止使用原生 uniCloud API。

## 核心约束

以下规则不可违背，每一条都源于框架的实际运行机制：

1. **禁止原生 uniCloud API** - 不使用 `uniCloud.database()`、`db.collection()` 等原生方法，所有数据库操作通过 `vk.baseDao` 完成
2. **前端请求用 vk 框架方法** - 不使用原生 `uniCloud.callFunction`，使用 `vk.callFunction` 或 `uni.vk.importObject`（云对象），因为它们自动携带 token、处理错误和加载状态
3. **后端使用标准模板** - 云函数模式用 `main: async (event) => {}` 解构 event；云对象模式用 `isCloudObject: true` + 方法函数 + `this` 上下文
4. **约定式路由** - URL 直接映射到 `service/` 目录下的文件路径（云函数）或文件名.方法名（云对象），不需要手动配置路由
5. **权限由命名决定** - 云函数通过目录名 `pub/kh/sys` 决定；云对象通过函数名前缀 `pub_`/`kh_`/`sys_` 或文件名决定
6. **vk.baseDao 没有 find() 方法** - 查询单条用 `findById()` 或 `findByWhereJson()`，查询多条用 `select()` 或 `selects()`

## 技术栈

- **框架**: uni-app (Vue 3 Composition API + `<script setup>`)
- **云开发**: vk-unicloud-router（单 router 云函数 + service 目录路由）
- **数据库**: uniCloud 云数据库（通过 vk.baseDao 操作）
- **状态管理**: Pinia 或 Vue 3 reactive/ref
- **目标平台**: 微信小程序

## 项目结构与路由映射

vk-unicloud-router 使用**单一云函数** `router` 作为后端入口，通过 `service/` 目录下的文件路径实现自动路由映射：

```
${uniCloud目录}/
└── cloudfunctions/
    └── router/                    # 唯一的后端入口云函数
        ├── index.js               # 入口文件（勿修改）
        ├── config.js              # 全局配置
        ├── package.json
        ├── service/               # 业务逻辑（URL 映射到此目录）
        │   ├── client/            # 客户端业务模块
        │   │   ├── moduleName/    # 业务模块名
        │   │   │   ├── pub/       # 公开接口（所有人可访问）
        │   │   │   │   └── getList.js
        │   │   │   └── kh/        # 用户接口（需登录）
        │   │   │       └── add.js
        │   │   └── ...
        │   ├── user/              # 用户中心模块
        │   │   ├── pub/           # 登录注册等公开接口
        │   │   ├── kh/            # 用户私有接口
        │   │   └── sys/           # 管理后台接口
        │   └── admin/             # 管理端模块
        │       └── system/
        │           └── sys/
        ├── middleware/             # 过滤器/中间件
        │   └── modules/           # 中间件模块（自动加载）
        ├── dao/                   # 数据访问层 (Dao 2.0)
        │   ├── base.js            # BaseDao 基类
        │   ├── config.js          # 表名配置
        │   └── modules/           # Dao 实现类
        └── util/                  # 工具函数
            └── pubFunction.js     # 公共函数
```

### 路由映射规则

前端 URL 直接对应 `service/` 目录下的文件路径，无需任何配置：

| 前端调用 URL | 映射到的文件 |
|---|---|
| `client/order/pub/getList` | `router/service/client/order/pub/getList.js` |
| `client/order/kh/add` | `router/service/client/order/kh/add.js` |
| `user/pub/login` | `router/service/user/pub/login.js` |
| `user/kh/getMyUserInfo` | `router/service/user/kh/getMyUserInfo.js` |
| `admin/system/sys/getList` | `router/service/admin/system/sys/getList.js` |

### 目录权限模型

| 目录 | 访问权限 | userInfo/uid 可用 | 典型场景 |
|---|---|---|---|
| `pub/` | 所有人可访问 | 否（除非传 `need_user_info:true`） | 登录、注册、公开数据查询 |
| `kh/` | 必须登录 | 是（自动注入） | 用户个人操作、订单、收藏等 |
| `sys/` | 必须登录 + 需管理员权限 | 是（自动注入） | 后台管理、系统设置 |

## 两种后端模式：云函数 vs 云对象

vk-unicloud-router 支持**云函数路由模式**和**云对象路由模式**，两者可在同一个 router 下共存。

| 特性 | 云函数模式 | 云对象模式 |
|---|---|---|
| 文件结构 | 一个文件 = 一个接口 | 一个文件 = 一组相关接口 |
| 标记 | 无需标记（默认） | 必须设置 `isCloudObject: true` |
| 函数签名 | `main: async (event) => {}` | `methodName: async function(data) {}` |
| 获取参数 | `event` 解构：`data`, `userInfo`, `util` | `data` 参数 + `this` 方法 |
| 获取 uid | `let { uid } = data;` | `let { uid } = this.getClientInfo();` |
| 获取 vk | `let { vk } = util;` | `vk = uniCloud.vk;`（顶部或 `_before` 中） |
| 权限控制 | 通过**目录名** `pub/kh/sys` | 通过**函数名前缀** `pub_`/`kh_`/`sys_` 或文件名 |
| 拦截器 | 全局中间件 `middleware/modules/` | 内置 `_before`/`_after` |
| 前端调用 | `vk.callFunction({ url })` | `vk.callFunction({ url })` 或 `uni.vk.importObject()` |

### 云函数模式

一个文件对应一个接口，通过目录名决定权限（`pub/kh/sys`）：

```
service/client/order/kh/getList.js   → 前端 URL: client/order/kh/getList
service/client/order/kh/add.js       → 前端 URL: client/order/kh/add
service/user/pub/login.js            → 前端 URL: user/pub/login
```

**标准模板：**

```js
'use strict';
module.exports = {
  /**
   * 函数描述
   * @url client/order/kh/getList 前端调用的url参数地址
   */
  main: async (event) => {
    let { data = {}, userInfo, util, filterResponse, originalParam } = event;
    let { customUtil, config, pubFun, vk, db, _, $ } = util;
    let { uid } = data;
    let res = { code: 0, msg: '' };
    // 业务逻辑开始-----------------------------------------------------------

    // 业务逻辑结束-----------------------------------------------------------
    return res;
  },
};
```

**event 对象结构：**

| 参数 | 说明 |
|---|---|
| `data` | 前端传过来的请求参数 |
| `userInfo` | 当前登录用户信息（仅 kh/sys 目录自动注入，pub 目录需传 `need_user_info:true`） |
| `uid` | 从 `data.uid` 获取，当前登录用户 ID（仅 kh/sys 目录可信） |
| `vk` | vk 实例对象，包含 `baseDao`、`pubfn` 等核心方法 |
| `db` | 数据库对象（一般不直接使用，通过 vk.baseDao 操作） |
| `_` | 等价于 `db.command`，数据库操作符 |
| `$` | 等价于 `db.command.aggregate`，聚合操作符 |
| `pubFun` | 公共函数（`router/util/pubFunction.js`） |
| `filterResponse` | 前置过滤器返回的数据 |
| `originalParam` | 原始请求参数（含 `originalParam.context.CLIENTIP` 等） |

**示例 - 分页查询：**

```js
'use strict';
module.exports = {
  /**
   * 获取订单列表
   * @url client/order/kh/getList
   */
  main: async (event) => {
    let { data = {}, util } = event;
    let { vk, _ } = util;
    let { uid, pageIndex = 1, pageSize = 10, status } = data;
    let res = { code: 0, msg: '' };

    let whereJson = { user_id: uid };
    if (status !== undefined) whereJson.status = status;

    res = await vk.baseDao.select({
      dbName: 'orders',
      pageIndex,
      pageSize,
      getCount: true,
      whereJson,
      sortArr: [{ name: '_add_time', type: 'desc' }],
    });

    return res;
  },
};
```

### 云对象模式

一个文件包含一组相关接口，通过**函数名前缀**或**文件名**决定权限：

```
service/client/order.js              → 前端 URL: client/order.getList
                                                  client/order.add
```

**标准模板：**

```js
'use strict';
let vk;
const cloudObject = {
  isCloudObject: true,       // 必须标记为 true

  _before: async function() {
    vk = uniCloud.vk;        // 初始化 vk 实例
  },

  _after: async function(options) {
    let { err, res } = options;
    if (err instanceof Error) return;
    return err ? err : res;
  },

  // kh 权限（默认，函数名无前缀 = kh）
  getList: async function(data) {
    let res = { code: 0, msg: '' };
    let { uid } = this.getClientInfo();
    let { pageIndex = 1, pageSize = 10, status } = data;

    let whereJson = { user_id: uid };
    if (status !== undefined) whereJson.status = status;

    res = await vk.baseDao.select({
      dbName: 'orders',
      pageIndex,
      pageSize,
      getCount: true,
      whereJson,
      sortArr: [{ name: '_add_time', type: 'desc' }],
    });

    return res;
  },

  // pub 权限（pub_ 前缀 = 所有人可访问）
  pub_getPublicList: async function(data) {
    let res = { code: 0, msg: '' };
    // ...
    return res;
  },

  // sys 权限（sys_ 前缀 = 需管理员权限）
  sys_deleteAll: async function(data) {
    let res = { code: 0, msg: '' };
    // ...
    return res;
  },
};

module.exports = cloudObject;
```

**云对象 this 内置方法：**

| 方法 | 说明 |
|---|---|
| `this.getClientInfo()` | 返回 `{ uid, os, appId, clientIP, platform, deviceId, filterResponse, originalParam }` |
| `this.getUserInfo()` | 获取当前登录用户信息（异步，同一请求内有缓存） |
| `this.getUtil()` | 返回 `{ customUtil, uniID, config, pubFun, db, _, $ }` |
| `this.getMethodName()` | 获取当前调用的方法名（在 `_before` 中常用） |
| `this.getParams()` | 获取当前请求参数 |
| `this.getUniIdToken()` | 获取用户 token |

**云对象权限优先级规则：** 函数名前缀（权重 3）> 文件名前缀（权重 2）> 所在目录名（权重 1）。例如 `pub.js` 文件中的 `kh_getList` 方法属于 kh 权限。

> 更多云对象详细用法（`_before`/`_after` 拦截器、this 完整 API、权限体系、跨模式调用等），参见 [cloud-object-patterns.md](references/cloud-object-patterns.md)

## vk.baseDao 核心方法速查

所有数据库操作通过 `vk.baseDao` 完成。框架自动为新增记录添加 `_add_time`（时间戳）和 `_add_time_str`（字符串时间）。

| 方法 | 说明 | 返回值 |
|---|---|---|
| `add({ dbName, dataJson })` | 新增单条 | 新记录 `_id` |
| `adds({ dbName, dataJson })` | 批量新增（dataJson 为数组） | `_id` 数组 |
| `deleteById({ dbName, id })` | 按 ID 删除 | 删除条数 |
| `del({ dbName, whereJson })` | 按条件批量删除 | 删除条数 |
| `updateById({ dbName, id, dataJson })` | 按 ID 修改 | 受影响行数 |
| `update({ dbName, whereJson, dataJson })` | 按条件批量修改 | 受影响行数 |
| `updateAndReturn({ dbName, whereJson, dataJson })` | 原子操作：修改并返回修改后数据 | 修改后的数据对象 |
| `findById({ dbName, id, fieldJson })` | 按 ID 查单条 | 数据对象 |
| `findByWhereJson({ dbName, whereJson, fieldJson })` | 按条件查单条 | 数据对象或 null |
| `select({ dbName, pageIndex, pageSize, ... })` | 分页查询（单表） | `{ rows, total, hasMore }` |
| `selects({ dbName, foreignDB, ... })` | 连表分页查询 | `{ rows, total, hasMore }` |
| `count({ dbName, whereJson })` | 统计记录数 | 数值 |
| `sum/max/min/avg({ dbName, fieldName, whereJson })` | 聚合统计 | 数值 |

> 详细的 API 文档、参数说明和高级用法（连表查询 foreignDB、分组 groupJson、事务等），参见 [basedao-api.md](references/basedao-api.md)

### 常用数据库操作符（`_` = db.command）

```js
// 比较
_.gt(18)          // 大于
_.gte(18)         // 大于等于
_.lt(100)         // 小于
_.lte(100)        // 小于等于
_.neq(0)          // 不等于
_.in([1, 2, 3])   // 包含其中任一
_.nin([1, 2, 3])  // 都不包含
_.exists(true)    // 字段存在

// 逻辑
_.and([条件1, 条件2])  // 且
_.or([条件1, 条件2])   // 或

// 更新
_.inc(1)          // 自增
_.inc(-1)         // 自减
_.remove()        // 删除字段

// 模糊查询用 RegExp
whereJson: { name: new RegExp('关键词') }
```

## 前端调用

两种模式都可以用 `vk.callFunction` 调用，云对象模式还额外支持 `uni.vk.importObject`。

### vk.callFunction（两种模式通用）

```js
// 调用云函数模式（路径对应文件）
vk.callFunction({
  url: 'client/order/kh/getList',
  title: '加载中...',
  data: { pageIndex: 1, pageSize: 10 },
  success: (data) => { console.log(data.rows); },
});

// 调用云对象模式（文件名.方法名）
vk.callFunction({
  url: 'client/order.getList',
  title: '加载中...',
  data: { pageIndex: 1, pageSize: 10 },
  success: (data) => { console.log(data.rows); },
});

// async/await 形式（两种模式同理）
let data = await vk.callFunction({
  url: 'client/order/kh/getList',
  data: { pageIndex: 1, pageSize: 10 },
});
```

### uni.vk.importObject（仅云对象模式）

云对象额外支持导入对象后直接调方法，类似面向对象调用：

```js
// 第1步：导入云对象
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

**importObject 高级选项：**

```js
// 开启简易传参（data 直接作为方法参数，省略 data 包裹）
const orderObj = uni.vk.importObject('client/order', { easy: true });
let data = await orderObj.getList({ pageIndex: 1, pageSize: 10 });

// 指定默认 loading 提示
const orderObj = uni.vk.importObject('client/order', { title: '请求中' });
```

> 更多前端调用模式（API 封装、Promise 形式、need_user_info 用法），参见 [frontend-patterns.md](references/frontend-patterns.md)

## 过滤器（中间件）

过滤器在云函数执行前后进行统一拦截，文件放在 `router/middleware/modules/` 下，所有 `.js` 文件自动加载。

框架内置三个过滤器：
- `pub` (index: 100) - 匹配 `/pub/`，放行所有请求
- `kh` (index: 200) - 匹配 `/kh/`，检测 token 有效性，注入 userInfo/uid
- `sys` (index: 300) - 匹配 `/sys/`，检测登录 + 管理员权限

### 自定义过滤器示例

```js
// middleware/modules/shopFilter.js
module.exports = [
  {
    id: 'shopManage',
    regExp: '^client/shop/manage',    // 正则匹配 URL（注意：不要用 * 通配符，用标准正则语法）
    description: '店铺管理接口权限检测',
    index: 250,                        // 执行顺序（必须 > 200，在 kh 过滤器之后）
    mode: 'onActionExecuting',         // 执行前拦截
    enable: true,
    main: async function(event) {
      let { util, filterResponse } = event;
      let { vk } = util;
      let { uid, userInfo = {} } = filterResponse;

      // 检查权限
      let shop = await vk.baseDao.findByWhereJson({
        dbName: 'shops',
        whereJson: { owner_id: uid },
      });
      if (!shop) {
        return { code: -1, msg: '无店铺管理权限' };
      }

      // 将店铺信息注入 filterResponse，后续云函数可通过 event.filterResponse.shop 获取
      filterResponse.shop = shop;
      return { code: 0, msg: 'ok' };
    },
  },
];
```

> 过滤器详细配置（mode 类型、regExp 写法、returnMode 等），参见 [cloud-function-patterns.md](references/cloud-function-patterns.md)

## Dao 2.0 数据访问层

Dao 2.0 通过类继承封装表操作，避免每次传 `dbName`：

```js
// dao/modules/orderDao.js
const { BaseDao, Tables } = require('../base.js');

class OrderDao extends BaseDao {
  constructor(obj) {
    super(obj);
    this.tableName = Tables.order;  // 在 dao/config.js 中配置表名
  }

  // 自定义方法
  async getByOrderNo(orderNo) {
    return await this.findByWhereJson({
      whereJson: { order_no: orderNo },
    });
  }
}

module.exports = OrderDao;
```

云函数中调用：

```js
// 简易调用
let order = await vk.daoCenter.orderDao.findById(orderId);

// 完整调用（支持事务）
let order = await vk.daoCenter.orderDao.findById({
  db: transaction,
  id: orderId,
  fieldJson: { order_no: true, amount: true },
});
```

## 标准返回格式

```js
// 成功
return { code: 0, msg: 'success', data: result };

// 失败
return { code: -1, msg: '错误信息' };

// 分页查询（select/selects 自动返回此格式）
return { code: 0, msg: '', rows: [], total: 0, hasMore: false };
```

## 参考文档

当需要查阅更详细的 API 文档时，阅读以下参考文件：

- **[basedao-api.md](references/basedao-api.md)** - vk.baseDao 完整 API 参考（所有方法参数、连表查询 foreignDB、分组 groupJson、事务操作、whereJson 条件语法、排序规则）
- **[cloud-function-patterns.md](references/cloud-function-patterns.md)** - 云函数路由模式（完整 event 对象、过滤器详细配置、Dao 2.0 完整指南、公共工具函数）
- **[cloud-object-patterns.md](references/cloud-object-patterns.md)** - 云对象路由模式（标准模板、this 内置 API、权限体系、_before/_after 拦截器、跨模式调用、常见问题）
- **[frontend-patterns.md](references/frontend-patterns.md)** - 前端调用模式（vk.callFunction 完整参数、uni.vk.importObject 云对象调用、API 封装规范）
- **官方文档目录** - 更多细节可查阅 `D:\Dev\Doc\vk-unicloud-docs\docs` 下的原始文档
