---
title: 从“赋值判断地狱”到规则驱动设计
comments: true
no_word_count: true 
reward: true 
copyright: true 
categories: 
- 广告
tags:
- 广告
typora-root-url: ../../dongyifeng.github.io
---



# 从“赋值判断地狱”到规则驱动设计

在业务快速演进的系统中，我们经常会遇到这样一类代码：
 <font color=blut>**大量 if 判断 + 重复的对象构建逻辑 + 强耦合的业务条件**</font>。
 短期能跑，长期必炸。

本文通过一个真实的 Java 代码重构案例，分享如何将 “赋值判断地狱”重构为<font color=blut>**规则驱动（Rule-based）的清晰结构，提高代码的可读性、扩展性和可维护性**</font>。



## 一、问题背景：Assignment 构建逻辑高度膨胀

原始代码的目标很简单：

> 根据 <font size=4.5 color=blut>**`AdInfo`**</font> 中不同的定向条件，生成一组<font size=4.5 color=blut>**`Assignment`**</font> 对象。

但实现方式却是经典的「一路 if 到底」。



<font color=blut size=4.8> **重构前代码特点**</font>

- **if-else 数量巨大**
- **每个分支都在重复构建 Assignment**
- **条件判断 + 赋值逻辑强耦合**
- 新增一个定向条件 → 必须改主流程方法
- 可读性差，不利于代码 review



重构前代码：

这种代码在业务初期还能接受，但一旦规则达到 **10+、20+**，就会迅速失控。

```java
public class IndexItemUtils {
    public static Set<Assignment> getKeyList(AdInfo adInfo) {
        Set<Assignment> assignments = new HashSet<>();
        if (adInfo.getType() == DeliveryMedia.PACKAGE.getCode() && StringUtils.isNotBlank(adInfo.getPackage_name())) {
            assignments.add(Assignment.builder()
                    .name("package_name")
                    .values(IndexUtils.convertStrToSet(adInfo.getPackage_name()))
                    .operatorEnum(OperatorEnum.CONTAINS_AT_LEAST_ONE)
                    .includeBase(true)
                    .build());
        }
        if (adInfo.getCity_mode() == DirectionalMode.INCLUDED.getCode() && StringUtils.isNotBlank(adInfo.getCities())) {
            assignments.add(Assignment.builder()
                    .name("cities")
                    .values(IndexUtils.convertStrToSet(adInfo.getCities()))
                    .operatorEnum(OperatorEnum.CONTAINS_AT_LEAST_ONE)
                    .includeBase(true)
                    .build());
        }
        if (adInfo.getCity_mode() == DirectionalMode.EXCLUDED.getCode() && StringUtils.isNotBlank(adInfo.getCities())) {
            assignments.add(Assignment.builder()
                    .name("city_exclude")
                    .values(IndexUtils.convertStrToSet(adInfo.getCities()))
                    .operatorEnum(OperatorEnum.CONTAINS_AT_LEAST_ONE)
                    .includeBase(false)
                    .build());
        }
        if (!adInfo.getOs_type().equals("COMMON") && StringUtils.isNotBlank(adInfo.getOs_type())) {
            assignments.add(Assignment.builder()
                    .name("os_type")
                    .values(IndexUtils.convertStrToSet(adInfo.getOs_type()))
                    .operatorEnum(OperatorEnum.EQUALITY)
                    .includeBase(true)
                    .build());
        }
        if (StringUtils.isNotBlank(adInfo.getOs_version())) {
            //1=大于，2=大于等于，3=小于，4=小于等于，5等于
            assignments.add(Assignment.builder()
                    .name("os_version")
                    .values(IndexUtils.convertStrToSet(adInfo.getOs_version()))
                    .operatorEnum(OperatorEnum.VERSIONING)
                    .includeBase(true)
                    .versionComparison(VersionComparison.fromCode(adInfo.getOs_version_condition()))
                    .build());
        }
        if (adInfo.getNetwork_mode() == DirectionalMode.INCLUDED.getCode() && StringUtils.isNotBlank(adInfo.getNetwork_type())) {
            assignments.add(Assignment.builder()
                    .name("network_type")
                    .values(IndexUtils.convertStrToSet(adInfo.getNetwork_type()))
                    .operatorEnum(OperatorEnum.CONTAINS_AT_LEAST_ONE)
                    .includeBase(true)
                    .build());
        }
        if (adInfo.getBrand_mode() == DirectionalMode.INCLUDED.getCode() && StringUtils.isNotBlank(adInfo.getBrand_include())) {
            assignments.add(Assignment.builder()
                    .name("brand_include")
                    .values(IndexUtils.convertStrToSet(adInfo.getBrand_include()))
                    .operatorEnum(OperatorEnum.CONTAINS_AT_LEAST_ONE)
                    .includeBase(true)
                    .build());
        }
        if (adInfo.getCrowd_mode() == DirectionalMode.INCLUDED.getCode() && StringUtils.isNotBlank(adInfo.getCrowd_include())) {
            assignments.add(Assignment.builder()
                    .name("crowd_include")
                    .values(IndexUtils.convertStrToSet(adInfo.getCrowd_include()))
                    .operatorEnum(OperatorEnum.CONTAINS_AT_LEAST_ONE)
                    .includeBase(true)
                    .build());
        }
        if (adInfo.getCrowd_mode() == DirectionalMode.INCLUDED.getCode() && StringUtils.isNotBlank(adInfo.getCrowd_exclude())) {
            assignments.add(Assignment.builder()
                    .name("crowd_exclude")
                    .values(IndexUtils.convertStrToSet(adInfo.getCrowd_exclude()))
                    .operatorEnum(OperatorEnum.CONTAINS_AT_LEAST_ONE)
                    .includeBase(false)
                    .build());
        }
        if (StringUtils.isNotBlank(adInfo.getInstall_pkg_include())) {
            assignments.add(Assignment.builder()
                    .name("install_pkg_include")
                    .values(IndexUtils.convertStrToSet(adInfo.getInstall_pkg_include()))
                    .operatorEnum(OperatorEnum.CONTAINS_AT_LEAST_ONE)
                    .includeBase(true)
                    .build());
        }
        if (StringUtils.isNotBlank(adInfo.getInstall_pkg_exclude())) {
            assignments.add(Assignment.builder()
                    .name("install_pkg_exclude")
                    .values(IndexUtils.convertStrToSet(adInfo.getInstall_pkg_exclude()))
                    .operatorEnum(OperatorEnum.CONTAINS_AT_LEAST_ONE)
                    .includeBase(false)
                    .build());
        }
        if (adInfo.getType() == DeliveryMedia.CHANNEL_PACKAGE.getCode() && StringUtils.isNotBlank(adInfo.getChannel()) && StringUtils.isNotBlank(adInfo.getChannel_package())) {
            Set<String> chanelSet = IndexUtils.convertStrToSet(adInfo.getChannel());
            Set<String> packageSet = IndexUtils.convertStrToSet(adInfo.getChannel_package());
            if (!CollectionUtils.isEmpty(chanelSet) && !CollectionUtils.isEmpty(packageSet)) {
                assignments.add(Assignment.builder()
                        .name("channel")
                        .values(chanelSet)
                        .operatorEnum(OperatorEnum.CONTAINS_AT_LEAST_ONE)
                        .includeBase(true)
                        .build());
                assignments.add(Assignment.builder()
                        .name("package_name")
                        .values(packageSet)
                        .operatorEnum(OperatorEnum.CONTAINS_AT_LEAST_ONE)
                        .includeBase(true)
                        .build());
            }
        }
        return assignments.stream()
                .filter(assignment -> !CollectionUtils.isEmpty(assignment.getValues()))
                .collect(Collectors.toSet());
    }
}
```



## 二、重构目标

在动手重构前，我们先明确目标：

✅ 消除大量 if 判断
 ✅ 消除重复的 Assignment 构建代码
 ✅ 将「判断逻辑」与「构建逻辑」解耦
 ✅ 新增规则时，不修改主流程
 ✅ 让代码**像配置一样可读**



## 三、核心思路：规则抽象（Rule Abstraction）

<font color=blut size=4.8> **抽象出「规则」这个概念**</font>

我们发现，每一个 if 分支本质上都在回答 4 个问题：

1. **字段名是什么？**
2. **原始值是什么？**
3. **在什么条件下生效？**
4. **如何构建 Assignment？**



于是，引入一个统一的规则模型：

每一个规则描述「**在什么条件下，用什么方式，生成一个 Assignment**」。`AssignmentRule` 将判断和构建逻辑完全封装

```java
public class AssignmentRule {
    private final String name;
    private final String rawValue;         // 原始字符串，空则无效
    private final Supplier<Boolean> condition;
    private final OperatorEnum operatorEnum;
    private final boolean includeBase;

    public AssignmentRule(AssignmentFieldEnum assignmentFieldEnum, String rawValue,
                          Supplier<Boolean> condition,
                          OperatorEnum operatorEnum,
                          boolean includeBase) {
        this.name = assignmentFieldEnum.getFieldName();
        this.rawValue = rawValue;
        this.condition = condition;
        this.operatorEnum = operatorEnum;
        this.includeBase = includeBase;
    }

    public boolean isValid() {
        return StringUtils.isNotBlank(rawValue) && condition.get();
    }

    public Assignment build() {
        Set<String> valueSet = IndexUtils.convertStrToSet(rawValue);
        if (CollectionUtils.isEmpty(valueSet)) {
            return null;
        }
        return Assignment.builder()
                .name(name)
                .values(valueSet)
                .operatorEnum(operatorEnum)
                .includeBase(includeBase)
                .build();
    }
}
```



 <font color=blut size=4.8>**关键设计点**</font>

- **Supplier<Boolean> condition**
  - 将 if 判断「函数化」
  - 让规则具备自解释能力
- **统一 build 逻辑**
  - 所有 Assignment 构建方式一致
- **天然支持扩展**
  - 新增规则 ≈ 新增一行配置



## 四、重构后的设计

<font color=blut size=4.8>**主流程：规则列表 + 流式处理**</font>

重构后，<font color=blut size=4.5>**`getKeyList`**</font> 的核心逻辑非常清晰：

```java
public class IndexItemUtils {

    public static Set<Assignment> getKeyList(AdInfo adInfo) {
        List<AssignmentRule> rules = Arrays.asList(
                new AssignmentRule(AssignmentFieldEnum.CHANNEL, adInfo.getChannel(),
                        () -> true,
                        OperatorEnum.CONTAINS_AT_LEAST_ONE, true),

                new AssignmentRule(AssignmentFieldEnum.PACKAGE_NAME, adInfo.getPackage_name(),
                        () -> adInfo.getType() == DeliveryMedia.INCLUDE_PACKAGE.getCode(),
                        OperatorEnum.CONTAINS_AT_LEAST_ONE, true),

                new AssignmentRule(AssignmentFieldEnum.PACKAGE_NAME, adInfo.getPackage_name(),
                        () -> adInfo.getType() == DeliveryMedia.EXCLUDE_PACKAGE.getCode(),
                        OperatorEnum.CONTAINS_AT_LEAST_ONE, false),

                new AssignmentRule(AssignmentFieldEnum.PACKAGE_NAME, adInfo.getChannel_package(),
                        () -> adInfo.getType() == DeliveryMedia.INCLUDE_CHANNEL_PACKAGE.getCode(),
                        OperatorEnum.CONTAINS_AT_LEAST_ONE, true),

                new AssignmentRule(AssignmentFieldEnum.PACKAGE_NAME, adInfo.getChannel_package(),
                        () -> adInfo.getType() == DeliveryMedia.EXCLUDE_CHANNEL_PACKAGE.getCode(),
                        OperatorEnum.CONTAINS_AT_LEAST_ONE, false),

                new AssignmentRule(AssignmentFieldEnum.CITIES, adInfo.getCities(),
                        () -> adInfo.getCity_mode() == DirectionalMode.INCLUDED.getCode(),
                        OperatorEnum.CONTAINS_AT_LEAST_ONE, true),

                new AssignmentRule(AssignmentFieldEnum.CITY_EXCLUDE, adInfo.getCities(),
                        () -> adInfo.getCity_mode() == DirectionalMode.EXCLUDED.getCode(),
                        OperatorEnum.CONTAINS_AT_LEAST_ONE, false),

                new AssignmentRule(AssignmentFieldEnum.OS_TYPE, adInfo.getOs_type(),
                        () -> !Objects.equals(adInfo.getOs_type(), "COMMON"),
                        OperatorEnum.EQUALITY, true),

                new AssignmentRule(AssignmentFieldEnum.NETWORK_TYPE, adInfo.getNetwork_type(),
                        () -> adInfo.getNetwork_mode() == DirectionalMode.INCLUDED.getCode(),
                        OperatorEnum.CONTAINS_AT_LEAST_ONE, true),

                new AssignmentRule(AssignmentFieldEnum.BRAND_INCLUDE, adInfo.getBrand_include(),
                        () -> adInfo.getBrand_mode() == DirectionalMode.INCLUDED.getCode(),
                        OperatorEnum.CONTAINS_AT_LEAST_ONE, true),

                new AssignmentRule(AssignmentFieldEnum.CROWD_INCLUDE, adInfo.getCrowd_include(),
                        () -> adInfo.getCrowd_mode() == DirectionalMode.INCLUDED.getCode(),
                        OperatorEnum.CONTAINS_AT_LEAST_ONE, true),

                new AssignmentRule(AssignmentFieldEnum.CROWD_EXCLUDE, adInfo.getCrowd_exclude(),
                        () -> adInfo.getCrowd_mode() == DirectionalMode.EXCLUDED.getCode(),
                        OperatorEnum.CONTAINS_AT_LEAST_ONE, false),

                new AssignmentRule(AssignmentFieldEnum.INSTALL_PKG_INCLUDE, adInfo.getInstall_pkg_include(),
                        () -> true, OperatorEnum.CONTAINS_AT_LEAST_ONE, true),

                new AssignmentRule(AssignmentFieldEnum.INSTALL_PKG_EXCLUDE, adInfo.getInstall_pkg_exclude(),
                        () -> true, OperatorEnum.CONTAINS_AT_LEAST_ONE, false)
        );

        Set<Assignment> assignments = new HashSet<>();
        handleOsVersion(adInfo, assignments);

        rules.stream()
                .filter(AssignmentRule::isValid)
                .map(AssignmentRule::build)
                .filter(Objects::nonNull)
                .forEach(assignments::add);
        return assignments;
    }
  
  
}
```

<font color=blut>**主流程不再关心任何业务细节，只做三件事：**</font>

- 规则是否有效
- 构建 Assignment
- 收集结果



## 五、重构收益分析

📈 <font color=blut>**可读性**</font>

- 从 200+ 行 if 判断
- 变成一眼可扫的规则列表

🔧 <font color=blut>**可维护性**</font>

- 修改规则 → 只改规则
- 不再破坏主流程

🚀 <font color=blut>**扩展性**</font>

- 新增定向字段？
  **不用再 copy 一整段 if**

🧪 <font color=blut>**可测试性**</font>

- <font size=4.5 color=blut>**`AssignmentRule`**</font> 可单测
- 条件判断和构建逻辑都能独立验证



## 六、适用场景总结

这种「规则驱动重构」非常适合：

- 广告 / 推荐 / 风控等<font color=blut>**规则密集型系统**</font>
- 有大量「条件 + 构建对象」逻辑的代码
- 经常新增、调整业务判断的模块



## 七、结语

好的重构，不是炫技，而是<font color=blut>**让代码更像业务本身**</font>。

当你发现：

> “我只是想加一个条件，却要改 5 个地方”

那基本就说明：<font color=blut>**是时候把 if-else 抽象成规则了。**</font>

希望这个案例能对你在实际工程中的重构决策有所启发。
