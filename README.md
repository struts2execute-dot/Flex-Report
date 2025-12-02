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

定义 推广业务DEMO promotion_cost_view.yaml

> [!说明]
> 实际根据自己公司的业务，将数据根据业务提前划分清楚，此处仅定义一个样例

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

字段说明

- dimensions_list：维度集合
- index_list：指标集合

业务关系

- 业务与指标：一对多；
- 业务与维度：多对多；
- 指标与业务：多对一；

---

### 3-2 按三层架构拼接sql

> [!说明]
> 需要对sql和ck有一定了解，即可根据已知条件逆向出算法。

按 demo 中定义的结构，通过算法可得到最终sql。按select层级，从内到外即对应前面说的三层架构。

```sql
SELECT
  tidy.day_time AS day_time,
  toDecimal64(tidy.impressions, 4) AS impressions,
  toDecimal64(tidy.views, 4) AS views,
  toDecimal64(tidy.cost, 4) AS cost,
  toDecimal64(tidy.game_users, 4) AS game_users,
  toDecimal64(tidy.game_rounds, 4) AS game_rounds,
  toDecimal64(tidy.kol_cost, 4) AS kol_cost,
  toDecimal64(tidy.impressions_1000_cost, 4) AS impressions_1000_cost
FROM(
  SELECT 
    res.day_time AS day_time,
    sum(res.impressions) AS impressions,
    sum(res.views) AS views,
    sum(res.cost)*1/10000 AS cost,
    sum(res.game_users) AS game_users,
    sum(res.game_rounds) AS game_rounds,
    sum(res.kol_cost)*1/10000 AS kol_cost,
    ifNull((((sum(res.cost)*1/10000)+(sum(res.kol_cost)*1/10000))/NULLIF(sum(res.impressions), 0)*1000), 0) AS impressions_1000_cost
  FROM (
    SELECT
      idate AS day_time,
      sum(impressions) AS impressions,
      sum(views) AS views,
      sum(cost * 5) AS cost,
      0 AS game_users,
      0 AS game_rounds,
      0 AS kol_cost
    FROM
      (
        select
          toUInt32(idate) as idate,
          sum(cost) as cost,
          sum(impressions) as impressions,
          sum(views) as views
        from d_pack_promotion_record FINAL 
        where 
          toUInt32(idate) BETWEEN 20251202 AND 20251202
          AND general_no IN (0)
        GROUP BY 
          idate
      )
    GROUP BY
      day_time
    UNION ALL
    SELECT
      i_time AS day_time,
      0 AS impressions,
      0 AS views,
      0 AS cost,
      countDistinctIf(user_id,bet_count>0) AS game_users,
      sum(bet_count) AS game_rounds,
      0 AS kol_cost
    FROM
      (
        select 
            --维度
            toYYYYMMDD(toDate(toString(toInt64(`hour_utc` / 100))) + toIntervalHour(`hour_utc` % 100) + toIntervalHour(0)) as i_time,user_id,
            --投注数据
            sum(v.bet) as bet,
            sum(v.bet_crypto) as bet_crypto,
            sum(v.bet_capital) as bet_capital,
            sum(v.bet_bonus) as bet_bonus,
            sum(v.hit) as hit,
            sum(v.hit_crypto) as hit_crypto,
            sum(v.hit_capital) as hit_capital,
            sum(v.hit_bonus) as hit_bonus,
            sum(v.bet_count)  as bet_count,
            sum(-v.win_capital) as win_capital,
            sum(v.hit-v.bet) as user_profit,
            sum(v.bet-v.hit) as platform_profit,
            sum(v.bet_crypto-v.hit_crypto) as platform_profit_crypto
        FROM t_dws_user_game_data as v final
        WHERE
          hour_utc >= 2025120200
          and hour_utc <= 2025120223 
          AND user_type < 100
          AND general_no IN (0)
        GROUP BY
          i_time,user_id
      )
    GROUP BY
      day_time
    UNION ALL
    SELECT
      i_time AS day_time,
      0 AS impressions,
      0 AS views,
      0 AS cost,
      0 AS game_users,
      0 AS game_rounds,
      sum(kol_amount) AS kol_cost
    FROM
      (
        select 
            --维度
            toYYYYMMDD(toDate(toString(toInt64(`hour_utc` / 100))) + toIntervalHour(`hour_utc` % 100) + toIntervalHour(0)) as i_time,
            --投注数据
            sum(v.bet) as bet,
            sum(v.bet_capital) as bet_capital,
            sum(v.bet_bonus) as bet_bonus,
            sum(v.hit) as hit,
            sum(v.hit_capital) as hit_capital,
            sum(v.hit_bonus) as hit_bonus,
            sum(v.bet_count)  as bet_count,
            sum(-v.win_capital) as win_capital,
            sum(v.hit-v.bet) as user_profit,
            sum(v.bet-v.hit) as platform_profit,
            --kol工资
            sum(v.kol_amount) as kol_amount,
            sum(v.kol_amount_crypto) as kol_amount_crypto,
            --赠送数据
            sum(v.total_give) as total_give,
            sum(v.total_commission) as total_commission,
            --账变
            sum(v.amount_sum) as amount_sum,
            sum(v.amount2_sum) as amount2_sum,
            --充值
            sumIf(v.amount_sum,opt_code in (201,203)) as pay_sum
        FROM t_dws_user_transactions_data as v final
        WHERE
          hour_utc >= 2025120200
          and hour_utc <= 2025120223 
          AND user_type < 100
          AND general_no IN (0)
        GROUP BY
          i_time
      )
    GROUP BY
      day_time
  ) AS res
  GROUP BY 
    res.day_time
) AS tidy
WHERE 1=1
```