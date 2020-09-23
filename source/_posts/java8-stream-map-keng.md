---
title: Java8 Collectors.toMap的坑
date: 2020-03-31 13:43:42
tags: 
	- Java8
categories:
    - java
    - stream
excerpt: Collectors.toMap的Duplicate Key异常
---

### 问题描述
在使用Collectors.toMap方法时，如果存在相同的key值时，会出现Duplicate Key异常。
示例代码如下:
```java
public class StreamDemo {
    public static void main(String[] args) {
        List<Integer> list = Lists.newArrayList(1, 2, 3, 4, 5, 6, 7, 8, 9, 1);
        Map<Integer, Integer> map = list.stream().collect(Collectors.toMap(Function.identity(), Function.identity()));
        System.out.println(map.toString());
    }
}
```
会出现如下异常信息:
```shell
Exception in thread "main" java.lang.IllegalStateException: Duplicate key 1
	at java.util.stream.Collectors.lambda$throwingMerger$0(Collectors.java:133)
	at java.util.HashMap.merge(HashMap.java:1254)
	at java.util.stream.Collectors.lambda$toMap$58(Collectors.java:1320)
	at java.util.stream.ReduceOps$3ReducingSink.accept(ReduceOps.java:169)
	at java.util.ArrayList$ArrayListSpliterator.forEachRemaining(ArrayList.java:1382)
	at java.util.stream.AbstractPipeline.copyInto(AbstractPipeline.java:482)
	at java.util.stream.AbstractPipeline.wrapAndCopyInto(AbstractPipeline.java:472)
	at java.util.stream.ReduceOps$ReduceOp.evaluateSequential(ReduceOps.java:708)
	at java.util.stream.AbstractPipeline.evaluate(AbstractPipeline.java:234)
	at java.util.stream.ReferencePipeline.collect(ReferencePipeline.java:499)
	at com.zhou.guide.StreamDemo.filter(StreamDemo.java:21)
	at com.zhou.guide.StreamDemo.main(StreamDemo.java:16)
```
### 解决方法
```java
Map<Integer, Integer> map = list.stream().collect(Collectors.toMap(Function.identity(), Function.identity(), (oldValue, newValue) -> newValue));
```
这样就做到了使用新的value替换原有value。

### 问题原因:
Collectors源码
```java
public static <T, K, U> Collector<T, ?, Map<K,U>> toMap(Function<? super T, ? extends K> keyMapper, Function<? super T, ? extends U> valueMapper) {
    return toMap(keyMapper, valueMapper, throwingMerger(), HashMap::new);
}

 private static <T> BinaryOperator<T> throwingMerger() {
        return (u,v) -> { throw new IllegalStateException(String.format("Duplicate key %s", u)); };
    }
```
在Collectors的toMap方法调用链中，最后在HashMap的**merge**方法中会调用到throwingMerger方法。
代码如下：
```java 
		if (old != null) {
            V v;
            if (old.value != null)
                v = remappingFunction.apply(old.value, value);
            else
                v = value;
            if (v != null) {
                old.value = v;
                afterNodeAccess(old);
            }
            else
                removeNode(hash, key, null, false, true);
            return v;
        }
```
如果传入的key已经存在，就会调用传入的方法。此处调用的是throwingMerger方法，抛出Duplicate key异常。
