---
title: IDEA类注释模板
date: 2020-03-25 11:18:44
tags: idea
categories:
	- idea
	- settings
excerpt: IDEA类注释模板
---

### 设置创建类时自动生成注释
> File -> Settings -> Editor -> File and Code Templates
> 在页面上Include按钮下选择 File Header文件。

添加如下代码：
```java 
/** 
 * @PACKAGE_NAME: ${PACKAGE_NAME}
 * @NAME: ${NAME}
 * @USER: ${USER}
 * @DATE: ${DATE}
 * @TIME: ${TIME}
 * @YEAR: ${YEAR}
 * @MONTH: ${MONTH}
 * @MONTH_NAME_SHORT: ${MONTH_NAME_SHORT}
 * @MONTH_NAME_FULL: ${MONTH_NAME_FULL}
 * @DAY: ${DAY}
 * @DAY_NAME_SHORT: ${DAY_NAME_SHORT}
 * @DAY_NAME_FULL: ${DAY_NAME_FULL}
 * @HOUR: ${HOUR}
 * @MINUTE: ${MINUTE}
 * @PROJECT_NAME: ${PROJECT_NAME}
**/
```
效果如下：
``` java
package com.zhou;

/**
 * @PACKAGE_NAME: com.zhou
 * @NAME: Demo
 * @USER: Zhou
 * @DATE: 2020/3/25
 * @TIME: 16:50
 * @YEAR: 2020
 * @MONTH: 3
 * @MONTH_NAME_SHORT: 三月
 * @MONTH_NAME_FULL: 三月
 * @DAY: 25
 * @DAY_NAME_SHORT: 星期三
 * @DAY_NAME_FULL: 星期三
 * @HOUR: 16
 * @MINUTE: 50
 * @PROJECT_NAME: JavaLife
 **/
public class Demo {
}
```