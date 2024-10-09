---
title: "Java 注解及原理" #标题
date: 2023-03-19T01:11:22+08:00 #创建时间
lastmod: 2023-03-19T01:11:22+08:00 #更新时间
author: ["River"] #作者
keywords: #关键词
    - java
    - 注解
categories: #类别
- 
tags: #标签
    - java
    - 注解
description: "" #描述
weight: # 输入1可以顶置文章，用来给文章展示排序，不填就默认按时间排序
slug: ""
draft: false # 是否为草稿
comments: true #是否展示评论
showToc: true # 显示目录
TocOpen: true # 自动展开目录
hidemeta: false # 是否隐藏文章的元信息，如发布日期、作者等
disableShare: true # 底部不显示分享栏
showbreadcrumbs: true #顶部显示当前路径
cover:
    image: "" #图片路径：posts/tech/文章1/picture.png
    zoom: # 图片大小，例如填写 50% 表示原图像的一半大小
    caption: "" #图片底部描述
    alt: ""
    relative: false
    
reward: true # 打赏
---

# Java 注解及原理

## 一、注解基础

注解是插入到源代码中使用其他工具可以对其进行处理的标签，也是附加在代码中的一些元信息，起到说明、配置的功能，语法为`@Annotation(value=xxx)`。这些工具可以在源码层次上进行操作，或者可以处理编译器在其中放置了注解的.class文件。

特点：

- java语言的类、方法、变量、参数和包都可以被注解标注。
- 注解不会改变程序的编译方式。Java编译器对于包含注解和不包含注解的代码会生成相同的虚拟机指令。当然在编译器生成.class文件时，注解可以被嵌入字节码中，而jvm也可以保留注解的内容，在运行时获取注解标注的内容信息。
- 从JVM的角度看，注解本身对代码逻辑没有任何影响，如何使用注解完全由工具决定。

注解的分类：

- 由编译器使用的注解：这类注解不会被编译进入`.class`文件，它们在编译后就被编译器扔掉了。比如`@Override`。`SOURCE`类型的注解主要由编译器使用，因此我们一般只使用，不编写。
- 由工具处理`.class`文件使用的注解：比如有些工具会在加载class的时候，对class做动态修改，实现一些特殊的功能。这类注解会被编译进入`.class`文件，但加载结束后并不会存在于内存中。这类注解只被一些底层库使用，一般我们不必自己处理。`CLASS`类型的注解主要由底层工具库使用，涉及到class的加载，一般我们很少用到。
- 在程序运行期能够读取的注解，它们在加载后一直存在于JVM中，这也是最常用的注解。`RUNTIME`类型的注解不但要使用，还经常需要编写。如，一个配置了`@PostConstruct`的方法会在调用构造方法后自动被调用（这是Java代码读取该注解实现的功能，JVM并不会识别该注解）

## 二、定义注解

### 2.1 注解定义语法

Java语言使用`@interface`语法来定义注解（`Annotation`）

```java
public @interface Report {
    int type() default 0;
    String level() default "info";
    String value() default "";
}
```

定义一个注解时，还可以定义配置参数。配置参数可以包括：

- 所有基本类型；
- String；
- 枚举类型；
- 基本类型、String、Class以及枚举的数组。

注解的参数类似无参数方法，可以用`default`设定一个默认值（强烈推荐），缺少某个配置参数时将使用默认值。

**最常用的参数应当命名为`value`，对此参数赋值，可以只写常量，相当于省略了value参数。**

### 2.2 元注解

元注解是是用于修饰注解的注解，定义注解时使用，也就是和关键字@interface配合使用，一般用来定义注解的作用目标，保留策略等。

| 元注解名称    | 功能描述                                                     |
| ------------- | ------------------------------------------------------------ |
| `@Retention`  | 标识这个注释解怎么保存，是只在代码中，还是编入类文件中，或者是在运行时可以通过反射访问 |
| `@Documented` | 标识这些注解是否包含在用户文档中                             |
| `@Target`     | 标识这个注解的作用范围                                       |
| `@Inherited`  | 标识注解可被继承类获取                                       |
| `@Repeatable` | 标识某注解可以在同一个声明上使用多次                         |

- `@Retention`：指定注解信息保留阶段，有如下三种枚举选择。只能选其一

  ```java
  public enum RetentionPolicy {
      /** 注解将被编译器丢弃，生成的class不包含注解信息 */
      SOURCE,
      /** 注解在class文件中可用，但会被JVM丢弃;当注解未定义Retention值时，默认值是CLASS */
      CLASS,
      /** 注解信息在运行期(JVM)保留（.class也有），可以通过反射机制读取注解的信息,
        * 操作方法看AnnotatedElement(所有被注释类的父类) */
      RUNTIME
  }
  ```

- `@Documented`：作用是告诉JavaDoc工具，当前注解本身也要显示在Java Doc中(不常用)

- `@Target`：指定注解作用范围，可指定多个

  ```java
  public enum ElementType {
      /** 适用范围：类、接口、注解类型，枚举类型enum */
      TYPE,
      /** 作用于类属性 (includes enum constants) */
      FIELD,
      /** 作用于方法 */
      METHOD,
      /** 作用于参数声明 */
      PARAMETER,
      /** 作用于构造函数声明 */
      CONSTRUCTOR,
      /** 作用于局部变量声明 */
      LOCAL_VARIABLE,
      /** 作用于注解声明 */
      ANNOTATION_TYPE,
      /** 作用于包声明 */
      PACKAGE,
      /** 作用于类型参数（泛型参数）声明 */
      TYPE_PARAMETER,
      /** 作用于使用类型的任意语句(不包括class) */
      TYPE_USE
  }
  ```

- @Inherited：表示**当前注解会被注解类的子类继承**。即在子类Class<T>通过getAnnotations()可获取父类被@Inherited修饰的注解。**而注解本身是不支持继承**。

  ```java
  @Inherited
  @Retention( value = RetentionPolicy.RUNTIME)
  @Target(value = ElementType.TYPE)
  public @interface ATest {  }
  ----被ATest注解的父类PTest----
  @ATest
  public class PTest{ }

  ---Main是PTest的子类----
  public class Main extends PTest {
      public static void main(String[] args){
          Annotation an = Main.class.getAnnotations()[0];
            //Main可以拿到父类的注解ATest，因为ATest被元注解@Inherited修饰
          System.out.println(an);
      }
  }
  ---result--
  @com.ATest()
  ```

- @Repeatable：表明自定义的注解可以在同一个位置重复使用。在没有该注解前，是无法在同一个类型上使用相同的注解多次。

  ```java
    //Java8前无法重复使用注解
    @FilterPath("/test/v2")
    @FilterPath("/test/v1")
    public class Test {}
  ```

## 三、处理注解/注解原理

因为注解定义后也是一种`class`，所有的注解都继承自`java.lang.annotation.Annotation`，因此，读取注解，需要使用反射API。

Java提供的使用反射API读取`Annotation`的方法包括：

- 判断`Class`、`Field`、`Method`或`Constructor`是否存在某个注解：

  - `Class.isAnnotationPresent(AnnotationClass)`

  - `Field.isAnnotationPresent(AnnotationClass)`

  - `Method.isAnnotationPresent(AnnotationClass)`

  - `Constructor.isAnnotationPresent(AnnotationClass)`

    ```java
    // 判断@Report是否存在于Person类:
    Person.class.isAnnotationPresent(Report.class);
    ```

- 使用反射API读取Annotation：

  - `Class.getAnnotation(AnnotationClass)`

  - `Field.getAnnotation(AnnotationClass)`

  - `Method.getAnnotation(AnnotationClass)`

  - `Constructor.getAnnotation(AnnotationClass)`

    ```java
    // 获取Person定义的@Report注解:
    Report report = Person.class.getAnnotation(Report.class);
    int type = report.type();
    String level = report.level();
    ```

读取方法、字段和构造方法的`Annotation`和Class类似。

```java
// 方法一
Class cls = Person.class;
if (cls.isAnnotationPresent(Report.class)) {
    Report report = cls.getAnnotation(Report.class);
    ...
}

// 方法二
Class cls = Person.class;
Report report = cls.getAnnotation(Report.class);
if (report != null) {
   ...
}
```

但要读取方法参数的`Annotation`就比较麻烦一点，因为方法参数本身可以看成一个数组，而每个参数又可以定义多个注解，所以，一次获取方法参数的所有注解就必须用一个二维数组来表示。例如，对于以下方法定义的注解：

```java
public void hello(@NotNull @Range(max=5) String name, @NotNull String prefix) { }
```

要读取方法参数的注解，我们先用反射获取`Method`实例，然后读取方法参数的所有注解：

```java
// 获取Method实例:
Method m = ...
// 获取所有参数的Annotation:
Annotation[][] annos = m.getParameterAnnotations();
// 第一个参数（索引为0）的所有Annotation:
Annotation[] annosOfName = annos[0];
for (Annotation anno : annosOfName) {
    if (anno instanceof Range) { // @Range注解
        Range r = (Range) anno;
    }
    if (anno instanceof NotNull) { // @NotNull注解
        NotNull n = (NotNull) anno;
    }
}
```

注解如何使用，完全由程序自己决定。例如，JUnit是一个测试框架，它会自动运行所有标记为`@Test`的方法。

我们来看一个`@Range`注解，我们希望用它来定义一个`String`字段的规则：字段长度满足`@Range`的参数定义：

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.FIELD)
public @interface Range {
    int min() default 0;
    int max() default 255;
}
```

在某个JavaBean中，我们可以使用该注解：

```java
public class Person {
    @Range(min=1, max=20)
    public String name;

    @Range(max=10)
    public String city;
}
```

但是，定义了注解，本身对程序逻辑没有任何影响。我们必须自己编写代码来使用注解。这里，我们编写一个`Person`实例的检查方法，它可以检查`Person`实例的`String`字段长度是否满足`@Range`的定义：

```java
void check(Person person) throws IllegalArgumentException, ReflectiveOperationException {
    // 遍历所有Field:
    for (Field field : person.getClass().getFields()) {
        // 获取Field定义的@Range:
        Range range = field.getAnnotation(Range.class);
        // 如果@Range存在:
        if (range != null) {
            // 获取Field的值:
            Object value = field.get(person);
            // 如果值是String:
            if (value instanceof String s) {
                // 判断值是否满足@Range的min/max:
                if (s.length() < range.min() || s.length() > range.max()) {
                    throw new IllegalArgumentException("Invalid field: " + field.getName());
                }
            }
        }
    }
}
```

这样一来，我们通过`@Range`注解，配合`check()`方法，就可以完成`Person`实例的检查。注意检查逻辑完全是我们自己编写的，JVM不会自动给注解添加任何额外的逻辑。
