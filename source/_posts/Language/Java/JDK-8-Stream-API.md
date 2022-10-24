---
title: JDK 8 Stream API
date: 2022-10-24 19:57:44
tags:
---


# Stream
stream 是数据渠道，用于操作数据源（集合、数组等）所生成的元素序列。

注意：

Stream 自己不会存储元素。
Stream 不改变源对象。相反，它们回返回一个持有结果的新 Stream。
Stream 操作时延迟进行的。这意味着它们会等到需要结果的时候才执行。


## map

- `List<A>` => `List<B>`

```java
public class FileDTO {
	private String name;
	private Long lastModified;
	private Long size;
	public FileDTO(File file) {
		this.name = file.getName();
		this.lastModified = file.lastModified();
		this.size = file.length();
	}	
}
```
```java
Arrays.stream(files).map(file -> new FileDTO(file)).collect(Collectors.toList());
```
```java
// List<Integer> => List<String>
List<String> stringList = intList.stream().map(String::valueOf).collect(Collectors.toList());
```

## distinct

去重，依据 `Object.equals()` 方法


## sorted

排序


- 自定义比较器

> `Comparator`，Returns a negative integer, zero, or a positive integer as the first argument is less than, equal to, or greater than the second.

```java
List<Map<String, Object>> sortedWorkerLogList = workerLogList.stream().sorted((workerLog1, workerLog2) -> {
    Date createTime1 = (Date) workerLog1.get("create_time");
    Date createTime2 = (Date) workerLog2.get("create_time");
    if (createTime1.after(createTime2)) {
        return 1;
    } else if (createTime1.before(createTime2)) {
        return -1;
    }
    return 0;
}).collect(Collectors.toList());
```

## collect

- 分隔字符串

```java
idList.stream().map(String::valueOf).collect(Collectors.joining(","));
```