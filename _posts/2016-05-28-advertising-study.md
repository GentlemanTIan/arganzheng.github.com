---
title: 互联网广告系统学习笔记
layout: post
tags: [广告]
category: 技术
catalog: true
---


### 广告变现方式

### 计费方式

| 结算方式    |   适用场景 | 
| :-------- | :--------:| 
| CPT  | 品牌广告 |  
| CPM     |   有受众选择需求的品牌广告 |  
| CPC      |    竞价广告 | 
| CPS/CPA/ROI      |    效果类广告联盟/DSP | 


### 一些指标

| 指标    |   含义 | 备注 | 
| :-------- | :--------:| :--------:|
| cpm  | 每千次展检索收费 | 检索端的核心KPI |  
| ctr  | 点击率 | VS 广告相关性 |
| acp  | 平均点击价格 | 高价广告 |
| pvr  | 出广告的检索PV比率 | 流量的商业属性，即PV中被用于变现的比例 |
| asn  | 平均展示条数 | 即：如果展现了推广广告，每次平均展现多少条。pvr * asn 红线 |
| arpu  | 户均消费 | 影响它的因素包括：点击流量大小、关键词的相关性、同一关键词的竞争度，以及客户在搜索引擎广告上的预算上限等。 |


**广告变现的收入公式**

```
Revenue = PV * PVR * ASN * CTR2 * ACP
```


### 触发算法(Keyword Targeting)

* QT: Query Targeting。根据用户当前搜索的query词来触发出query相关的广告。
	* 匹配模式：检索词与广告主拍卖词匹配程度
		* 精确
		* 高级短语
			* 精确包含
			* 切词包含
		* 宽泛
* UT: User Targeting。根据用户的行为数据，产出以userId为key的广告数据。
	* lookalike
	* CF
	* retargeting
	* 用户画像
* GT: Geo Targeting。根据用户的当前位置、家、工作地点、常驻点等位置信息，推荐周边相关的广告，特别适合于O2O类型广告。以geohash为key。
* KGT: 根据知识图谱（关系）进行触发。
* ...


### 排序计费

* 收入与体验的权衡
* 拍卖机制：竞价类广告基本以GSP及其变种为主
* 计费与排序:
	* 位置: 大图，自然结果中间，左边栏，下方，etc.
	* 位次: 相同展现样式队列内部的排序
	* 费用: 预估点击价格
	* 质量: 广告与Query的相关性
* 各种Q值预估：
	* 预估query和广告的相关性
	* 预估点击率
	* 预估广告点击满意度
	* 一滑到底Q值，用于判断滑屏时是否要出现下方广告
	* 预估展现时上下文环境对广告点击率的影响
	* ...


### 广告系统

* 商户平台
	* 用户管理
	* 账户管理（充值）
	* 物料管理（广告和推广计划）
	* 数据报表
* 触发子系统
* 索引子系统
* 预算控制
* 实验和抽样平台
* 反作弊（流式数据处理）
* 模型预估(各种Q值模型计算)




