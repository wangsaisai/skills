---
name: mapper-gen
description: 根据用户提供的字段 mapping 规则，自动生成或更新 Java 接口（Interface）、实现类（Impl）、以及 Model
  类。当用户描述类似「item.quantity = listing.quantitySold + listing.quantityAvailable」这样的
  mapping 时触发此 Skill。
disable: true
---

# mapper-gen

根据 mapping 规则自动生成 Java 代码的 Skill。

## 使用场景

当用户提供类似以下形式的 mapping 规则时使用：

```
item.quantity = listing.quantitySold + listing.quantityAvailable
item.id = listing.item.id
```

用户希望通过 mapping 描述字段关系，让 AI 自动生成：
- 目标接口（如 `Item.java`）
- 实现类（如 `ItemImpl.java`）
- Model 类（如 `Listing.java` 中追加嵌套对象字段）

## 前置条件

在执行任何生成操作之前，**必须先读取 `references/codebase.md`**，了解当前项目的包结构、命名规范和已有的类。

## 工作流程

### 第 1 步：解析 mapping

用户提供的 mapping 格式为：

```
目标类.目标字段 = 表达式
```

表达式中可能包含：
- 简单字段引用：`listing.quantitySold`
- 嵌套对象字段：`listing.item.id`
- 算术运算：`listing.quantitySold + listing.quantityAvailable`
- 方法调用：`listing.calcTotalAmount()`

**解析步骤：**

1. 提取等号左侧的「目标类名」和「目标字段名」
2. 提取等号右侧的「来源对象名」和「来源字段/方法」
3. 推断每个字段的 Java 类型：
   - 整数运算 → `int` 或 `long`
   - 金额相关 → `BigDecimal`
   - 字符串 → `String`
   - boolean 条件 → `boolean`
   - 集合操作 → `List<T>`（需根据上下文推断泛型类型）
4. 识别嵌套对象引用（如 `listing.item.id` 中的 `item`），这意味着 `Listing` 中需要有一个 `item` 字段，类型为被引用类（可能是新类型）

### 第 2 步：判断目标类是否存在

根据 `references/codebase.md` 中的已知类列表，判断：

- **目标接口是否存在**：如 `Item.java` → 在 `com.example.demo.api` 包中查找
- **目标实现类是否存在**：如 `ItemImpl.java` → 在 `com.example.demo.impl` 包中查找
- **嵌套 Model 是否存在**：如 `item` → 在 `com.example.demo.model` 包中查找

如果类不存在，则新建；如果已存在，则读取现有内容，检查是否已有对应字段，没有则追加。

### 第 3 步：确定接口方法

根据 mapping 推断接口应包含的方法：

| mapping 示例 | 推断的接口方法 | 返回类型 |
|-------------|---------------|----------|
| `item.quantity = ...` | `getQuantity()` | `int` |
| `item.id = ...` | `getId()` | `long` 或 `String` |
| `item.price = ...` | `getPrice()` | `BigDecimal` |
| `item.tags = ...` | `getTags()` | `List<String>` |
| `item.active = ...` | `isActive()` | `boolean` |

### 第 4 步：生成或更新代码

#### 4.1 新建接口（如不存在）

在 `src/main/java/com/example/demo/api/` 下创建接口文件：

- 包声明：`package com.example.demo.api;`
- 类注释：Javadoc 说明接口职责
- 方法签名：每个字段对应一个 getter 方法
- 如果字段涉及嵌套对象（引用的类），方法返回该嵌套类的接口类型

示例：

```java
package com.example.demo.api;

/**
 * Item 接口：商品项的核心属性
 */
public interface Item {

    /**
     * 获取商品数量
     *
     * @return 数量
     */
    int getQuantity();
}
```

#### 4.2 新建实现类（如不存在）

在 `src/main/java/com/example/demo/impl/` 下创建实现类：

- 包声明：`package com.example.demo.impl;`
- 持有被引用对象（如 `Listing`）作为依赖
- 持有嵌套对象（如 `Item`）作为依赖
- 实现所有接口方法，方法体为简单的委托或计算表达式
- 构造方法中对依赖进行 null 检查

示例：

```java
package com.example.demo.impl;

import com.example.demo.api.Item;
import com.example.demo.model.Listing;

public class ItemImpl implements Item {

    private final Listing listing;

    public ItemImpl(Listing listing) {
        if (listing == null) {
            throw new IllegalArgumentException("listing 不能为 null");
        }
        this.listing = listing;
    }

    @Override
    public int getQuantity() {
        return listing.getQuantitySold() + listing.getQuantityAvailable();
    }
}
```

#### 4.3 更新 Listing（如需要嵌套对象）

如果 mapping 中引用了 Listing 中不存在的嵌套对象（如 `listing.item.id`）：

1. 读取现有的 `Listing.java`
2. 在 model 包下创建被引用的嵌套 Model 类（如 `Item.java`）
3. 在 `Listing` 中添加嵌套对象字段（如 `private Item item;`）及其 getter/setter
4. 如果 Listing 需要无参构造，补充完整

**更新 Listing 示例（添加 `item` 字段）：**

```java
// 在 Listing.java 中追加：

/** 关联的商品对象 */
private Item item;

public Item getItem() {
    return item;
}

public void setItem(Item item) {
    this.item = item;
}
```
