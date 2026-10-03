---
title: "履约补偿不只是重试：从 Kitsilano 的同步账本看可靠性边界"
description: "结合补偿命令、调用记录和事件监听器，分析有限扫描、冷却、远端状态核对与重放副作用；也解释为什么有重试并不等于只执行一次。"
pubDate: 2026-10-03
tags: ["Kitsilano", "Laravel", "可靠性", "系统集成"]
draft: true
---

仓库已经完成出库，本地也有物流单号，但电商平台没有更新。这样的状态差异，不能只靠队列的即时重试解决：任务可能耗尽重试次数，进程可能中断，或者远端成功之后本地保存失败。

Kitsilano 的 `ReconcileShopifyFulfillments` 提供了一个有价值的补偿入口：根据本地业务状态与调用账本筛选候选，再重放履约事件。但它也展示了补偿机制最容易被忽略的部分——候选判定只是线索，事件重放可能产生多个副作用，而同步成功与成功记录也不是同一件事。

本文基于本地阅读快照，基准提交 `54314e8d4aba`。代码块为原代码节选；测试结论只描述仓库已有断言，本文未执行源项目测试或任何履约命令。**阅读时，这个补偿命令的定时调度配置仍被注释，不能据此认定生产环境正在自动运行它。**

## 从“某次调用失败”转向“业务状态是否收敛”

队列重试围绕一次任务展开。补偿扫描则重新观察当前业务状态：订单属于 Shopify 渠道、履约已完成、有物流单号，并处在配置的时间窗口内。

代码默认回看 72 小时，单次最多选取 50 条。通过基础条件后，它识别两类候选：本地没有远端履约 ID；或者有远端 ID，但没有对应动作的成功调用记录。

```php
->where(function (Builder $query): void {
    // Case A: fulfillment never created in Shopify
    $query->whereNull('fulfillment_id')
        // Case B: fulfillment_id set but no successful shopify_calls record
        ->orWhere(function (Builder $query): void {
            $query->whereNotNull('fulfillment_id')
                ->whereDoesntHave('shopifyCalls', function (Builder $query): void {
                    $query->whereIn('action', [
                        ShopifyCall::ACTION_FULFILLMENT_CREATE,
                        ShopifyCall::ACTION_TRACKING_UPDATE,
                    ])
                    ->where('status', ShopifyCall::STATUS_SUCCESS);
                });
        });
})
->limit($maxPerRun)
->get();
```

第二类候选很重要。远端 ID 只是某个对象的引用，不是“本次同步完整结束”的证据。监听器创建远端履约后，先保存本地远端 ID，再保存成功调用记录；进程如果在两次保存之间失败，就会留下“有 ID、无成功账本”的状态。

不过，两个条件都属于本地推断。本地无 ID，也可能是远端创建成功但响应丢失；本地无成功记录，也可能是远端早已正确同步。它们适合筛选待核对项，不足以直接证明远端缺少履约。

还有一个命名陷阱：`$fulfillment->id` 是本地记录主键，`$fulfillment->fulfillment_id` 是 Shopify 的远端 ID；而调用账本 `shopify_calls.fulfillment_id` 关联的却是前者。追踪补偿链路时，要先弄清每个 ID 的归属，不能仅凭字段同名推断它们指向同一对象。

## 调用账本把重试从猜测变成可解释的决定

`ShopifyCall` 区分动作与结果：履约创建、物流更新、支付捕获等动作，以及 `success`、`error`、`skipped` 等状态。请求和响应记录帮助还原某次尝试，但这里不展示实际业务载荷。

补偿命令会过滤最近一小时存在错误记录的履约：

```php
$unsyncedFulfillments = $unsyncedFulfillments->reject(function (Fulfillment $fulfillment): bool {
    return ShopifyCall::query()
        ->where('fulfillment_id', $fulfillment->id)
        ->where('status', ShopifyCall::STATUS_ERROR)
        ->where('created_at', '>=', Carbon::now()->subHour())
        ->exists();
});
```

冷却期降低了对刚刚失败项的立即重放频率，数量上限约束了单次事件派发规模，`--dry-run` 则只展示候选、不发送事件。它们让补偿过程更容易观察和控制，但要准确理解边界。

当前实现先在 SQL 中取上限，再在内存里过滤近期错误。如果前 50 条都在冷却，后面还有可处理项，这次运行仍可能没有实际进展。查询也没有明确的排序和游标，不能假设各候选一定能公平获得处理机会。基础查询后的逐条错误查询，还会增加数据库往返。

此外，回看使用履约记录的 `created_at`：很早创建、刚刚才更新物流的记录可能落在窗口外。错误冷却查询没有限定履约创建或物流更新动作，因此同一履约关联的其他错误也可能影响入选。上限和窗口是运营取舍，不是天然完整的可靠性证明。

## 补偿监听器要重新核对远端状态

如果本地已有远端履约 ID，`CreateShopifyFulfillment` 会先读取 Shopify 履约，比较物流单号；相同就提前返回，不同才发起物流更新。

```php
$existingFulfillmentResponse = $this->shopifyClient->inStore("{$region}.private")
    ->get("orders/{$fulfillment->order_id}/fulfillments/{$fulfillment->fulfillment_id}");
$existingFulfillment = json_decode($existingFulfillmentResponse->getBody(), true)['fulfillment'] ?? [];
$existingTrackingCode = $existingFulfillment['tracking_number'] ?? null;
if ($event->fulfillment->tracking_code === $existingTrackingCode) {
    return;
}
```

这避免了仅凭本地账本缺失就无条件修改远端。但提前返回路径没有保存新的成功或核对记录。假设这条履约此前没有成功账本：扫描选中它，监听器发现远端已正确便返回，下一轮扫描仍然可能选中它。

**一次没有必要的远端写入被避免了，但本地候选状态还没有收敛。** 一个改进方向是明确记录“已核对、目标状态一致”的结果，并让扫描识别这个证据。记录应说明核对了哪一版物流信息，而不是用一条永久成功标记掩盖将来的变化。

同样，当前扫描只要求存在历史成功调用。以前成功更新过物流，不代表后来修改的物流已经送达。若需要覆盖这类变化，成功证据应该关联目标状态版本，例如规范化物流内容的摘要或独立同步版本。这个建议尚未在本文阅读的实现中落地。

## 422 与超时，不能用同一种语言描述

监听器会把 HTTP 422 记录为 `skipped` 并返回，其他客户端异常记录为 `error` 后抛出：

```php
if ($e->getCode() === 422) {
    $shopifyCall->action = $shopifyCall->action ?: ShopifyCall::ACTION_FULFILLMENT_CREATE;
    $shopifyCall->response = $e->getMessage();
    $shopifyCall->status = ShopifyCall::STATUS_SKIPPED;
    $shopifyCall->save();
    return;
}
$shopifyCall->action = $shopifyCall->action ?: ShopifyCall::ACTION_FULFILLMENT_CREATE;
$shopifyCall->response = $e->getMessage();
$shopifyCall->status = ShopifyCall::STATUS_ERROR;
$shopifyCall->save();
throw $e;
```

这段代码没有依据响应内容进一步区分 422 原因，因此不能把它总结成“422 表示已经履约，无需处理”。被跳过的请求仍可能需要修复输入或人工核对；而 `skipped` 既不算扫描中的成功，也不触发错误冷却，因而仍可能反复入选。

超时则有另一种不确定性：不知道远端有没有执行。即使把本地 ID 和账本保存合并到一个数据库事务，也不能让远端 HTTP 请求与本地事务一起原子提交。远端成功、本地回滚后，下一次创建仍可能重复。

因此，需要把“明确拒绝”“可重试故障”“执行结果未知”和“已核对完成”区分开。结果未知时，优先寻找可核对的远端对象或协议提供的幂等机制，而不是仅凭异常再次创建。这里是可靠性改进方向，不是当前代码已经具备的保证。

## 重放事件时，要检查全部监听器

补偿命令没有直接调用物流更新服务，它重新发布了 `WarehouseFulfilled`。事件注册中，这个事件同时对应两个监听器：

```php
Events\Order\WarehouseFulfilled::class => [
    Listeners\Transaction\CapturePayment::class,
    Listeners\Order\CreateShopifyFulfillment::class,
],
```

于是，“补物流”可能重新进入支付路径。在手动捕获模式下，`CapturePayment` 会派发支付捕获任务。支付管理器会读取尚未捕获的交易、限制捕获金额，并对某些已捕获错误返回；这些是已有防线。但仅凭这些分支，不能推导出并发重放、远端成功但本地状态滞后等情况下支付一定不会重复产生影响。

审查一个重放入口时，应追踪所有消费者，而不只看命令名字。若目的仅是修复物流，一个只表达物流同步的命令或事件，可以缩小副作用范围；如果仍然复用完整业务事件，则每个监听器都需要明确的重复执行契约和验证。

这是本文没有运行补偿命令的原因：代码分析不需要实际触发订单处理，也不能把对某个监听器的理解当作整条事件链已被证明安全。

## 调度与测试分别证明什么

仓库已经写了每六小时运行一次、避免重叠、单服务器运行的调度配置，但阅读时整段仍处于注释中。即使启用这些限制，它们也主要控制调度任务的并发，并不自动保证异步监听器只执行一次。

已有 `ReconcileShopifyFulfillmentsTest` 覆盖两类候选，以及渠道、状态、物流单号、时间窗口、数量上限、近期错误和预览模式的选择行为。测试使用 `Event::fake()`，证明的是哪些事件会被派发，而不是 Shopify 或支付的真实最终效果。

`ShopifyCallLoggingTest` 另行覆盖创建或更新的成功记录、错误记录与 422 跳过记录。把两组测试连起来看，可以理解实现；但它们仍不等于完整补偿链路的端到端证明。

继续演进时，下面这些回归场景更接近真正的故障窗口：

| 应补充的场景 | 需要证明的行为 |
| --- | --- |
| 远端物流正确，本地无成功账本 | 核对后不再反复入选 |
| 历史更新成功，物流后来变化 | 新目标状态仍能被发现 |
| 前一批候选都在冷却 | 后面的有效候选能推进 |
| 远端创建成功，响应或本地保存失败 | 重新执行能够识别已有对象 |
| 两个消费者同时处理同一履约 | 创建、通知与支付副作用受控 |
| 不同原因的 422 | 已完成、不可恢复错误和待修复输入被区分 |

这些是建议新增的测试方向，不是现有测试已经通过的声明。

## 补偿的目标，是让状态和证据一起收敛

这个实现最有价值的起点，是不再只关注一次任务是否报错，而是通过业务状态与调用记录重新找回遗漏。它的限制也同样有启发：一次远端核对如果不留下可消费的证据，扫描可能永不结束；一条历史成功记录如果没有版本，也可能让新的变化永远被跳过。

可靠的补偿需要三件事配合：候选规则能发现目标状态差异，执行器能面对重复和结果未知，账本能证明当前目标已完成。重试次数、冷却期和调度锁是工具，最终要验证的是这三个环节是否真正衔接。

### 代码阅读索引

以下为 Kitsilano 仓库相对路径：

- `app/Console/Commands/Shopify/ReconcileShopifyFulfillments.php`：候选扫描、冷却与事件重放。
- `app/Listeners/Order/CreateShopifyFulfillment.php`：远端核对、创建与调用记录。
- `app/Models/Kitsilano/ShopifyCall.php`：动作和结果状态。
- `app/Providers/EventServiceProvider.php`：完整事件消费关系。
- `app/Listeners/Transaction/CapturePayment.php` 与 `app/Managers/Order/PaymentCaptureManager.php`：支付副作用链路。
- `app/Console/Kernel.php`：被注释的补偿调度配置。
- `tests/Feature/Fulfillment/ReconcileShopifyFulfillmentsTest.php`、`tests/Feature/Fulfillment/ShopifyCallLoggingTest.php`：选择行为与调用记录断言。

同系列：[Acadia 图片同步与分阶段迁移](/blog/acadia-image-sync-identity/) · [订单 API 迁移如何保持业务语义](/blog/kitsilano-api-migration-contracts/)
