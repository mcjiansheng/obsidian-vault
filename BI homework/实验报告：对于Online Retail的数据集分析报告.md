241250033 谢浩天

## 数据集选用
本试验使用的数据集是 UCI Machine Learning Repository 的 Online Retail数据集，该数据集记录了一家英国在线零售企业中2010年12月至2011年12月期间的交易明细，业务内容主要包括订单、商品、客户、国家、数量、单价和交易时间等信息。
原始文件：`Online Retail.xlsx`
![[截屏2026-05-26 19.07.26.png]]

## 数据清洗

使用python的pandas库进行了数据清洗，主要的清洗规则如下：
1. 标准化文本字段，例如：订单号、商品编码、商品描述和国家名称
2. 将订单时间转换为时间类型
3. 将数量、单价等字段转换为数值类型
4. 过滤`UnitPrice <= 0`的异常价格记录
5. 去除重复记录
6. 对缺失客户编号统一填充为`UNKNOWN`
7. 根据数量和订单号识别退货或取消的订单

清洗结果如下：

|指标|数值|
|---|--:|
|原始行数|541,909|
|清洗后行数|534,129|
|删除商品描述为空记录|1,454|
|删除异常价格记录|1,063|
|删除重复记录|5,263|
|退货 / 取消订单行数|9,251|
![[Pasted image 20260526191317.png]]

## 数据转换
在ETL转换阶段新增了以下字段

|字段|说明|
|---|---|
|`abs_quantity`|数量绝对值|
|`invoice_date_key`|日期键，格式为 `YYYYMMDD`|
|`invoice_year`|年份|
|`invoice_quarter`|季度|
|`invoice_month`|月份|
|`invoice_day`|日期中的日|
|`invoice_hour`|交易小时|
|`invoice_weekday`|星期|
|`gross_amount`|原始金额，`quantity * unit_price`|
|`sales_amount`|销售金额，退货记录为 0|
|`return_amount`|退货金额，正常销售记录为 0|
|`is_return`|是否退货|
|`is_cancelled_invoice`|是否取消订单|
|`order_type`|订单类型，`Sale` 或 `Return`|
## 数据装载

通过DBeaver，将数据csv导入PostgresSQL

![[截屏2026-05-26 19.16.25.png]]

## 数据仓库星型模型设计

本试验选择的主题为`Online Retail 销售订单分析主题`
该主题围绕销售金额、销售数量、订单数、客户数、退货金额等指标展开分析

### 星型模型结构
本试验选用星型模型，中心为销售事实表，周围包括时间、商品、客户和国家维度表

```
               DIM_CUSTOMER
                   |
DIM_PRODUCT -- FACT_SALES -- DIM_DATE
                   |
               DIM_COUNTRY
```

| 表名                | 类型  | 说明                      |
| ----------------- | --- | ----------------------- |
| `dw.dim_date`     | 维度表 | 时间维度，支持年、季度、月、日、小时等层级分析 |
| `dw.dim_product`  | 维度表 | 商品维度，描述商品编码和商品名称        |
| `dw.dim_customer` | 维度表 | 客户维度，描述客户编号和是否未知客户      |
| `dw.dim_country`  | 维度表 | 国家维度，描述销售发生的国家或地区       |
| `dw.fact_sales`   | 事实表 | 销售订单明细事实表，保存数量、金额、退货等度量 |
![[截屏2026-05-26 19.26.48.png]]
### 事实表设计
| 字段名                    | 说明                           |
| ---------------------- | ---------------------------- |
| `sales_key`            | 事实表代理键                       |
| `invoice_no`           | 发票号 / 订单号，作为退化维度保留在事实表中      |
| `date_key`             | 关联 `dw.dim_date`             |
| `product_key`          | 关联 `dw.dim_product`          |
| `customer_key`         | 关联 `dw.dim_customer`         |
| `country_key`          | 关联 `dw.dim_country`          |
| `invoice_date`         | 完整订单时间                       |
| `quantity`             | 原始数量，销售为正，退货为负               |
| `abs_quantity`         | 数量绝对值                        |
| `unit_price`           | 单价                           |
| `gross_amount`         | 原始金额，`quantity * unit_price` |
| `sales_amount`         | 销售金额，退货记录为 0                 |
| `return_amount`        | 退货金额，正常销售记录为 0               |
| `is_return`            | 是否退货                         |
| `is_cancelled_invoice` | 是否取消订单                       |
| `order_type`           | 订单类型，取值为 `Sale` 或 `Return`   |
事实表 `dw.fact_sales` 的粒度为：
`一张发票中的一个商品明细行`
也就是说，原始数据中的一行订单商品明细会对应事实表中的一行记录。该粒度可以支持从商品、订单、客户、国家和时间等多个角度进行灵活汇总。

![[截屏2026-05-26 19.29.57.png]]
### 维度表设计

#### 时间维度
`dim_date` 时间维度来源于ODS表中的`invoice_date`和相关派生字段

|字段名|说明|
|---|---|
|`date_key`|日期主键，格式为 `YYYYMMDD`|
|`full_date`|完整日期，不含时间|
|`year`|年份|
|`quarter`|季度|
|`month`|月份|
|`day`|日|
|`week`|年内周数|
|`weekday`|星期名称|
|`is_weekend`|是否周末|
支持以下OLAP层级：
```
Year -> Quarter -> Month -> Day
```

#### 商品维度

`dim_product` 商品维度来源于`stock_code` 和 `description`

|字段名|说明|
|---|---|
|`product_key`|商品维度代理键|
|`stock_code`|商品编码|
|`description`|商品描述|
### 客户维度

`dim_customer` 客户维度来源于 `customer_id`

|字段名|说明|
|---|---|
|`customer_key`|客户维度代理键|
|`customer_id`|客户编号|
|`is_unknown_customer`|是否为未知客户|
因为原始数据中存在客户编号丢失现象，所以保留了未知客户记录，将其标记为`UNKNOWN`,避免直接删除导致大量数据丢失的统计偏差

#### 国家维度
`dim_country` 国家维度来源于`country`

|字段名|说明|
|---|---|
|`country_key`|国家维度代理键|
|`country`|国家或地区名称|
该维度支持按照国家进行销售额、订单数、退货额等指标分析。

最终统计：

| 表名                | 说明   |    行数 |
| ----------------- | ---- | ----: |
| `dw.dim_date`     | 时间维度 |   305 |
| `dw.dim_product`  | 商品维度 | 3,938 |
| `dw.dim_customer` | 客户维度 | 4,372 |
| `dw.dim_country`  | 国家维度 |    38 |

## Cube构建

本实验在 PostgreSQL 中基于已建立的星型模型构建销售分析 Cube。Cube 名称为 `OnlineRetailSalesCube`，分析主题为“在线零售销售多维分析”。Cube 以 `fact_sales` 事实表为数据基础，关联 `dim_date`、`dim_product`、`dim_customer` 和 `dim_country` 等维度表，形成统一的多维分析视图。

该 Cube 的事实粒度为`一张发票中的一个商品明细行`。在此粒度下，既可以保留订单明细信息，又可以根据分析需要向上汇总到日期、月份、季度、年份、国家、商品和客户等不同层级。

Cube的主要度量值如下：

| 度量值  | 计算方式                                             | 含义          |
| ---- | ------------------------------------------------ | ----------- |
| 销售金额 | `SUM(sales_amount)`                              | 正常销售产生的总金额  |
| 退货金额 | `SUM(return_amount)`                             | 退货订单对应金额    |
| 净销售额 | `SUM(gross_amount)`                              | 销售额扣除退货后的结果 |
| 销售数量 | `SUM(quantity_sold)`                             | 正常销售商品数量    |
| 退货数量 | `SUM(return_quantity)`                           | 退货商品数量      |
| 订单数  | `COUNT(DISTINCT invoice_no)`                     | 不重复订单数量     |
| 客户数  | `COUNT(DISTINCT customer_id)`                    | 不重复客户数量     |
| 客单价  | `SUM(sales_amount) / COUNT(DISTINCT invoice_no)` | 平均每笔订单销售金额  |
由于 PostgreSQL 不是专门的 MOLAP 引擎，因此本实验采用 SQL 语义层的方式实现 Cube，即通过视图、物化视图和聚合查询将事实表与维度表整合为多维分析结构，并基于该结构完成 OLAP 操作。

![[截屏2026-05-27 10.47.54.png]]

## OLAP操作说明

### Slice 切片分析

分析`United Kingdom`的月度销售情况

```sql
SELECT
    year,
    quarter,
    month,
    ROUND(SUM(sales_amount), 2) AS sales_amount,
    COUNT(DISTINCT invoice_no) AS order_count
FROM olap.v_online_retail_sales_cube
WHERE country = 'United Kingdom'
GROUP BY year, quarter, month
ORDER BY year, month;
```
![[截屏2026-05-27 11.00.06.png]]


### Dice 切块分析

针对`Germany`、`France`和`Eire`三个国家2011年的销售情况进行分析
```sql
SELECT
    country,
    quarter,
    month,
    ROUND(SUM(sales_amount), 2) AS sales_amount,
    COUNT(DISTINCT invoice_no) AS order_count
FROM olap.v_online_retail_sales_cube
WHERE year = 2011
  AND country IN ('Germany', 'France', 'Eire')
GROUP BY country, quarter, month
ORDER BY country, quarter, month;
```

![[截屏2026-05-27 18.53.56.png]]

### Pivot 旋转分析

将国家作为行，2011年个月份作为列，销售额作为值进行旋转分析
```sql
SELECT *
FROM olap.v_country_month_pivot
ORDER BY sales_rank
LIMIT 10;
```

![[截屏2026-05-27 18.56.54.png]]

### Roll-up 上卷分析

使用`PostgreSQL`的 `ROLLUP`实现时间层级的汇总

```sql
SELECT
    year,
    quarter,
    month,
    rollup_level,
    ROUND(sales_amount, 2) AS sales_amount
FROM olap.v_time_rollup_sales
ORDER BY year NULLS LAST, quarter NULLS LAST, month NULLS LAST;
```

![[截屏2026-05-27 18.58.53.png]]
### Drill-down 下钻分析

基于时间维度进行下钻

`Year -> Quarter -> Month`
```sql
SELECT
    year,
    quarter,
    month,
    ROUND(sales_amount, 2) AS sales_amount,
    order_count
FROM olap.v_monthly_sales_trend
ORDER BY year, month;
```

![[截屏2026-05-27 18.55.08.png]]

