---
title: "从 URL 到稳定身份：Acadia 的图片同步与分阶段数据库迁移"
description: "一次 Shopify 图片导入改造，串起稳定业务键、完整快照校验、父行锁、批量 upsert，以及历史数据回填和唯一索引的上线顺序。"
pubDate: 2026-10-03
tags: ["Acadia", "Laravel", "MySQL", "数据同步"]
draft: true
---

商品图片同步看起来很简单：拿到图片列表，逐条写入数据库。然而，一张图片更换 CDN 地址后，是更新旧记录，还是新增一条？上游删掉的图片什么时候清理？如果两个任务同时导入同一个商品，又会发生什么？

Saxx-Acadia 的 `ProductImportJob` 把这些问题放在同一条链路里解决。它的技术价值不只是把循环写入改成批量写入，而是重新定义了图片身份，并安排了一个可以跨越历史数据的迁移过程。

本文基于本地代码阅读快照，基准提交为 `4bd8cf044f6e`。代码块均为原代码节选；文中的例子使用虚构数据。测试部分描述仓库已有的断言，不代表本文执行过该项目测试，也不代表生产部署状态。

## URL 是属性，图片 ID 才是身份

设想一张上游图片的 ID 是 `101`，第一次导入的 URL 是 `image.png?v=1`，第二次变成 `image.png?v=2`。如果匹配依赖 URL，数据库就很容易留下两条记录。同样，两个不同的上游图片 ID 也可能指向同一个 URL；按 URL 合并会错误地丢失一个对象。

当前实现使用 `(shopify_image_id, type)` 表达 PDP 图片的业务身份，用 `img_url` 保存可变化的地址。PDP 指商品详情页图片。导入时先查出已有记录的本地主键，再一次性构造待写入数组：

```php
$incomingIds = array_column($this->product['images'], 'id');
$existingIds = Image::query()
    ->where('type', 'pdp')
    ->whereIn('shopify_image_id', $incomingIds)
    ->lockForUpdate()
    ->pluck('id', 'shopify_image_id');
$images = [];

foreach ($this->product['images'] as $image) {
    $images[] = [
        'id' => $existingIds[$image['id']] ?? null,
        'shopify_image_id' => $image['id'],
        'img_url' => $image['src'],
        'type' => 'pdp',
        'style_id' => $colour->getAttribute('style_id'),
        'colour_id' => $colour->getAttribute('id'),
    ];
}

Image::query()->upsert($images, ['shopify_image_id', 'type'], [
    'shopify_image_id', 'img_url', 'style_id', 'colour_id',
]);
```

这段代码同时携带业务 ID 和数据库主键，并不是冗余。迁移第一阶段只有普通索引，尚未建立业务键的唯一约束。把已匹配的本地主键带进批量写入，才能在这一阶段更新已有记录。

一个容易忽略的细节是：**MySQL 并不会因为 `upsert` 第二个参数写了两个字段，就自动按这两个字段判重。** Laravel 官方文档明确说明，MySQL 和 MariaDB 使用数据库实际存在的主键、唯一索引识别冲突，忽略该参数。因此，上面的查询、主键回填和后续唯一索引是一套配合机制。[Laravel 的 upsert 文档](https://laravel.com/framework/docs/13.x/eloquent#upserts)

PLP，也就是商品列表页图片，则按颜色记录和 `type='plp'` 管理，`shopify_image_id` 保持为空。两类图片拥有不同的身份模型，不能为了形式统一而强行共用同一个匹配规则。

## 删除之前，先证明输入是一个有效快照

同步不仅要写入，也要删除已经消失的图片。危险在于：接口返回字段缺失、解析失败或者收到分页中的一页时，把它当成完整列表就可能误删。

Acadia 在操作图片之前先校验输入：

```php
Validator::make($this->product, [
    'images' => 'present|array',
    'images.*.id' => 'required|integer|min:1|distinct',
    'images.*.src' => 'required|string|max:255',
    'plpImage' => 'nullable|string|max:255',
])->validate();
```

这里最有意义的是 `present|array`：缺少 `images` 字段和明确传入空数组，是不同的情况。前者无法证明图片已全部删除，校验会失败；后者表达“这个快照没有 PDP 图片”。数组元素的 ID 还必须有效且互不重复。

写入完成后，只在当前颜色、当前图片类型内删除不再出现的 ID：

```php
Image::query()
    ->where('colour_id', $colour->id)
    ->where('type', 'pdp')
    ->whereNotIn('shopify_image_id', $incomingIds)
    ->delete();
```

这个范围约束避免误删其他颜色和 PLP 图片。不过，结构校验无法证明上游列表完整：一个格式正确、只包含半数图片的数组仍然会通过校验。**按差集删除的前提是完整快照；增量事件、分页结果和局部补丁不能直接套用这个规则。** 如果输入协议改变，删除策略也必须跟着改变。

历史 PDP 记录没有上游 ID 时，还存在迁移期间的特殊行为：非空列表不会通过这条 `NOT IN` 条件清除其 `NULL` ID 记录；明确的空列表则会清除该颜色的全部 PDP 图片。这是后续历史清理流程必须理解的边界。

## 为什么锁颜色行，而不只锁图片行

图片同步运行在数据库事务内。进入协调过程后，代码先锁住父级颜色记录：

```php
$colour = Colour::query()
    ->whereKey($colour->id)
    ->lockForUpdate()
    ->firstOrFail();
```

只锁已存在的图片，会遇到一个问题：首次导入时根本没有图片行可锁。两个任务可能同时查出空结果，然后都准备插入。颜色行已经存在，适合作为同一颜色图片同步的协调点。使用同一入口的任务先取得这把锁，再读取和更新图片状态。

锁不是单独的一行魔法代码：它依赖外层事务，随事务提交或回滚释放。MySQL 的锁定读取文档也说明了这一事务前提。[MySQL 锁定读取文档](https://dev.mysql.com/doc/refman/8.0/en/innodb-locking-reads.html)

这把锁控制的是同一颜色内的并发，不是所有商品的全局并发。跨颜色错误复用上游 ID、其他绕过此入口的写入，仍需要业务假设和数据库约束兜底。它也不能替代“快照是否完整”的校验。

另一处值得注意的设计是：商品时间戳没变时，代码仍然执行图片同步：

```php
if ($colour = $this->unchangedProductColour()) {
    // Images still need reconciliation and the initial Shopify ID backfill.
    DB::transaction(fn () => $this->upsertProductImages($colour));
    return;
}
```

商品内容没有更新，不等于本地已经完成新的身份字段回填。如果把“上游没变化”当作跳过一切工作的理由，历史记录就可能永远停留在旧格式。数据版本与业务更新时间，是两条不同的轴。

## 批量化要用什么证据衡量

代码将图片匹配和写入移出逐图数据库循环：一次查询建立 ID 映射，一次批量 upsert，再执行范围删除。构造数组仍然要遍历图片，但 SQL 往返次数不必随每张图片线性增加。

仓库的 `imageQueryCountDoesNotGrowWithTheNumberOfImages` 用同一个商品分别导入 1 张和 20 张图片，记录涉及 `saxx_image` 的查询数量。下面节选计数与断言部分，省略外层循环，并补充中文注释说明断言的位置：

```php
DB::enableQueryLog();
DB::flushQueryLog();
try {
    (new ProductImportJob($product, 'ca'))->handle();
    $counts[] = count(array_filter(DB::getQueryLog(), fn ($query) =>
        str_contains($query['query'], 'saxx_image')
    ));
} finally {
    DB::disableQueryLog();
}

// 在两组导入完成后比较：
$this->assertGreaterThan(0, $counts[0]);
$this->assertSame($counts[0], $counts[1]);
```

这是一个具体的结构性回归检查：防止后来又把数据库操作塞回逐图循环。它没有证明整个导入任务是常数时间，也没有给出吞吐提升百分比。SQL 中处理的行数、数组内存和锁等待仍然会随输入或并发变化。

其他已有测试还覆盖了：同一个上游 ID 换 URL 后保留原记录身份；不同 ID 使用同一 URL 时保留两条记录；空快照删除 PDP、保留 PLP；无效快照失败后保留原图片；未变化商品也会回填 ID。这些行为测试与查询数量检查互相补充。

## 唯一约束为什么最后才上线

直接新增非空唯一列，在历史数据尚未满足约束时会失败。项目的迁移说明采用以下顺序：

| 阶段 | 动作 | 进入下一阶段的证据 |
| --- | --- | --- |
| 扩展结构 | 新增可空 ID 列和普通索引，部署兼容导入代码 | 新旧数据都可处理，队列工作进程已更新 |
| 回填身份 | 全区域重新导入，包括没有变化的商品 | 所有派发的导入任务成功完成，失败项已补齐 |
| 清理历史 | 预览并清理旧的无 ID PDP 记录 | 已核实导入覆盖范围和清理候选 |
| 收紧约束 | 显式运行后续迁移，建立业务键唯一索引 | 无遗留空 ID PDP，无重复非空业务键 |

“派发任务结束”只说明任务进入了队列；“过了一夜”也无法证明消费者没有失败。这里真正需要的是所有区域和任务的覆盖证据。

历史清理命令还有一个细节：读取候选主键后，删除时重新套用候选条件。

```php
$orphans = Image::query()->where('type', 'pdp')->whereNull('shopify_image_id');

$deleted = 0;
$orphans->select('id')->chunkById($chunk, function ($images) use ($orphans, &$deleted) {
    // Recheck eligibility in case an import populated the ID after selection.
    $deleted += (clone $orphans)->whereKey($images->modelKeys())->delete();
});
```

如果某条记录在读取与删除之间被导入任务补上 ID，它就不应再被清理。重新检查条件缩小了这个竞争窗口。按主键推进分批处理，也避免依赖不断缩小的数据集的页码偏移。

但 `NULL` 本身不是“孤儿”的证明：可能是尚未成功导入的商品。因此，预览数量和分批删除都不能替代回填覆盖确认。代码中的 `--dry-run` 提供了审查入口，并不自动证明删除安全。

后续迁移被放在默认迁移扫描之外，要求显式执行。它先拒绝仍有空 ID PDP 的状态，再用一个 `ALTER TABLE` 建立 `(shopify_image_id, type)` 唯一索引并删除旧索引。这个安排让“允许新旧数据共存”和“最终强制唯一”分开上线；单条 DDL 也不等于无锁、无停机，执行成本仍需结合实际数据库版本和表规模评估。

## 可以带走的设计原则

一次稳定的同步，至少要回答三个问题：对象由什么字段识别，输入代表完整快照还是局部变化，以及失败后如何恢复到可继续处理的状态。

Acadia 的实现把答案落在可检查的代码中：上游 ID 区分身份与属性；校验和范围删除表达快照契约；事务与父行锁协调同一实体的写入；普通索引、回填和唯一约束分阶段衔接。真正值得复用的是这些前提之间的关系，而不是把某个 `upsert` 调用复制到另一个项目。

### 代码阅读索引

以下为 Saxx-Acadia 仓库相对路径：

- `app/Jobs/Shopify/ProductImportJob.php`：图片协调、未变化商品回填、批量写入。
- `app/Console/Commands/Maintenance/DeleteOrphanImages.php`：历史清理与删除时复核。
- `docs/development/shopify-image-cleanup.md`：部署和回填顺序。
- `database/migrations/follow-up/2026_09_11_000001_make_shopify_image_id_unique.php`：收紧唯一约束。
- `tests/Feature/Product/ProductImportJobTest.php`：身份、删除边界和查询数量断言。

同系列：[订单 API 迁移如何保持业务语义](/blog/kitsilano-api-migration-contracts/) · [履约补偿为什么不只是重试](/blog/kitsilano-fulfillment-reconciliation/)
