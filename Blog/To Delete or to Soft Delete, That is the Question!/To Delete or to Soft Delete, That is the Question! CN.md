软删除是一种在业务应用程序中广泛应用的模式。它允许你将某些记录标记为已删除，而不是真正从数据库中删除。用户将不能选择标记为软删除的记录，同时旧的记录仍可以引用软删除的记录。

## 什么情况下应该使用软删除?

如果你随机询问一些开发人员，他们很可能会说软删除主要用于恢复已删除的记录。但是，如果这是唯一促使你使用软删除的因素，那么请三思，这个理由并不充分。
在决定项目中是否使用软删除时，除了考虑 “支持数据还原” 这一诉求，有以下几点也是需要考虑的：

1. 历史跟踪和审计（比如有法律规定）
2. 保持参照完整性，避免级联删除
3. 你需要 “优雅地” 删除。例如。长期运行的业务流程可能需要用到 “被删除” 数据来完成业务。


如果你发现自己的业务需求与上面的列表匹配，则意味着你可能需要在应用程序中实现 “软删除” 功能。

实现软删除似乎挺容易！你只需将 “Is_Deleted” 或 “Delete_Date” 列添加到数据库表（或文档属性）中，并将应用程序代码中的所有 `delete` 语句替换为 `update` 语句。同时，你需要修改所有查询语句以加入对删除标记字段的判断。 嗯? 是不是创建一个视图更好？ 使用物化视图怎么样？ 在这种情况下，我们可能需要实现 “instead of ” 触发器或使用存储过程 API 来处理视图。在删除实体时，你可能希望删除其引用的某些属性。在这种情况下，为了能够恢复删除，还需要存储删除上下文。还有... 。对，开发人员的生活就是因此变地复杂。


## 实施软删除的挑战

实际上，实现 “合适” 的软删除并非易事。有时，开发人员选择不实施软删除，只是因为这需要做很多工作。你可能会想到其他选项，例如使用专门的“回收站”表，该表是原始表的一个镜像，存储从 “原始” 表中删除的记录。
但是，如果决定实施软删除，就需要对以下情况充分考虑。

**数据库支持** 将记录标记为已删除的问题之一是唯一键。 举例来说-我们需要将用户及其电子邮件存储在一起。电子邮件必须是唯一的，因此，我们需要为存储用户的表创建唯一的索引。而且，如果删除了用户，理论上我们应该可以使用相同的电子邮件再插入新用户。所以，你的数据库应该能够处理此类情况。方法之一 - 在唯一键中包含一个软删除标记字段。如果你的标记字段不是简单的布尔值，而是删除时间戳，则可以解决问题。

另外一个方法 - 使用特殊的唯一索引。比如，在 PostgreSQL 中，有 [分部唯一索引](https://www.postgresql.org/docs/12/indexes-partial.html) ，在 Mysql 中有
[过滤索引](https://docs.microsoft.com/en-us/sql/relational-databases/indexes/create-filtered-indexes)。这些机制允许在构建索引时以断言的方式排除一部分记录。所以对于软删除机制的实现，我们可以排除掉已标记为删除的记录。

如果你的数据库支持视图触发器，则可以使用视图和 `instead of` 类型的触发器来创建数据模型。但是这种方法会影响性能，并且无法通过对基础表应用唯一约束来解决问题。

**应用程序支持** 确保所有对涉及到软删除实体的查询都经过了细致的检查，否则可能导致意外的数据泄漏和严重的性能问题。在理想情况下，软删除对开发人员应该早透明的。因此，如果可能，请尝试设计自己的数据访问 API，并防止开发人员使用其他工具来访问数据库（例如直接JDBC连接）。你甚至可能要覆盖标准的 API，例如 JPA 的 Entity Manager，以正确地支持软删除。


**重新审视对标准 API 的使用** 首先，可能得放弃使用一些标准的 DB 和 JPA 功能。如果你已经精心设计和调整了数据库表和约束模型以及 JPA 层，那么引入软删除要求你考虑以下事项：

1. 不能使用标准级联策略。在进行软删除的情况下，你不能删除记录，而是要对其进行更新。如果仍然需要级联删除，触发器可以帮助你实现此行为。
   
2. 对于触发器 - 你也需要进行检查，确保之前在 pre- 和 post- delete 触发器中的代码可以正确地迁移到 pre- post- update 触发器。这些执行软件删除的代码不能影响正常的 update 处理。 

3. JPA 的生命周期回调 `@PreRemove` 和 `@PostRemove` 将失效。 需要寻找其它方法来实现之前在这两个回调中实现的逻辑。这里与触发器需要做的修改相似，你需分辨出普通的更新和软删除执行的更新。
 
**你仍然需要满足监管要求** 比如 GDPR ，规则要求如果用户请求删除数据，则数据必须被完全删除。为了合规，除了软删除，你必须实现一个 “清除” 功能来执行物理删除。在大部分情况下，物理删除会影响数据库中相关数据。所以，你需要保持谨慎，不要删错关键业务数据。


## CUBA 平台中的软删除实现

CUBA 平台支持开箱即用的软删除功能，并且软删除功能是默认启用的。在大多数情况下，开发人员不必考虑是否使用软删除。CUBA 中的数据操作 API 与众所周知的 JPA API 99% 相同。作为开发人员，你可以使用 `EntityManager` 的 `remove()`，甚至可以使用任意的 JPQL 处理应用程序中的数据。

在底层，CUBA 平台会拦截对 EntityManager 和 JPQL 查询的所有调用，并根据应用程序设置进行转换。框架做了大量复杂的工作来保证正确的应用程序行为：`被删除` 的记录将不受更新影响，`DELETE` 语句将转换为 `UPDATE`，而 `SELECT` 将不再获取 “已删除” 数据 。

该实现基于三个主要的设计：

1.`SoftDelete` 接口。CUBA 使用JPA（EclipseLink）作为访问数据的默认框架。如果你的 JPA 实体实现了 `SoftDelete` 接口，则 CUBA 就不会从数据库中删除数据，而只是将其标记为已删除。

2. `EntityManager` 类。这是处理 CUBA 中数据的主要 API。它拦截对数据库的所有请求，并根据实体类型及其 `soft delete` 属性值（启用或禁用）来确定是调用 `delete` 还是 `update` 语句。因此，你可以根据需要在运行时禁用实体的软删除。尽管 CUBA 平台在内部执行的是更新语句，但这种方法仍允许我们调用与实体删除相关的 JPA 回调。

3. 核心框架中 `JoinExpressionProvider` API 接口。`SoftDeleteJoinExpressionProvider` 实现生成一个谓词，以检查实体状态（是否删除）及其在查询中的位置。例如。如果我们选择引用了“已删除”的记录，`SoftDeleteJoinExpressionProvider` 实现会给查询添加适当的条件以选择 “已删除” 记录。

这些 API 使软删除过程对开发人员透明，并且消除了每次调用 JPA API 或发送 JPQL 查询时回答 “删除或软删除” 问题的负担。

CUBA 的 API 能满足绝大部分情况，但是在这个世界上没有完美的事物。为了灵活，CUBA 仍然允许你执行原生 SQL，并允许获得对基础 DataSource 对象的访问。因此，即使使用解决了许多软删除问题的框架，你也需要对低级查询进行仔细检查。

## 总结

即使只对一个实体使用软删除，也会给你带来一堆新的开发约定。你需要时刻留意几个问题：删除过程不仅是删除，有时是更新、数据库表中的数据并非都可用。这意味着你的代码应该对应用程序的所有层面都进行考虑- 从数据库架构（文档结构等）到用户界面，然后做整体设计。

使用合适的框架，你将能够解决所有这些问题，但你仍应注意，已经“删除”了数据仍然存在，并基于这一点来制定相应的编码标准和约定。

