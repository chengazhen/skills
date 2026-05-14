# 前端调用模式

本文档涵盖 vk-unicloud 前端调用的完整模式，包括 vk.callFunction 和 uni.vk.importObject 两种调用方式、API 封装规范。

## 目录

- [vk.callFunction 完整参数](#vkcallfunction-完整参数)
- [vk.callFunction 调用方式](#vkcallfunction-调用方式)
- [uni.vk.importObject 云对象调用](#univkimportobject-云对象调用)
- [API 封装规范](#api-封装规范)

---

## vk.callFunction 完整参数

`vk.callFunction` 是两种模式通用的前端调用方法。

| 参数 | 类型 | 默认值 | 说明 |
|---|---|---|---|
| `name` | String | `'router'` | 云函数名（通常不需要改） |
| `url` | String | **必填** | 云函数：文件路径（如 `client/order/kh/getList`）；云对象：文件名.方法名（如 `client/order.getList`） |
| `data` | Object | `{}` | 请求参数 |
| `title` | String | - | loading 遮罩层提示语（传值即显示 loading） |
| `loading` | Boolean/Object | - | 自定义 loading 行为 |
| `needAlert` | Boolean | `true` | 请求失败时是否自动弹窗提示 |
| `timeout` | Number | - | 请求超时时间（毫秒） |
| `retryCount` | Number | `0` | 系统异常自动重试次数 |
| `encrypt` | Boolean | `false` | 是否加密通信 |
| `success` | Function | - | 成功回调 |
| `fail` | Function | - | 失败回调 |
| `complete` | Function | - | 完成回调 |

---

## vk.callFunction 调用方式

### 回调形式

```js
vk.callFunction({
  url: 'client/order/kh/getList',   // 云函数模式
  // url: 'client/order.getList',   // 云对象模式
  title: '加载中...',
  data: { pageIndex: 1, pageSize: 10 },
  success: (data) => {
    console.log(data.rows);
  },
  fail: (err) => {
    console.log(err.msg);
  },
  complete: (res) => {},
});
```

### Promise 形式

```js
vk.callFunction({
  url: 'client/order/kh/getList',
  data: { pageIndex: 1, pageSize: 10 },
})
  .then((data) => { console.log(data.rows); })
  .catch((err) => { console.log(err.msg); });
```

### async/await 形式（推荐）

```js
try {
  let data = await vk.callFunction({
    url: 'client/order/kh/getList',
    data: { pageIndex: 1, pageSize: 10 },
  });
  console.log(data.rows);
} catch (err) {
  console.log(err.msg);
}
```

---

## uni.vk.importObject 云对象调用

仅限云对象模式使用，提供面向对象的调用体验。

### 基础用法

```js
// 第1步：导入云对象（路径对应 service/ 下的 .js 文件，不含 .js 后缀）
const orderObj = uni.vk.importObject('client/order');

// 第2步：调用方法（回调形式）
orderObj.getList({
  title: '加载中...',
  data: { pageIndex: 1, pageSize: 10 },
  success: (data) => { console.log(data.rows); },
  fail: (err) => { console.log(err.msg); },
});

// 第2步：调用方法（async/await 形式）
let data = await orderObj.getList({
  title: '加载中...',
  data: { pageIndex: 1, pageSize: 10 },
});
```

### importObject 高级选项

```js
// 指定默认 title（后续调用无需每次传 title）
const orderObj = uni.vk.importObject('client/order', { title: '请求中' });

// 简易传参模式（data 直接作为方法参数，省略 data 包裹）
const orderObj = uni.vk.importObject('client/order', { easy: true });
let data = await orderObj.getList({ pageIndex: 1, pageSize: 10 });

// 设置默认请求参数（每次调用都会自动合并）
const orderObj = uni.vk.importObject('client/order', { data: { shop_id: '123' } });

// 指定其他 router 云函数名
const orderObj = uni.vk.importObject('client/order', { name: 'router2' });

// 开启加密通信
const orderObj = uni.vk.importObject('client/order', { encrypt: true });
```

---

## API 封装规范

### 云函数模式封装

```js
// api/order.js
export const orderApi = {
  getList: (params) => vk.callFunction({
    url: 'client/order/kh/getList',
    data: params,
  }),
  add: (params) => vk.callFunction({
    url: 'client/order/kh/add',
    title: '提交中...',
    data: params,
  }),
  cancel: (params) => vk.callFunction({
    url: 'client/order/kh/cancel',
    title: '取消中...',
    data: params,
  }),
};
```

### 云对象模式封装

```js
// api/order.js
const orderObj = uni.vk.importObject('client/order', { easy: true });

export const orderApi = {
  getList: (params) => orderObj.getList(params),
  add: (params) => orderObj.add({ ...params, title: '提交中...' }),
  cancel: (params) => orderObj.cancel({ ...params, title: '取消中...' }),
};
```

### 页面中使用（两种封装方式通用）

```vue
<script setup>
import { ref } from 'vue'
import { orderApi } from '@/api/order.js'

const orderList = ref([])

const fetchOrders = async () => {
  let data = await orderApi.getList({
    pageIndex: 1,
    pageSize: 20,
    status: 1,
  })
  orderList.value = data.rows
}

const createOrder = async () => {
  await orderApi.add({ product_id: 'xxx', quantity: 1 })
  uni.showToast({ title: '下单成功' })
}
</script>
```
