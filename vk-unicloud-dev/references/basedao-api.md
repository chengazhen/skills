# vk.baseDao 完整 API 参考

本文档是 vk.baseDao 的完整 API 参考。所有数据库操作都应通过此 API 完成，禁止使用原生 `db.collection()` 方法。

## 目录

- [新增操作](#新增操作)
- [删除操作](#删除操作)
- [修改操作](#修改操作)
- [查询操作 - 单条](#查询操作---单条)
- [查询操作 - 分页](#查询操作---分页)
- [查询操作 - 统计](#查询操作---统计)
- [连表查询 foreignDB](#连表查询-foreigndb)
- [分组查询 groupJson](#分组查询-groupjson)
- [事务操作](#事务操作)
- [whereJson 条件语法](#wherejson-条件语法)
- [fieldJson 字段显示规则](#fieldjson-字段显示规则)
- [sortArr 排序规则](#sortarr-排序规则)
- [addFields 自定义字段](#addfields-自定义字段)

---

## 新增操作

### vk.baseDao.add - 新增单条

```js
let id = await vk.baseDao.add({
  dbName: 'orders',           // 表名 [必填]
  dataJson: {                 // 新增数据 [必填]
    user_id: uid,
    order_no: 'ORD20240101',
    amount: 100,
    status: 0,
  },
  cancelAddTime: false,       // 取消自动生成 _add_time [可选]
  cancelAddTimeStr: false,    // 取消自动生成 _add_time_str [可选]
  db: transaction,            // 事务实例 [可选]
});
// 返回值: 新记录的 _id 字符串，失败返回 null
// 自动添加字段: _add_time (时间戳) 和 _add_time_str (字符串时间)
```

### vk.baseDao.adds - 批量新增

```js
let ids = await vk.baseDao.adds({
  dbName: 'orders',           // 表名 [必填]
  dataJson: [                 // 数据数组 [必填]
    { user_id: 'u1', amount: 100 },
    { user_id: 'u2', amount: 200 },
  ],
  cancelAddTime: false,       // 取消自动生成 _add_time [可选]
  needReturnIds: true,        // 是否返回 ids 数组 [可选，大数据量设 false 省内存]
});
// 返回值: _id 数组
```

---

## 删除操作

### vk.baseDao.deleteById - 按 ID 删除

```js
let num = await vk.baseDao.deleteById({
  dbName: 'orders',
  id: 'xxx',
  db: transaction,            // 事务实例 [可选]
});
// 返回值: 被删除的记录条数
```

### vk.baseDao.del - 按条件批量删除

```js
let num = await vk.baseDao.del({
  dbName: 'orders',
  whereJson: { status: -1 },  // where 条件 [必填]
});
// 返回值: 被删除的记录条数
// 删除全表: whereJson: { _id: _.exists(true) }
```

---

## 修改操作

### vk.baseDao.updateById - 按 ID 修改

```js
let num = await vk.baseDao.updateById({
  dbName: 'orders',
  id: 'xxx',
  dataJson: {
    status: 1,
    pay_time: Date.now(),
  },
  getUpdateData: false,       // 是否返回修改后的数据 [可选]
  db: transaction,            // 事务实例 [可选]
});
// 返回值: 受影响行数（getUpdateData=true 时返回修改后的数据对象）
```

### vk.baseDao.update - 按条件批量修改

```js
let num = await vk.baseDao.update({
  dbName: 'orders',
  whereJson: { status: 0 },   // where 条件 [必填]
  dataJson: { status: -1 },   // 要修改的数据 [必填]
});
// 返回值: 受影响行数
// 注意: 先查后改，并发不安全。需并发安全请用 updateAndReturn
```

### vk.baseDao.updateAndReturn - 原子操作：修改并返回

```js
let updatedData = await vk.baseDao.updateAndReturn({
  dbName: 'counters',
  whereJson: { name: 'order_seq' },  // 只改满足条件的第一条
  dataJson: { value: _.inc(1) },
  db: transaction,
});
// 返回值: 修改后的完整数据对象
// 特点: 原子操作，并发安全，只计一次写操作
// 用途: 自增序列号、阅读计数、余额增减等需要并发安全的场景
```

### vk.baseDao.setById - 有则替换，无则新增（原子操作）

```js
let result = await vk.baseDao.setById({
  dbName: 'settings',
  dataJson: {
    _id: 'global_config',     // 必须包含 _id [必填]
    key1: 'value1',
    key2: 'value2',
  },
});
// 返回值: { type: 'add' | 'update', id: '_id值' }
// 注意: 存在则完全替换（未包含在 dataJson 中的字段会被删除）
```

---

## 查询操作 - 单条

### vk.baseDao.findById - 按 ID 查询

```js
let data = await vk.baseDao.findById({
  dbName: 'orders',
  id: 'xxx',
  fieldJson: { order_no: true, amount: true },  // 字段过滤 [可选]
  db: transaction,
});
// 返回值: 数据对象
```

### vk.baseDao.findByWhereJson - 按条件查询单条

```js
let data = await vk.baseDao.findByWhereJson({
  dbName: 'orders',
  whereJson: { order_no: 'ORD20240101' },
  fieldJson: { _id: true, order_no: true, amount: true },
  sortArr: [{ name: '_add_time', type: 'desc' }],  // 排序后取第一条
});
// 返回值: 满足条件的第一条记录，无匹配返回 null
```

---

## 查询操作 - 分页

### vk.baseDao.select - 单表分页查询

```js
let res = await vk.baseDao.select({
  dbName: 'orders',
  pageIndex: 1,               // 页码，默认 1
  pageSize: 10,               // 每页条数，默认 10，最大 1000（可设 -1 查全部）
  getCount: true,             // 是否返回总数，默认 false
  getMain: false,             // 是否只返回 rows 数组，默认 false
  getOne: false,              // 是否只返回第一条（rows 变成对象），默认 false
  hasMore: false,             // 是否精确计算 hasMore，默认 false
  whereJson: {
    user_id: uid,
    status: _.in([0, 1]),
  },
  fieldJson: { _id: true, order_no: true, amount: true },
  sortArr: [{ name: '_add_time', type: 'desc' }],
});
// 返回值:
// {
//   code: 0, msg: '',
//   rows: [...],             // 数据列表
//   total: 100,              // 总记录数（getCount=true 时）
//   hasMore: true,           // 是否还有下一页
//   pagination: { pageIndex: 1, pageSize: 10 },
// }
```

**pageSize 特殊值：**
- 默认最大 1000（云厂商限制）
- 可设置 > 1000 突破限制（仅按 `_id` 排序时启用游标查询，性能最优）
- 设置 `-1` 查询全部数据

### vk.baseDao.selects - 连表分页查询

```js
let res = await vk.baseDao.selects({
  dbName: 'orders',
  pageIndex: 1,
  pageSize: 10,
  getCount: true,
  whereJson: { status: 1 },
  fieldJson: { _id: true, order_no: true, amount: true, user_id: true },
  sortArr: [{ name: '_add_time', type: 'desc' }],
  foreignDB: [                // 连表规则
    {
      dbName: 'uni-id-users',
      localKey: 'user_id',
      foreignKey: '_id',
      as: 'userInfo',
      limit: 1,               // 1=返回对象，>1=返回数组
      fieldJson: { nickname: true, avatar: true },
    },
  ],
  // 以下参数性能较差，尽量避免使用
  lastWhereJson: {},          // 连表后的 where 条件
  lastSortArr: [],            // 连表后的排序
  groupJson: {},              // 分组规则
  addFields: {},              // 自定义字段
});
```

### vk.baseDao.getTableData - 表格数据查询

参数同 `selects`，默认 `getCount: true`，主要用于管理后台万能表格。

---

## 查询操作 - 统计

### vk.baseDao.count

```js
let total = await vk.baseDao.count({
  dbName: 'orders',
  whereJson: { status: 1 },
});
```

### vk.baseDao.sum / max / min / avg

```js
let totalAmount = await vk.baseDao.sum({
  dbName: 'orders',
  fieldName: 'amount',
  whereJson: { status: 1 },
});
```

### vk.baseDao.sample - 随机取 N 条

```js
let randomItems = await vk.baseDao.sample({
  dbName: 'products',
  size: 5,
  whereJson: { status: 1 },
});
```

---

## 连表查询 foreignDB

foreignDB 是一个数组，每个元素定义一个连表关系：

```js
foreignDB: [
  {
    dbName: 'uni-id-users',     // 副表名 [必填]
    localKey: 'user_id',        // 主表外键字段 [必填]
    foreignKey: '_id',          // 副表外键字段 [必填]
    as: 'userInfo',             // 别名 [必填]
    limit: 1,                   // 1=返回对象，>1=返回数组
    fieldJson: {},              // 副表字段过滤
    whereJson: {},              // 副表条件过滤
    sortArr: [],                // 副表排序
    localKeyType: 'string',     // 主表外键类型，'array' 表示主表字段是数组
    foreignKeyType: 'string',   // 副表外键类型，'array' 表示副表字段是数组
    foreignDB: [],              // 副表的副表（嵌套连表，最多 15 层）
  },
]
```

### 常见连表场景

**一对一关系（订单→用户）：**
```js
foreignDB: [{
  dbName: 'uni-id-users', localKey: 'user_id', foreignKey: '_id',
  as: 'userInfo', limit: 1,
  fieldJson: { nickname: true, avatar: true },
}]
```

**一对多关系（用户→订单列表）：**
```js
foreignDB: [{
  dbName: 'orders', localKey: '_id', foreignKey: 'user_id',
  as: 'orderList', limit: 100,
  sortArr: [{ name: '_add_time', type: 'desc' }],
}]
```

**主表外键是数组（用户的多个角色）：**
```js
foreignDB: [{
  dbName: 'roles', localKey: 'role_ids', localKeyType: 'array',
  foreignKey: '_id', as: 'roleList', limit: 1000,
}]
```

**多层嵌套连表：**
```js
foreignDB: [{
  dbName: 'uni-id-users', localKey: 'user_id', foreignKey: '_id',
  as: 'userInfo', limit: 1,
  foreignDB: [{
    dbName: 'departments', localKey: 'dept_id', foreignKey: '_id',
    as: 'deptInfo', limit: 1,
  }],
}]
```

---

## 分组查询 groupJson

配合 `selects` 使用聚合操作符 `$`：

```js
let res = await vk.baseDao.selects({
  dbName: 'orders',
  groupJson: {
    _id: '$user_id',                    // 按 user_id 分组（_id 固定，必填）
    user_id: $.first('$user_id'),       // 保留 user_id 字段
    total_amount: $.sum('$amount'),     // 求和
    order_count: $.sum(1),              // 计数
  },
  sortArr: [{ name: 'total_amount', type: 'desc' }],
  foreignDB: [{
    dbName: 'uni-id-users', localKey: 'user_id', foreignKey: '_id',
    as: 'userInfo', limit: 1,
  }],
});
```

### 按日期分组统计

```js
groupJson: {
  _id: $.dateToString({
    date: $.add([new Date(0), '$_add_time']),
    format: '%Y-%m',            // 格式：%Y年 %m月 %d日 %H时 %M分 %S秒
    timezone: '+08:00',
  }),
  count: $.sum(1),
}
```

### 多字段分组

```js
groupJson: {
  _id: { status: '$status', type: '$type' },
  status: $.first('$status'),
  type: $.first('$type'),
  count: $.sum(1),
}
```

---

## 事务操作

```js
// 1. 开启事务
const transaction = await vk.baseDao.startTransaction();

try {
  // 2. 所有操作加 db: transaction 参数
  let user = await vk.baseDao.findById({
    db: transaction,
    dbName: 'uni-id-users',
    id: uid,
  });

  if (user.balance < 100) {
    return await vk.baseDao.rollbackTransaction({
      db: transaction,
      err: { code: -1, msg: '余额不足' },
    });
  }

  await vk.baseDao.updateById({
    db: transaction,
    dbName: 'uni-id-users',
    id: uid,
    dataJson: { balance: _.inc(-100) },
  });

  await vk.baseDao.add({
    db: transaction,
    dbName: 'orders',
    dataJson: { user_id: uid, amount: 100, status: 1 },
  });

  // 3. 提交事务
  await transaction.commit();
  return { code: 0, msg: '操作成功' };
} catch (err) {
  // 4. 异常回滚
  return await vk.baseDao.rollbackTransaction({ db: transaction, err });
}
```

**事务注意事项：**
- 开启到提交不能超过 10 秒
- 仅 `add`、`findById`、`updateById`、`deleteById`、`updateAndReturn`、`setById` 支持事务
- 事务中 add 不执行自动建表，表必须预先存在

---

## whereJson 条件语法

### 基础比较

```js
whereJson: {
  age: _.gt(18),              // 大于
  score: _.gte(80),           // 大于等于
  count: _.lt(100),           // 小于
  num: _.lte(50),             // 小于等于
  status: _.neq(0),           // 不等于
  _id: _.in(['id1', 'id2']), // 包含其中任一
  type: _.nin([3, 4]),        // 都不包含
  phone: _.exists(true),      // 字段存在
}
```

### 流式语法（同字段多条件）

```js
whereJson: {
  num: _.gte(0).lte(100),                   // 0 <= num <= 100
  age: _.lte(0).or(_.gte(100)),             // age <= 0 或 age >= 100
}
```

### 跨字段逻辑

```js
// OR 逻辑
whereJson: _.or([
  { status: 1 },
  { status: 2 },
])

// AND 嵌套 OR
whereJson: _.and([
  { user_id: uid },
  _.or([
    { status: 1 },
    { status: 2 },
  ]),
])
```

### 模糊查询

```js
// 单字段
whereJson: { name: new RegExp('关键词') }

// 多字段模糊搜索
let reg = new RegExp(searchValue);
whereJson: _.or([
  { username: reg },
  { nickname: reg },
  { mobile: reg },
])
```

### 日期范围

```js
let { todayStart, todayEnd } = vk.pubfn.getCommonTime();
whereJson: {
  _add_time: _.gte(todayStart).lte(todayEnd),
}
```

---

## fieldJson 字段显示规则

```js
// 方式1: 只显示指定字段
fieldJson: { _id: true, name: true, age: true }

// 方式2: 隐藏指定字段
fieldJson: { password: false, token: false }

// 隐藏 _id（默认显示）
fieldJson: { _id: false }
```

---

## sortArr 排序规则

```js
// 单字段排序
sortArr: [{ name: '_add_time', type: 'desc' }]

// 多字段排序
sortArr: [
  { name: 'priority', type: 'desc' },
  { name: '_add_time', type: 'desc' },
]

// 升序
sortArr: [{ name: 'sort', type: 'asc' }]
```

---

## addFields 自定义字段

将副表字段提升到主表顶层，配合 `selects` 使用：

```js
let res = await vk.baseDao.selects({
  dbName: 'orders',
  foreignDB: [{
    dbName: 'uni-id-users', localKey: 'user_id', foreignKey: '_id',
    as: 'userInfo', limit: 1,
  }],
  addFields: {
    user_name: '$userInfo.nickname',
    user_avatar: '$userInfo.avatar',
  },
  fieldJson: { userInfo: false },  // 隐藏原副表字段
});
```
