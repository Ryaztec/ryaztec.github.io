---
title: "API 升级不是字段改名：Kitsilano 如何保住订单与金额的业务语义"
description: "从 Amazon Orders API 迁移代码看兼容边界：DTO 适配、含税价格、折扣税拆分、受控回退，以及面向 OMS 和 D365 的契约测试。"
pubDate: 2026-10-03
tags: ["Kitsilano", "API设计", "系统集成", "测试"]
draft: true
---

升级订单 API 时，最容易看到的是字段名和 SDK 类型变化。真正危险的变化却藏在金额语义中：一个总折扣是否包含运费折扣？商品价格是否已经含税？某个字段没返回，究竟表示零，还是信息不足？

Kitsilano 对 Amazon Orders API `v2026-01-01` 的迁移，通过兼容入口和 DTO 适配器，把这些变化集中在系统边界。下游 OMS（订单管理系统）与 D365 的既有计算契约仍然被认真对待，而不是因为新接口字段相似就假设两者等价。

本文基于本地阅读快照，基准提交 `54314e8d4aba`。PHP 代码块为原代码节选，流程图概括实际调用链，金额举例均为虚构数据。测试结论来自已有测试代码的断言阅读，本文未执行源项目测试；这里分析的是应用计算契约，不是通用税务规则。

## 先画出兼容边界

迁移链路可以概括为：

```text
外部 Orders API v2026
        ↓
SPAPIService：获取订单及必要的旧版补充数据
        ↓
OrdersV20260101Adapter：规范化为既有 DTO 和折扣税上下文
        ↓
现有 OMS / D365 映射与计算
```

此外，`OrdersApiCompat` 保留一部分旧的调用参数和响应形状，并翻译到新版请求。这两层承担不同职责：兼容入口处理“怎么调用”，DTO 适配处理“返回的数据意味着什么”。

例如，旧的履约渠道 `MFN`、`AFN` 分别映射成 `MERCHANT`、`AMAZON`，状态 `Canceled` 对应新版的 `CANCELLED`。这些变化由官方迁移指南列出；不能简单假设大小写转换就足够。[Amazon Orders API 迁移指南](https://developer-docs.amazon/sp-api/docs/orders-api-migration-guide)

兼容也不意味着接受所有参数。`OrdersApiCompat` 对自身不支持的一些旧过滤条件抛出异常；`SPAPIService` 对按 Amazon order IDs 获取等路径另有旧版路由。这比悄悄忽略过滤条件更容易发现问题。外层调用者需要知道支持范围，不能把这个适配器理解成旧 API 的完整替代品。

## 同名价格，可能有不同计算口径

适配器提取新版 `ITEM`、`SHIPPING` 小计，分别放入旧 DTO 的商品价和运费价。官方指南说明，独立报告税额时，新版 `ITEM` 小计不包含该税额；没有独立税额时情况不同。因此，看见字段名 `ItemPrice` 并不能直接得出两个版本口径一致。[价格字段映射说明](https://developer-docs.amazon/sp-api/docs/orders-api-migration-guide#pricing-attribute-mappings)

项目既有的 EU 商品价格计算和 OMS 路径期待含税的 `ItemPrice`。适配器在相应市场分支中恢复这个口径：

```php
$country = OrderMapper::mapFromSaleschannelToRegion($order->salesChannel->marketplaceName ?? '');
if ((new self())->getRegionByCountry($country) === 'eu') {
    // The existing EU price calculation and OMS expect a tax-inclusive
    // ItemPrice. v2026 ITEM proceeds exclude separately reported item tax.
    // Shipping and gift-wrap calculations use the delivery country instead.
    foreach ($items as $item) {
        if (isset($item->itemPrice->amount, $item->itemTax->amount)) {
            $item->itemPrice->amount = bcadd($item->itemPrice->amount, $item->itemTax->amount, 2);
        }
    }
}
```

两个细节值得留意。

第一，判断依据是销售市场映射，而不是随手套用收货国家；代码注释说明，运费和礼品包装计算另有收货国家逻辑。这是当前系统的历史业务契约，不应被总结成“所有欧洲地址都要加税”。

第二，只有商品价格和商品税都存在时才相加。如果商品价缺失、只剩税额，不能把税额伪装成商品价。仓库专门用 `testDoesNotCreateEuItemPriceFromTaxAlone` 固定这个边界。

金额通过十进制字符串和 `bcadd`、`bccomp` 运算，避免在这一适配过程中引入二进制浮点误差。这里使用的两位精度是项目已有契约的一部分；复制到其他币种或舍入制度时，应重新验证精度规则。

## 总折扣不能同时充当商品折扣

假设一条订单行的商品折扣为 `10.00`，运费折扣为 `5.00`，新版总折扣为 `15.00`。如果把总折扣填进 `promotionDiscount`，又把运费折扣填进 `shippingDiscount`，下游分别扣减两个字段时就会扣掉 `20.00`。

因此，适配器从明细中分别提取两个分量。下面是 DTO 构造参数的节选，省略了其他字段：

```php
shippingDiscount: self::detail($item, 'DISCOUNT', 'SHIPPING'),
// DISCOUNT subtotal includes shipping; using it here discounts
// shipping twice because downstream maps ShippingDiscount separately.
promotionDiscount: self::detail($item, 'DISCOUNT', 'ITEM'),
promotionDiscountTax: self::detail($item, 'TAX', 'DISCOUNT'),
```

最后一行还不是完整解决方案。新版 `TAX / DISCOUNT` 合并了旧版商品折扣税和运费折扣税，并不提供这里所需的独立拆分。这个临时值在需要补充数据时，会被后面的恢复逻辑替换。[折扣税字段的迁移说明](https://developer-docs.amazon/sp-api/docs/orders-api-migration-guide#pricing-attribute-mappings)

仅有一个合计，通常无法唯一恢复两个分量。例如总折扣税 `2.50`，可能是商品 `2.00`、运费 `0.50`，也可能是其他分配。按折扣金额比例分摊，等于额外引入未经证实的假设；遇到不同税率或免税分量时，假设尤其容易失效。

## 回退到旧 API，也要验证两份数据能否拼接

`getOrderData` 先拉取新版完整订单；只有适配器判断折扣信息需要补足时，才获取旧版订单行。

```php
$newOrder = $this->fetchOrder($orderId, ['BUYER', 'RECIPIENT', 'FULFILLMENT', 'PROCEEDS']);
$order = OrdersV20260101Adapter::toLegacyOrder($newOrder);
$legacyItems = OrdersV20260101Adapter::needsLegacyDiscounts($newOrder)
    ? $this->getOrderItemsV0($orderId) : [];
$items = OrdersV20260101Adapter::toLegacyItems($newOrder, $legacyItems);

return [
    'order' => $order,
    'order_items' => $items,
    'shipping_address' => $order->shippingAddress,
    'amazon_discount_taxes' => OrdersV20260101Adapter::discountTaxes($items),
];
```

这里的 `PROCEEDS` 是获取价格明细需要的数据集。旧版补充是按受影响订单进行的，不是对每条订单行单独再发一遍请求。`getOrderItemsV0` 会获取所有分页；请求失败时抛出异常，而不是返回部分订单行并猜测剩余金额。

恢复过程中，旧版行先按 `orderItemId` 匹配，还要核对数量、币种、已有折扣分量和合计。尤其重要的是下面这个规则：

```php
$taxField = $field . 'Tax';
$tax = $legacy->$taxField;
// Missing is only safely zero when this component has no discount.
if ($tax?->amount === null && bccomp($discount->amount, '0', 2) === 0) {
    $tax = new LegacyMoney($currency, '0.00');
}
if ($tax?->amount === null || $tax->currencyCode !== $currency) {
    throw new RuntimeException("Missing or inconsistent v0 {$taxField} for item {$source->orderItemId}.");
}
$item->$taxField = clone $tax;
```

**缺失不是零。** 只有该分量明确没有折扣时，这里的缺失折扣税才被允许补成零；有折扣却缺少税额，意味着数据不足，应该失败并让调用链处理，不能以一个看似完整的订单继续计算。

两次请求可能观察到不同版本的订单，所以代码还对合计做交叉检查：

```php
$discountSum = bcadd($item->promotionDiscount->amount, $item->shippingDiscount->amount, 2);
$taxSum = bcadd($item->promotionDiscountTax->amount, $item->shippingDiscountTax->amount, 2);
if (($total && bccomp($total->amount, $discountSum, 2) !== 0)
    || ($combinedTax && ($combinedTax->currencyCode !== $currency || bccomp($combinedTax->amount, $taxSum, 2) !== 0))) {
    throw new RuntimeException("Inconsistent v0 discount totals for item {$source->orderItemId}.");
}
```

这可以拒绝明显不一致的组合，却不等于两个 API 请求拥有原子快照：数量和金额相同的两份响应，其他字段仍可能变化。它是一道针对关键业务不变量的防线，不是跨系统事务保证。

旧 API 回退还存在寿命边界。它买到了迁移时间，也留下了将来移除依赖的工作：需要持续核对官方能力与退役计划，找到有证据的替代数据来源，并让测试证明新来源保持同样契约。不能把“临时兼容”变成无人追踪的永久依赖。

## 规范化一次，把同一份语义送到所有下游

如果 D365 从 DTO 读取一套税额，而 OMS 在另一个地方再次解析原始响应，两个系统可能出现不同的分配结果。

当前实现从已经规范化的 DTO 生成 OMS 需要的折扣税上下文：

```php
public static function discountTaxes(array $items): array
{
    $taxes = [];
    foreach ($items as $item) {
        // Derive OMS context from the same normalized DTO used by D365.
        $taxes[$item->orderItemId] = [
            'item' => $item->promotionDiscountTax->amount ?? '0.00',
            'shipping' => $item->shippingDiscountTax->amount ?? '0.00',
        ];
    }

    return $taxes;
}
```

这个上下文与订单一起传递，仓库还检查它在队列序列化之后仍然保留。边界处计算正确只是第一步，异步传输不能把恢复出来的信息又丢掉。

## 契约测试应该比较业务结果

只断言 DTO 有某个字段，证明不了下游算出的价格仍然正确。这里更有价值的测试把旧版数据作为行为基线，再把新版适配后的数据送进实际的 OMS 和 D365 映射，比较输出。

| 已有测试关注点 | 它防止什么回归 |
| --- | --- |
| EU、UK、跨市场、数量为 2、缺失或零税额 | 商品价口径和单价计算改变 |
| 商品折扣、运费折扣、混合折扣、不同折扣税率 | 重复扣减和错误分摊 |
| 旧版缺字段、币种不一致、数量不匹配、合计不一致 | 把不可靠数据拼成完整订单 |
| OMS 与 D365 的旧版 / 新版输出对比 | 适配看似成功，下游行为却改变 |
| 首次请求限流重试、失败响应不猜测税额 | 回退失败被静默降级 |
| 折扣税上下文经过队列序列化 | 异步边界丢失业务信息 |

这些断言展示了测试意图和覆盖方向，并不意味着所有外部订单变体都已经被覆盖。新枚举、新的价格明细形状，以及旧版补充数据消失后的路径，仍需要持续补充。

## 迁移完成的标准，是业务契约仍然成立

适配器的作用是把外部协议变化挡在边界，同时明确不能安全翻译的情况。对字段改名可以做映射；对金额口径要做转换；对缺失信息要补充或失败；对无法支持的请求条件要显式拒绝。

这次代码最值得借鉴的地方，是把“不知道”保留下来，而不是用零、默认值或比例分摊消灭它。财务相关数据一旦进入下游，错误往往比异常更晚被发现；让异常停在边界，反而更容易查清问题。

### 代码阅读索引

以下为 Kitsilano 仓库相对路径：

- `app/Services/Amazon/OrdersV20260101Adapter.php`：DTO、价格与折扣恢复。
- `app/Services/OrdersApiCompat.php`：调用参数和响应形状兼容。
- `app/Services/SPAPIService.php`：数据获取、选择性回退及分页。
- `tests/Unit/Services/Amazon/OrdersV20260101AdapterTest.php`：金额不变量与下游契约断言。
- `tests/Unit/Services/OrdersApiCompatTest.php`：请求路由、失败处理与队列传输断言。

同系列：[Acadia 图片同步与分阶段迁移](/blog/acadia-image-sync-identity/) · [履约补偿为什么不只是重试](/blog/kitsilano-fulfillment-reconciliation/)
