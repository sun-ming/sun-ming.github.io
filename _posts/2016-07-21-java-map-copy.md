---
layout: post
title: "Map对象的浅拷贝与深拷贝"
date: 2016-09-29 09:45:00
categories: java 数据结构
tags: java 数据结构
author: Sun-Ming
---

* content
{:toc}

问题：map拷贝时发现数据会变化。
先看例子：
```
public class CopyMap {
    public static void main(String[] args) {
        Map<String,Integer> map = new HashMap<String,Integer>();
        map.put( "key1", 1);
        Map<String,Integer> mapFirst = map;
        System. out.println( mapFirst);
        map.put( "key2", 2);
        System. out.println( mapFirst);
    }
}
```




上面程序的期望输出值是，mapFrist的值均为1，
但是实际上输出结果为：
```
{key1=1}
{key2=2, key1=1}
```


这里是因为map发生了浅拷贝，mapFirst只是复制了map的引用，和map仍使用同一个内存区域，所以，在修改map的时候，mapFirst的值同样会发生变化。

PS：

所谓浅复制：则是只复制对象的引用，两个引用仍然指向同一个对象，在内存中占用同一块内存。被复制对象的所有变量都含有与原来的对象相同的值，而所有的对其他对象的引用仍然指向原来的对象。换言之，浅复制仅仅复制所考虑的对象，而不复制它所引用的对象。

深复制：被复制对象的所有变量都含有与原来的对象相同的值，除去那些引用其他对象的变量。那些引用其他对象的变量将指向被复制过的新对象，而不再是原有的那些被引用的对象。换言之，深复制把要复制的对象所引用的对象都复制了一遍。

如何解决？
使用深拷贝，拷贝整个对象，而非引用
Map中有个方法叫做putAll方法，可以实现深拷贝，如下：

```
public class CopyMap {
    public static void main(String[] args) {
        Map<String,Integer> map = new HashMap<String,Integer>();
        map.put( "key1", 1);
        Map<String,Integer> mapFirst = new HashMap<String,Integer>();
        mapFirst.putAll(map); //深拷贝
        System. out.println(mapFirst);
        map.put( "key2", 2);
        System. out.println(mapFirst);
    }
}
```

如上，输出结果为：

```
{key1=1}
{key1=1}
```

版权声明：本文为博主原创文章，未经博主允许不得转载。