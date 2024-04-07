---
title: foreach-test
date: 2024-04-07 18:56:18
categories:
- Java
---

foreach 中取出的是可迭代对象的引用还是拷贝？

>​	和 C++ 的引用初始化后不能更改（也就是const A* p）不同， Java 中引用可以更改指向的对象（也就是A* p）
>
>​    Java 中对于引用类型，变量名只是对象的一个引用（重新赋值后地址发生改变，不像C/C++中变量标识一个固定地址（也就是对象本身）需要用指针实现引用）
>
>​     引用类型赋值（形参传递）为引用的复制，foreach 中取出的 tmpA 是列表元素（实际是引用）的复制，相当于另创了一个引用
>
>​      这两个引用指向同一个对象，但原引用和新引用是相互独立的，如果修改新引用指向的对象，原引用指向的对象不会改变
>
>​      更加统一的说，基本数据类型和引用数据类型的赋值/传参都是值传递（深拷贝），只不过基本数据类型传递的是值，引用数据类型传递的是引用的值
>
>​      这一点上 Java 和 C++ 是一样的，只不过是 Java 借助引用类型实现了类似指针的功能




```java
import java.util.ArrayList;
import java.util.List;

class A {
    int a;

    A(int a) {
        this.a = a;
    }
}

public class ForEachTest {
    public static void main(String[] args) {
        // foreach 中取出的是可迭代对象的引用还是拷贝？
        List<Integer> list = new ArrayList<>();
        for (int i = 0; i < 5; i++) {
            list.add(i);
        }
        for (int tmp : list) {
            tmp = 10;
        }
        list.forEach(System.out::println);
//        List<A> list = new ArrayList<>();
////        for (int i = 0; i < 1; i++) {
////            list.add(new A(i));
////        }
////        System.out.println(list.get(0));
////        for (A tmpA : list) {
////            System.out.println(tmpA);
////            System.out.println(tmpA.a);
////            tmpA = new A(2);
////            System.out.println(tmpA);
////            System.out.println(tmpA.a);
////        }
////        // 和 C++ 的引用初始化后不能更改（也就是const A* p）不同， Java 中引用可以更改指向的对象（也就是A* p）
////        // Java 中对于引用类型，变量名只是对象的一个引用（重新赋值后地址发生改变，不像C/C++中变量标识一个固定地址（也就是对象本身）需要用指针实现引用）
//          // 引用类型赋值（形参传递）为引用的复制，foreach 中取出的 tmpA 是列表元素（实际是引用）的复制，相当于另创了一个引用
            // 这两个引用指向同一个对象，但原引用和新引用是相互独立的，如果修改新引用指向的对象，原引用指向的对象不会改变

            // 更加统一的说，基本数据类型和引用数据类型的赋值/传参都是值传递（深拷贝），只不过基本数据类型传递的是值，引用数据类型传递的是引用的值
            // 这一点上 Java 和 C++ 是一样的，只不过是 Java 借助引用类型实现了类似指针的功能

////        System.out.println(list.get(0));
////        System.out.println(list.get(0).a);
//        List<StringBuilder> list = new ArrayList<>();
//        list.add(new StringBuilder("a1"));
//        list.add(new StringBuilder("b1"));
//        list.add(new StringBuilder("c1"));
//
//        for (StringBuilder tmp : list) {
//            tmp.reverse();
//        }
//
//        list.forEach(System.out::println);
    }
}

```

