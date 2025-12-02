# 动态报表

---
## 一 架构图

![project](https://github.com/struts2execute-dot/Flex-Report/blob/main/img/framework.png)

基础概念

- 维度：数据特征，如用户id,渠道id，总代id，用户类型，游戏id等维表分组字段；
- 指标：通常类型为number，如bet金额，充值金额，提现金额等；
- 复合指标：基础指标做数值运算；
- 业务：可以是一个表/一个试图/一段sql；

---

## 二 算法核心

核心逻辑：不同业务下的不同指标，在相同的维度下，可以通过union all横向组合成报表。

---

## 三 实现流程



