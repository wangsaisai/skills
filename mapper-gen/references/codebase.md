## 项目基础信息

- **项目路径**: `/Users/bamboo/workspace/demo`
- **语言**: Java
- **包前缀**: `com.example.demo`
- **Java 版本**: 17

## 包结构

```
com.example.demo/
├── api/         # 接口定义（Xxx.java）
├── impl/        # 实现类（XxxImpl.java）
└── model/       # 数据模型（Xxx.java）
```

## 命名规范

| 类别 | 命名规则 | 示例 |
|------|----------|------|
| 接口 | 与功能同名，无后缀 | `Sale.java` |
| 实现类 | 接口名 + `Impl` | `SaleImpl.java` |
| Model | 与业务实体同名 | `Listing.java` |
| 包 | 按职责分：api / impl / model | - |

## 代码风格

- 每个类顶部写 Javadoc 文档注释，说明类的职责
- 方法使用 `@Override` 注解
- 方法参数和返回值类型要精确（用 `List<String>` 而非 raw type）
- 字段使用 `private`，提供 getter/setter
- 数字金额类使用 `BigDecimal` 而非 `double`
- boolean 字段用 `isXxx()` 命名
- Model 类需要无参构造方法（供序列化使用）
- 所有文件使用 UTF-8 编码

---

## 已知类

### com.example.demo.api.Sale

接口，定义销售订单的核心操作，包含 18 个方法：

| 方法名 | 返回类型 | 说明 |
|--------|----------|------|
| `getOrderId()` | String | 订单编号 |
| `getCustomerName()` | String | 客户姓名 |
| `getCustomerPhone()` | String | 客户电话 |
| `getProductName()` | String | 商品名称 |
| `getUnitPrice()` | BigDecimal | 商品单价 |
| `getQuantity()` | int | 购买数量 |
| `getTotalAmount()` | BigDecimal | 订单总金额 |
| `getDiscountRate()` | double | 折扣率 |
| `getActualPayment()` | BigDecimal | 折后实付金额 |
| `getOrderDate()` | LocalDate | 下单日期 |
| `getExpectedDeliveryDate()` | LocalDate | 预计发货日期 |
| `getOrderStatus()` | String | 订单状态 |
| `getShippingAddress()` | String | 收货地址 |
| `getSalespersonName()` | String | 销售人员姓名 |
| `getRemark()` | String | 备注 |
| `getProductTags()` | List\<String\> | 商品标签 |
| `isPaid()` | boolean | 是否已支付 |
| `isRefundable()` | boolean | 是否支持退款 |

### com.example.demo.impl.SaleImpl

实现类，持有 `Listing` 对象，通过委托模式实现 `Sale` 接口：

```java
public class SaleImpl implements Sale {
    private final Listing listing;

    public SaleImpl(Listing listing) {
        if (listing == null) {
            throw new IllegalArgumentException("listing 不能为 null");
        }
        this.listing = listing;
    }
    // ... 委托方法
}
```

### com.example.demo.model.Listing

Model 类，封装销售挂单数据，包含以下字段：

| 字段 | 类型 | 说明 |
|------|------|------|
| `orderId` | String | 订单编号 |
| `orderStatus` | String | 订单状态 |
| `orderDate` | LocalDate | 下单日期 |
| `expectedDeliveryDate` | LocalDate | 预计发货日期 |
| `remark` | String | 备注 |
| `productName` | String | 商品名称 |
| `unitPrice` | BigDecimal | 单价 |
| `quantityAvailable` | int | 可售库存 |
| `quantitySold` | int | 已售数量 |
| `tags` | List\<String\> | 商品标签 |
| `discountRate` | double | 折扣率 |
| `price` | String | 商品价格（供 Item.price mapping） |
| `item` | ItemModel | 关联的商品数据模型 |
| `customerName` | String | 客户姓名 |
| `customerPhone` | String | 客户电话 |
| `shippingAddress` | String | 收货地址 |
| `salespersonName` | String | 销售人员姓名 |
| `paid` | boolean | 是否已支付 |
| `refundable` | boolean | 是否支持退款 |

**业务计算方法：**

```java
// 总金额 = 单价 × 已售数量
public BigDecimal calcTotalAmount()

// 折后实付 = 总金额 × 折扣率
public BigDecimal calcActualPayment()
```

### com.example.demo.model.ItemModel

商品数据模型，作为 Listing 与 Item 接口之间的桥梁，避免 model 包与 api 包之间的循环依赖。

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | String | 商品 ID |

### com.example.demo.api.Item

接口，定义商品项的核心属性，包含以下方法：

| 方法名 | 返回类型 | 说明 |
|--------|----------|------|
| `getQuantity()` | int | 商品数量（可售库存 + 已售数量） |
| `getId()` | String | 商品 ID（来自 ItemModel） |
| `getPrice()` | double | 商品价格（String -> double） |

### com.example.demo.impl.ItemImpl

实现类，持有 `Listing` 对象，通过委托模式实现 Item 接口：

```java
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
        return listing.getQuantityAvailable() + listing.getQuantitySold();
    }

    @Override
    public String getId() {
        if (listing.getItem() == null) return null;
        return listing.getItem().getId();
    }

    @Override
    public double getPrice() {
        String priceStr = listing.getPrice();
        if (priceStr == null || priceStr.trim().isEmpty()) return 0.0;
        return Double.parseDouble(priceStr.trim());
    }
}
```
