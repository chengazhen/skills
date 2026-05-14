# 云函数高级模式

本文档涵盖 vk-unicloud-router 云函数路由模式的高级用法，包括完整 event 对象、过滤器详细配置、Dao 2.0 完整指南和公共工具函数。

> 云对象路由模式请参见 [cloud-object-patterns.md](cloud-object-patterns.md)

## 目录

- [完整 event 对象](#完整-event-对象)
- [云函数模板详解](#云函数模板详解)
- [过滤器详细配置](#过滤器详细配置)
- [Dao 2.0 完整指南](#dao-20-完整指南)
- [公共工具函数](#公共工具函数)

---

## 完整 event 对象

```js
let { data = {}, userInfo, util, filterResponse, originalParam } = event;
let { customUtil, uniID, config, pubFun, vk, db, _, $ } = util;
let { uid } = data;
```

### 参数详解

| 参数 | 说明 | 备注 |
|---|---|---|
| `data` | 前端传过来的请求参数 | 总是可用 |
| `userInfo` | 当前登录用户信息 | kh 目录自动注入；pub 目录需前端传 `need_user_info:true` |
| `uid` | 从 `data.uid` 获取当前登录用户 ID | kh 目录可信（从 token 解密）；pub 目录不解析 token |
| `util` | 工具包对象 | 总是可用 |
| `vk` | vk 实例（含 baseDao、pubfn 等） | 核心对象 |
| `db` | 数据库对象 | 一般通过 vk.baseDao 操作，不直接使用 |
| `_` | `db.command` 操作符 | 用于 `_.gt()`、`_.inc()` 等 |
| `$` | `db.command.aggregate` 聚合操作符 | 用于 `$.sum()`、`$.first()` 等 |
| `pubFun` | 公共函数 | 文件位于 `router/util/pubFunction.js` |
| `customUtil` | 自定义工具包 | 开发者自定义 |
| `uniID` | uni-id 实例对象 | 用于用户体系操作 |
| `config` | 全局配置 | 来自 `router/config.js` |
| `filterResponse` | 过滤器返回的数据 | 可获取过滤器注入的额外信息 |
| `originalParam` | 原始请求参数 | 含 event 和 context |

### originalParam.context 常用属性

```js
let clientIP = originalParam.context.CLIENTIP;     // 客户端 IP
let platform = originalParam.context.PLATFORM;     // mp-weixin、app-plus 等
let os = originalParam.context.OS;                 // android、ios
let appid = originalParam.context.APPID;           // manifest.json 中的 appid
let deviceId = originalParam.context.DEVICEID;     // 设备标识
let spaceInfo = originalParam.context.SPACEINFO;   // 云空间信息
```

### need_user_info 机制

**kh 目录：** 默认自动获取 userInfo。如果云函数不需要用户信息，前端传 `need_user_info: false` 可减少一次数据库查询（快约 100ms），但 uid 仍然可用（从 token 解密）。

```js
// 前端调用（kh 目录，不需要 userInfo 时）
vk.callFunction({
  url: 'client/order/kh/getList',
  data: {
    need_user_info: false,    // 注意：放在 data 内部，不是与 data 同级
    pageIndex: 1,
  },
});
```

**pub 目录：** 默认不获取 userInfo 也不获取 uid。如需获取，前端传 `need_user_info: true`：

```js
// 前端调用（pub 目录，需要 userInfo 时）
vk.callFunction({
  url: 'client/product/pub/getDetail',
  data: {
    need_user_info: true,     // 放在 data 内部
    product_id: 'xxx',
  },
});
```

---

## 云函数模板详解

### 完整模板（推荐）

```js
'use strict';
module.exports = {
  /**
   * 函数描述
   * @url client/moduleName/kh/functionName 前端调用的url参数地址
   * @description 详细描述
   * @param {Object} data 请求参数
   * @param {String} uniIdToken 用户token
   * @param {String} userInfo 当前登录用户信息（仅kh目录有此值）
   * @param {Object} util 公共工具包
   * @param {Object} filterResponse 过滤器返回的数据
   * @param {Object} originalParam 原始请求参数
   */
  main: async (event) => {
    let { data = {}, userInfo, util, filterResponse, originalParam } = event;
    let { customUtil, uniID, config, pubFun, vk, db, _, $ } = util;
    let { uid } = data;
    let res = { code: 0, msg: '' };
    // 业务逻辑开始-----------------------------------------------------------

    // 业务逻辑结束-----------------------------------------------------------
    return res;
  },
};
```

### 简易模板

```js
'use strict';
module.exports = {
  /**
   * 函数描述
   * @url client/moduleName/pub/functionName
   */
  main: async (event) => {
    let { data = {}, userInfo, util, originalParam } = event;
    let { customUtil, config, pubFun, vk, db, _, $ } = util;
    let res = { code: 0, msg: '' };
    // 业务逻辑开始-----------------------------------------------------------

    // 业务逻辑结束-----------------------------------------------------------
    return res;
  },
};
```

---

## 过滤器详细配置

过滤器文件放在 `router/middleware/modules/` 下，所有 `.js` 文件自动加载。每个文件导出一个数组：

### 配置结构

```js
module.exports = [
  {
    id: 'myFilter',                     // 全局唯一 ID（相同 ID 会覆盖）
    regExp: '^client/shop/manage',      // 正则匹配 URL
    description: '过滤器描述',
    index: 250,                         // 执行顺序（越小越先执行）
    mode: 'onActionExecuting',          // 执行模式
    enable: true,                       // 是否启用
    main: async function(event) {
      let { util, data, filterResponse, url } = event;
      let { vk, db, _ } = util;

      // 拦截: 返回 code 非 0
      // return { code: -1, msg: '拦截原因' };

      // 放行: 返回 code 0
      return { code: 0, msg: 'ok' };
    },
  },
];
```

### 参数说明

| 参数 | 类型 | 说明 |
|---|---|---|
| `id` | String | 全局唯一 ID |
| `regExp` | String/Array | 正则匹配规则。字符串或字符串数组 |
| `description` | String | 描述 |
| `index` | Number | 执行顺序（越小越先执行） |
| `mode` | String | 执行模式（见下表） |
| `enable` | Boolean | 是否启用 |
| `main` | Function | 执行函数 |
| `returnMode` | Number | 返回值模式：0=Object.assign 合并，1=完全替换 |

### 执行模式

| mode | 说明 | 典型场景 |
|---|---|---|
| `onActionExecuting` | action 执行前（最常用） | 权限检查、参数验证 |
| `onActionExecuted` | action 执行后 | 结果转换、日志记录 |
| `onActionIntercepted` | action 被其他中间件拦截后 | 资源清理 |
| `onActionError` | action 异常时 | 异常处理、日志 |

### regExp 写法

```js
// 匹配所有
regExp: '(.*)'

// 匹配 kh 目录
regExp: '/kh/'

// 匹配指定模块
regExp: '^client/shop/manage'

// 精确匹配
regExp: '^client/order/kh/getList$'

// 多个匹配
regExp: ['^client/order/kh/add$', '^client/order/kh/update$']

// 匹配范围
regExp: '^client/(order|product)/(kh|sys)/(.*)'
```

**重要：** regExp 使用标准正则语法，`*` 是量词（匹配前一字符零次或多次），不是通配符。匹配任意字符用 `(.*)`。

### 框架内置过滤器

| ID | regExp | index | 说明 |
|---|---|---|---|
| pub | `/pub/` | 100 | 放行所有请求 |
| kh | `/kh/` | 200 | 检测 token，注入 userInfo/uid |
| sys | `/sys/` | 300 | 检测登录 + 管理员权限 |

自定义过滤器的 `index` 必须根据需要设置：
- 在 kh 之后执行：index > 200
- 在 sys 之后执行：index > 300

### 过滤器注入数据

过滤器可以向 `filterResponse` 注入数据，后续云函数通过 `event.filterResponse` 获取：

```js
// 过滤器中
filterResponse.shop = shopData;

// 云函数中
let { filterResponse } = event;
let shop = filterResponse.shop;
```

---

## Dao 2.0 完整指南

Dao 2.0 通过类继承封装表操作，推荐在正式项目中使用。

### 表名配置

```js
// dao/config.js
module.exports = {
  Tables: {
    user: 'uni-id-users',
    order: 'orders',
    product: 'products',
    shop: 'shops',
  },
};
```

### 编写 Dao 类

```js
// dao/modules/orderDao.js
const { BaseDao, Tables } = require('../base.js');

class OrderDao extends BaseDao {
  constructor(obj) {
    super(obj);
    this.tableName = Tables.order;
  }

  // 自定义方法（如无特殊需求，可不写，基类方法已足够）
  async getByOrderNo(orderNo) {
    return await this.findByWhereJson({
      whereJson: { order_no: orderNo },
    });
  }

  async getUserOrders(userId, pageIndex = 1, pageSize = 10) {
    return await this.select({
      pageIndex,
      pageSize,
      whereJson: { user_id: userId },
      sortArr: [{ name: '_add_time', type: 'desc' }],
      getCount: true,
    });
  }
}

module.exports = OrderDao;
```

**文件命名规则：** 文件名必须以 `Dao.js` 结尾（如 `orderDao.js`、`userDao.js`）。

### BaseDao 继承的方法

所有 `vk.baseDao` 的方法都可在 Dao 类中调用（不需要传 `dbName`）：

```js
// BaseDao 提供的方法（等同 vk.baseDao，但省去 dbName 参数）
this.add(dataJson)
this.adds(dataArray)
this.findById(id, fieldJson)
this.findByWhereJson(whereJson, fieldJson)
this.select(params)
this.selects(params)
this.count(whereJson)
this.updateById(id, dataJson)
this.update(whereJson, dataJson)
this.updateAndReturn(whereJson, dataJson)
this.deleteById(id)
this.del(whereJson)
this.setById(dataJson)
this.sum({ fieldName, whereJson })
this.max({ fieldName, whereJson })
this.min({ fieldName, whereJson })
this.avg({ fieldName, whereJson })
```

### 调用方式

```js
// 在云函数中调用
// 简易调用
let order = await vk.daoCenter.orderDao.findById(orderId);

// 完整调用（支持事务）
let order = await vk.daoCenter.orderDao.findById({
  db: transaction,
  id: orderId,
  fieldJson: { order_no: true, amount: true },
});

// 自定义方法
let order = await vk.daoCenter.orderDao.getByOrderNo('ORD20240101');

// 分页查询
let result = await vk.daoCenter.orderDao.getUserOrders(uid, 1, 20);
```

---

## 公共工具函数

### vk.pubfn 常用方法

```js
// 获取常用时间
let {
  todayStart,    // 今天开始时间戳
  todayEnd,      // 今天结束时间戳
  weekStart,     // 本周开始
  weekEnd,       // 本周结束
  monthStart,    // 本月开始
  monthEnd,      // 本月结束
  yearStart,     // 本年开始
  yearEnd,       // 本年结束
} = vk.pubfn.getCommonTime();

// 生成唯一 ID
let id = vk.pubfn.createOrderNo(); // 订单号

// 日期格式化
let str = vk.pubfn.timeFormat(Date.now(), 'yyyy-MM-dd hh:mm:ss');
```

### pubFunction.js 自定义公共函数

```js
// router/util/pubFunction.js
module.exports = {
  // 自定义函数
  async myFunction(vk, params) {
    // ...
  },
};
```

在云函数中使用：`pubFun.myFunction(vk, params)`
