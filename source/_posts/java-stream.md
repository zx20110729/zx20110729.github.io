---
title: Java8 Collectors.toMap的坑
date: 2020-03-31 13:43:42
tags: 
	- Java8
	- Stream
	- toMap
	- Collectors
excerpt: Java Stream 使用
---

# Java Stream 使用
### 1. 排序

	``` java
		// 简单排序 
		List<Long> ids = new ArrayList<>();
	    ids.add(100L);
	    ids.add(50L);
	    ids.add(80L);
	    ids.add(10L);
	    ids.sort(Comparator.comparing(Function.identity()));

		// 复杂排序
		/**
		* class User{
		*  long id;	
		* }
		*/
		List<User> users = new ArrayList<>();
        users.add(new User(100L));
        users.add(new User(50L));
        users.add(new User(80L));
        users.add(new User(10L));
        Ordering<User> ordering = Ordering.from((User a, User b) -> Longs.compare(a.getId(), b.getId()));
        users.sort(ordering);
	```