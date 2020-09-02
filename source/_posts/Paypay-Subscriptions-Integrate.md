---
title: 'Paypal 实现自动订阅'
date: 2018-08-30 16:30:00
tags: 
- PHP
---


官方给出的自动续费分五步 [Intergrate Subscriptions](https://developer.paypal.com/docs/subscriptions/integrate/integrate-steps/)。实际开发中，还需要实现支付结果处理和订阅管理等：

1. 事先创建计划，并激活；
2. 用户创建订阅，跳转到paypal网站等待用户同意；
3. 用户同意后，跳转回网站，执行订阅；
4. 获取用户账单，包括每次扣款结果通知的接收或支付结果的主动查询；
5. 处理用户取消订阅等通知。



### 使用 Palpal SDK

``` php
composer require paypal/rest-api-sdk-php
```

官方有完整的 [Samples](https://paypal.github.io/PayPal-PHP-SDK/sample/#billing)；

可以通过 [Paypal Sandbox](https://www.sandbox.paypal.com/) 方便的进行调试。



### 创建订阅计划并激活

- 订阅计划（Billing Plan）等同于的产品，需要为每个商品不同价格创建不同的计划。不过可以针对不同用户在创建协议时更改；
- Payment 中创建 `TRIAL` 类型支付时，也必须存在 `REGULAR` 的支付。`TRAIL` 并不能自动判断是否为新用户等条件，所以新用户首次的优惠需要业务代码自己实现。
- 由于创建用户订阅协议时，**协议生效时间必须在当前时间24小时以后**，所以循环扣款的设置无法立刻扣款，最早也需要24小时。一般业务需要立刻进行首次扣款，可以用 `MerchantPreferences` 的 `setSetupFee` 来设置首次扣款的费用；
- Paypal SDK 会报错 `"NotifyUrl" value is NULL`，该错误为 Paypal 服务端错误，但官方未修复，解决办法见 [issue](https://github.com/paypal/PayPal-PHP-SDK/pull/1152/files)。



### 创建订阅

- 用户可以创建针对同一订阅计划的多个订阅协议（Billing Agreement），创建后跳转至 Paypal 网站等待用户同意协议；

- 因协议开始时间 `start_date` 最早为当前时间24小时之后，所以该值实际上设置的是第二次扣款时间。所以，若设置按月付款，`start_date` 需要设置成一个月以后，然后通过设置 `setSetupFee` 价格来设置首次扣款费用；

- 创建订阅后，还没有生成 `Agreement.id`，这时候需要从跳转链接中提取出 `token` 来使创建的订阅与用户同意后跳转的回来的协议信息相对应。

  ``` php
  $link = $agreement->getApprovalLink();
  parse_str(parse_url($link, PHP_URL_QUERY), $params);
  $token = $params['token'];
  ```



### 执行订阅

- 同一个订阅计划可以被同一个用户多次订阅。所以根据需要，需要在执行新协议时，手动取消该用户之前的协议；
- 实际扣款时间有延迟，每次循环扣款执行的时间，都会比`AgreementDetail.next`显示的时间晚几个小时。所以为保证连续性，可以设置提前一天扣款。



### 支付结果接收与查询

- 可以在 `My Apps -> REST API apps -> WEBHOOKS` 设置 `webhook` 通知。当每次循环扣款成功时，Paypal 都会发送 `PAYMENT.SALE.COMPLETED` 的事件通知，可以通过其中的 `billing_agreement_id` 字段与已创建的订阅相匹配，找出对应付款的协议。
- 每次 `AgreementDetail` 都会返回下次收款时间 `next` 参数。可以在超过这个时间后，通过 `Agreement::searchTransactions` 方法查询该协议的所有交易。需要注意的是，Paypal 实际的扣款时间一般都会延迟，所以需要多次重试。



### 用户订阅取消与删除等

- 取消订阅会通过 `webhook` 发送 `BILLING.SUBSCRIPTION.CANCELLED` 通知，订阅暂停会发送 `BILLING.SUBSCRIPTION.SUSPENDED` 通知
- 直接删除计划并不会自动删除基于该计划的协议，所以再删除计划前，需要手动取消所有订阅该计划的协议。



### 参考资料

<https://developer.paypal.com/docs/subscriptions/>

<https://paypal.github.io/PayPal-PHP-SDK/sample/>

<https://www.cnblogs.com/pheye/p/6603126.html>

<!--more-->