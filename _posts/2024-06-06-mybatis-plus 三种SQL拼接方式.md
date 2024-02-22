---
layout:     post
title:      "mybatis-plus 三种SQL拼接方式"
subtitle:   ""
date:       2024-06-06
author:     "hcy"
header-img: "img/gene-head-img.jpg"
tags:
- java
- mybatis-plus
typora-root-url: ..
---


# mybatis-plus 三种SQL拼接方式
    
mybatis-plus有三种传参方式，最终效果就是拼接sql参数分别如下。

sql的三种拼接方式

- 标准模式，使用类似于 new QueryWrapper<>().eq("request_no","aaa")
- lambda模式，类似于 new QueryWrapper<TbClearingCostDelay>().lambda().eq(TbClearingCostDelay::getRequestNo, "")
- 实体类模式 new QueryWrapper<>(new TbClearingCostDelay());

模式1和模式2没有本质区别，主要是要取得字段名称和字段值，生成sql片段。  
lambda方式使用了java序列化的特殊方法的到的字段名 (可以搜索下获取使用序列化接口获取lambda的方法名), 而模式1直接传递的字段名。  
这两种种方式内部会使用 MergeSegments 对象存储设置的字段和值，MergeSegments 对象内部会拼接成sql片段


模式3直接存储实体类在wrapper中，而启动时,框架会将所有的 mybatis 接口对应的方式生成与之对应的 xml 脚本，脚本类似于下面这样。  
其中ew对象就是wrapper对象，启动时遍历所有字段帮我们把类似原生写法的xml生成好。  
```xml
<script>
DELETE FROM tb_clearing_cost_delay 
<if test="ew != null">
    <!-- 实体类 -->
    <where>
    <if test="ew.entity != null">
        <if test="ew.entity['requestNo'] != null"> AND request_no=#{ew.entity.requestNo}</if>
        <if test="ew.entity['clearingType'] != null"> AND clearing_type=#{ew.entity.clearingType}</if>
        <if test="ew.entity['failTime'] != null"> AND fail_time=#{ew.entity.failTime}</if>
        <if test="ew.entity['requestTime'] != null"> AND request_time=#{ew.entity.requestTime}</if>
        <if test="ew.entity['bizIdentify'] != null"> AND biz_identify=#{ew.entity.bizIdentify}</if>
        <if test="ew.entity['productCode'] != null"> AND product_code=#{ew.entity.productCode}</if>
        <if test="ew.entity['transStep'] != null"> AND trans_step=#{ew.entity.transStep}</if>
        <if test="ew.entity['fluxAction'] != null"> AND flux_action=#{ew.entity.fluxAction}</if>
        <if test="ew.entity['bizExtend'] != null"> AND biz_extend=#{ew.entity.bizExtend}</if>
        <if test="ew.entity['clearingDetails'] != null"> AND clearing_details=#{ew.entity.clearingDetails}</if>
        <if test="ew.entity['clearingStatus'] != null"> AND clearing_status=#{ew.entity.clearingStatus}</if>
        <if test="ew.entity['remark'] != null"> AND remark=#{ew.entity.remark}</if>
        <if test="ew.entity['lastValidTime'] != null"> AND last_valid_time=#{ew.entity.lastValidTime}</if>
        <if test="ew.entity['utcCreateTime'] != null"> AND utc_create_time=#{ew.entity.utcCreateTime}</if>
        <if test="ew.entity['utcModifyTime'] != null"> AND utc_modify_time=#{ew.entity.utcModifyTime}</if>
    </if>
    <!-- 如果前面有任何条件，则 -->
    <if test="ew.sqlSegment != null and ew.sqlSegment != '' and ew.nonEmptyOfWhere">
        <if test="ew.nonEmptyOfEntity and ew.nonEmptyOfNormal"> AND</if>
         ${ew.sqlSegment}
     </if>
    </where>
    <!-- 没有实体、没有lambda、没有正常条件的情况下，以segment为准 -->
    <if test="ew.sqlSegment != null and ew.sqlSegment != '' and ew.emptyOfWhere">
        ${ew.sqlSegment}
    </if>
</if> 
    <!--sql备注 -->
<choose>
    <when test="ew != null and ew.sqlComment != null">
    ${ew.sqlComment}
    </when>
</choose>
</script>
```


从上面xml可以看出，entity写法和标准写法是相互不关心的，两种可以共存，两种写法操作同一个字段也不会自动去重，而是条件使用and连接。  
同时会对字段做null判断但不会做blank判断
