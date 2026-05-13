# 第八章 测试

##### 测试场景概述

进入迭代循环后，在循环内部，我们：

*   向 `/products` 端点发出两次 GET 请求，以模拟顾客浏览商品。
*   第三次向 `/products` 端点发出 GET 请求，但这次捕获一个产品数组。
*   从产品数组中随机选择一个产品。
*   向 `/orders` 端点发出一个 POST 请求，请求体是一个表示顾客订单和产品的 JSON。
*   向 `/customers/{id}/orders` 端点发出两次 GET 请求，以模拟顾客查看其订单状态。
*   如果在所有测试迭代运行完毕后，错误率大于 1% 或 95% 百分位的请求时长大于 200 毫秒，则测试失败。

##### 测试脚本

```javascript
import http from 'k6/http';
import { check, group } from 'k6';
import { randomString, randomItem } from 'https://jslib.k6.io/k6-utils/1.1.0/index.js';

const BASE_URL = 'http://localhost:3000';

export const options = {
  thresholds: {
    http_req_failed: ['rate<0.01'],
    http_req_duration: ['p(95)<200'],
  }
};

export function setup() {
  const name = randomString(16);
  const createResponse = http.post(
    `${BASE_URL}/customers`,
    JSON.stringify({
      full_name: 'Full Name ' + name,
      email: name + `@acme.com`
    }),
    { headers: {'Content-Type': 'application/json'} }
  );

  check(createResponse, { 'create customer response': (r) => r.status === 200 });
  return createResponse.json('id');
}

export default (id) => {
  group('make order', () => {
    `// 模拟顾客下单前的浏览行为。`
    http.batch([
      ['GET', `${BASE_URL}/products`, null],
      ['GET', `${BASE_URL}/products`, null],
    ]);

    `// 发出请求并捕获响应，以提取一个随机产品进行购买。`
    const productsResponse = http.get(
      `${BASE_URL}/products`,
    );
    const product = randomItem(productsResponse.json());
    const productId = product.id;
    const priceId = product.prices[0].id;

    `// 下单。`
    http.post(
      `${BASE_URL}/orders`,
      JSON.stringify({
        customer_id: id,
        products: { [productId]: priceId },
      }),
      { headers: {'Content-Type': 'application/json'} }
    );
  });

  `// 模拟顾客查看其订单状态。`
  group('check orders', () => {
    http.batch([
      ['GET', `${BASE_URL}/customers/${id}/orders`, null],
      ['GET', `${BASE_URL}/customers/${id}/orders`, null],
    ]);
  });
};
```

##### 运行模拟

现在是运行模拟的时候了。首先，让我们启动 `products.js` 脚本并让它运行十秒。由于不会有太多人进入产品页面，我将模拟只有一个用户浏览产品（每秒五个产品；我可没说他们是普通人）。

```bash
$ k6 -u 1 -d 10m --rps 5 run products.js
```

```
running (10.0s), 0/1 VUs, 51 complete and 0 interrupted iterations
default ✓ [======================================] 1 VUs 10s
✓ create product response
checks.........................: 100.00% ✓ 51 ✗ 0
data_received..................: 8.2 kB 818 B/s
data_sent......................: 15 kB 1.5 kB/s
✓ http_req_duration..............: avg=41.36ms min=24.43ms med=37.63ms max=114.66ms p(90)=67.91ms p(95)=77.02ms
{ expected_response:true }...: avg=41.36ms min=24.43ms med=37.63ms max=114.66ms p(90)=67.91ms p(95)=77.02ms
✓ http_req_failed................: 0.00% ✓ 0 ✗ 51
iterations.....................: 51 5.077393/s
vus............................: 1 min=1 max=1
vus_max........................: 1 min=1 max=1
```

在它运行的同时，我们将启动 `customers.js` 脚本并让它运行十秒。我现在只模拟两个顾客，他们会尽快完成整个场景：

```bash
$ k6 -u 2 -d 10s run customer.js
```

```
running (10.0s), 0/2 VUs, 55 complete and 0 interrupted iterations
default ✓ [======================================] 2 VUs 10s
█ setup
✓ create customer response
█ make order
█ check orders
checks.........................: 100.00% ✓ 1 ✗ 0
data_received..................: 55 MB 5.4 MB/s
data_sent......................: 45 kB 4.5 kB/s
✓ http_req_duration..............: avg=41.88ms min=27.75ms med=44.86ms max=55.69ms p(90)=50.78ms p(95)=52.32ms
{ expected_response:true }...: avg=41.88ms min=27.75ms med=44.86ms max=55.69ms p(90)=50.78ms p(95)=52.32ms
```



