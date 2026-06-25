241250033 谢浩天
[[Business Intelligence]]
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

## OLAP 多维分析详细结论
### 一、总体经营表现
基于 `olap.v_online_retail_sales_cube` 的汇总结果，Online Retail 数据集整体表现如下：

| 指标 | 数值 |
|---|---:|
| 销售额 | 10,642,110.80 |
| 退货额 | 893,979.73 |
| 净销售额 | 9,748,131.07 |
| 订单数 | 23,796 |
| 整体退货率 | 8.40% |

**结论：整体业务规模较大，但退货对净销售额有明显影响。**

退货金额约占销售额的 `8.40%`。这说明企业虽然销售规模较高，但退货会对最终净销售额造成一定压力，后续应关注退货商品、退货国家和退货月份的分布。

### 二、时间维度分析
#### 季度销售趋势

| 年份 | 季度 | 销售额 | 退货额 | 订单数 |
|---:|---:|---:|---:|---:|
| 2010 | Q4 | 821,452.73 | 74,729.12 | 1,885 |
| 2011 | Q1 | 1,928,572.43 | 191,083.48 | 4,437 |
| 2011 | Q2 | 2,066,812.11 | 162,372.94 | 5,343 |
| 2011 | Q3 | 2,532,352.69 | 131,088.44 | 5,554 |
| 2011 | Q4 | 3,292,920.84 | 334,705.75 | 6,577 |
**结论：销售额呈现明显的年末旺季特征，2011 年 Q4 是销售高峰。**
#### 月度销售变化
销售额较高的月份集中在 2011 年 9 月至 11 月：

| 年份 | 月份 | 销售额 | 退货额 | 订单数 |
|---:|---:|---:|---:|---:|
| 2011 | 9 | 1,056,435.19 | 38,838.51 | 2,170 |
| 2011 | 10 | 1,151,263.73 | 81,895.50 | 2,402 |
| 2011 | 11 | 1,503,866.78 | 47,720.98 | 3,210 |
| 2011 | 12 | 637,790.33 | 205,089.27 | 965 |
**结论 : 2011 年 11 月销售额最高，但 12 月退货压力明显上升。**
2011 年 11 月销售额是全周期最高月份。2011 年 12 月销售额下降明显，但退货额显著上升。

### 三、国家 / 地区维度分析
#### 国家销售贡献

| 排名 | 国家 | 销售额 | 销售占比 | 订单数 |
|---:|---|---:|---:|---:|
| 1 | United Kingdom | 9,001,744.09 | 84.59% | 21,391 |
| 2 | Netherlands | 285,446.34 | 2.68% | 100 |
| 3 | Eire | 283,140.52 | 2.66% | 360 |
| 4 | Germany | 228,678.40 | 2.15% | 603 |
| 5 | France | 209,625.37 | 1.97% | 461 |
| 6 | Australia | 138,453.81 | 1.30% | 69 |
| 7 | Spain | 61,558.56 | 0.58% | 105 |
| 8 | Switzerland | 57,067.60 | 0.54% | 74 |
| 9 | Belgium | 41,196.34 | 0.39% | 119 |
| 10 | Sweden | 38,367.83 | 0.36% | 46 |
**结论 ：业务高度依赖英国市场，市场集中度非常高。而Netherlands、Eire、Germany、France 是较重要的海外市场。**
United Kingdom 的销售额占比达到 `84.59%`，远高于其他国家。这说明该企业主要市场在英国，本土市场对整体业绩具有决定性影响。如果英国市场波动，整体销售额会受到显著影响。
除英国外，Netherlands、Eire、Germany 和 France 销售额相对较高，合计贡献约 `9.46%` 的销售额。这些国家可以作为海外市场扩展和重点客户维护的优先区域。

#### 国家客单价差异
排除订单数少于 20 的国家后，客单价较高的国家如下：

| 国家          |        销售额 | 订单数 |      客单价 |
| ----------- | ---------: | --: | -------: |
| Netherlands | 285,446.34 | 100 | 2,854.46 |
| Australia   | 138,453.81 |  69 | 2,006.58 |
| Japan       |  37,416.37 |  28 | 1,336.30 |
| Norway      |  36,165.44 |  40 |   904.14 |
| Denmark     |  18,955.34 |  21 |   902.64 |
| Sweden      |  38,367.83 |  46 |   834.08 |
| Eire        | 283,140.52 | 360 |   786.50 |
**结论 ：Netherlands 和 Australia 客单价高，具有高价值市场特征。**
### 四、商品维度分析
#### 商品销售额分析
  
|  排名 | 商品编码   | 商品描述                               |        销售额 |  销售占比 |   销售数量 |
| --: | ------ | ---------------------------------- | ---------: | ----: | -----: |
|   1 | DOT    | DOTCOM POSTAGE                     | 206,248.77 | 1.94% |    706 |
|   2 | 22423  | REGENCY CAKESTAND 3 TIER           | 174,156.54 | 1.64% | 13,851 |
|   3 | 23843  | PAPER CRAFT , LITTLE BIRDIE        | 168,469.60 | 1.58% | 80,995 |
|   4 | 85123A | WHITE HANGING HEART T-LIGHT HOLDER | 104,462.75 | 0.98% | 37,641 |
|   5 | 47566  | PARTY BUNTING                      |  99,445.23 | 0.93% | 18,283 |
|   6 | 85099B | JUMBO BAG RED RETROSPOT            |  94,159.81 | 0.88% | 48,371 |
|   7 | 23166  | MEDIUM CERAMIC TOP STORAGE JAR     |  81,700.92 | 0.77% | 78,033 |
|   8 | POST   | POSTAGE                            |  78,101.88 | 0.73% |  3,150 |
|   9 | M      | MANUAL                             |  77,750.27 | 0.73% |  6,984 |
|  10 | 23084  | RABBIT NIGHT LIGHT                 |  66,870.03 | 0.63% | 30,739 |
|  11 | 22086  | PAPER CHAIN KIT 50'S CHRISTMAS     |  64,875.59 | 0.61% | 19,329 |
|  12 | 84879  | ASSORTED COLOUR BIRD ORNAMENT      |  58,927.62 | 0.55% | 36,362 |
**结论 ：商品销售并非完全依赖单一爆款，Top 10 商品销售额占比约 10.82%。**

### 五、客户维度分析

| 客户编号  |        销售额 |  销售占比 | 订单数 |       客单价 |
| ----- | ---------: | ----: | --: | --------: |
| 14646 | 280,206.02 | 2.63% |  76 |  3,686.92 |
| 18102 | 259,657.30 | 2.44% |  62 |  4,188.02 |
| 17450 | 194,390.79 | 1.83% |  55 |  3,534.38 |
| 16446 | 168,472.50 | 1.58% |   3 | 56,157.50 |
| 14911 | 143,711.17 | 1.35% | 248 |    579.48 |
| 12415 | 124,914.53 | 1.17% |  26 |  4,804.41 |
| 14156 | 117,210.08 | 1.10% |  66 |  1,775.91 |
| 17511 |  91,062.38 | 0.86% |  46 |  1,979.62 |
| 16029 |  80,850.84 | 0.76% |  76 |  1,063.83 |
| 12346 |  77,183.60 | 0.73% |   2 | 38,591.80 |

**少数客户贡献明显，Top 10 已知客户贡献约 14.45% 的销售额。**

### 六、退货维度分析
#### 月度退货率
  
|   年份 |  月份 |        销售额 |        退货额 |  退货金额率 |  退货订单率 |
| ---: | --: | ---------: | ---------: | -----: | -----: |
| 2011 |  12 | 637,790.33 | 205,089.27 | 32.16% | 15.13% |
| 2011 |   1 | 689,811.61 | 131,363.05 | 19.04% | 19.32% |
| 2011 |   6 | 760,547.01 |  70,569.78 |  9.28% | 17.67% |
| 2010 |  12 | 821,452.73 |  74,729.12 |  9.10% | 17.29% |
| 2011 |   4 | 536,968.49 |  44,600.65 |  8.31% | 16.15% |
**2011 年 12 月退货金额率异常偏高。**

#### 国家退货率
排除销售额较低的国家后，退货率较高的国家如下：
  
| 国家 | 销售额 | 退货额 | 退货率 | 订单数 |
|---|---:|---:|---:|---:|
| Singapore | 21,279.29 | 12,158.90 | 57.14% | 10 |
| Hong Kong | 15,483.00 | 5,574.76 | 36.01% | 15 |
| Portugal | 33,683.05 | 4,380.08 | 13.00% | 71 |
| Spain | 61,558.56 | 6,802.53 | 11.05% | 105 |
| United Kingdom | 9,001,744.09 | 812,491.79 | 9.03% | 21,391 |
| Eire | 283,140.52 | 20,147.14 | 7.12% | 360 |
