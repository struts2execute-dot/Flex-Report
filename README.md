# 动态报表

---
## 一 架构图

![project](https://github.com/struts2execute-dot/Flex-Report/blob/main/img/framework.png)

基础概念

- 维度：即数据的联合主键字段或者数据标识；
- 基础指标：dws表的某一个数值字段的聚合结果；
- 复合指标：基础指标的上层指标，是通过基础指标进行加减乘除运算以后的结果指标；
- 指标业务：基础指标的数据来源，实际是一个表或者视图或者一段view；
- 数据字典：维度的KEY值映射，用于解析sql中的枚举字段；

---

## 二 算法核心

核心逻辑：不同业务下的不同指标，在相同的维度下，可以通过union all横向组合成报表。

---

## 三 实现流程

### 3-1 定义业务与基础指标

定义 推广业务 promotion_cost_view.yaml

```yaml
select:
  dimensions_list:
    - content: idate
      name: day_time
      inner_format: toUInt32(idate)
      inner_name: idate
    - content: general_no
      name: general_no
    - content: pack_no
      name: pack_no
  index_list:
    - content: sum(cost * {USD_RATE})
      name: cost
      data_type: number
      definition: 消耗
      is_money: true
      agg: sum
      desc: 消耗
    - content: sum(impressions)
      name: impressions
      data_type: number
      definition: 展示量
      is_money: false
      agg: sum
      desc: 展示量
    - content: sum(views)
      name: views
      data_type: number
      definition: 访问量
      is_money: false
      agg: sum
      desc: 访问量
  from: > 
    (
      select
        {SELECT_DIM}
        sum(cost) as cost,
        sum(impressions) as impressions,
        sum(views) as views
      from mg_dynamic_{APP_ID}.d_pack_promotion_record FINAL 
      where 
        toUInt32(idate) BETWEEN {LOCAL_START_DAY} AND {LOCAL_END_DAY}
        {WHERE}
      GROUP BY 
        {GROUP_DIM}
    )
```


