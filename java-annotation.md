title: Java Annotation Tutorials
date: 2016/4/28 20:45:37
categories: 
tags: Java
---

>注：本文主要翻译自[Lesson: Annotations](https://docs.oracle.com/javase/tutorial/java/annotations/index.html)，学习一下Java注解。

<!--more-->

Annotations（注解）是一种元数据（metadata），用于描述一段程序但对程序的运行不产生直接影响。

注解有一些作用，其中包括：

- 为编译器提供信息——注解可以被编译器用于探测 errors 或者 suppress warnings。
- 编译时或部署时处理——软件处理注解后可生成代码，XML 文件等。
- 运行时处理——一些注解可以用于运行时检查。

## Annotations基础
### 注解的格式
注解最简单的格式如下：

```
@Entity
```
`@`字符告知编译器接下来是一个注解，下面这个例子是一个名为`Override`的注解：

```java
@Override
void mySuperMethod() { ... }
```
注解还可以包含一些带命名或不带命名的元素，每个元素会有相应的 value：

```java
@Author(
   name = "Benjamin Franklin",
   date = "3/27/2003"
)
class MyClass() { ... }
```

或

```java
@SuppressWarnings(value = "unchecked")
void myMethod() { ... }
```
如果只包含一个带命名的 value，那么这个 value 的 name 可以忽略：

```java
@SuppressWarnings("unchecked")
void myMethod() { ... }
```
如果注解里没有元素，那么括号可以被省略，参考前面的`@Override`示例。

对于同一个声明，可以用多个注解：

```java
@Author(name = "Jane Doe")
@EBook
class MyClass { ... }
```

如果注解的类型相同，这种情况被称为重复注解（repeating annotation）：

```java
@Author(name = "Jane Doe")
@Author(name = "John Smith")
class MyClass { ... }
```

重复注解自 Java 8 开始支持，详见 Repeating Annotations 部分。

注解的类型可以是`java.lang`或`java.lang.annotation`包下面定义的类型，比如前面例子中的`Override`和`SuppressWarnings`就是 Java 预置的注解类型。此外还可以自定义注解类型，如前面例子中的`Author`和`Ebook`。

### 注解可以用在什么地方
注解可以用于声明类、域、方法和其他程序的元素，当用于声明时，注解往往自成一行。

从 Java 8 开始，注解还可作为类型（type），如下：

- 创建类的实例：

	```java
	new @Interned MyObject();
	```
- 转型：

	```java
	myString = (@NonNull String) str;
	```
	
- 实现接口：
	```java
	class UnmodifiableList<T> implements
        @Readonly List<@Readonly T> { ... }
	```
	
- 抛异常的声明：
	```java
	void monitorTemperature() throws
        @Critical TemperatureException { ... }
	```
	
这种形式的注解被称为类型注解（type annotation）。

## 声明一个注解类型
在代码中很多注解代替了注释（comments）。

比如，一个开发团队通常在一个类的 body 起始的位置添加一些重要的注释：

```java
public class Generation3List extends Generation2List {

   // Author: John Doe
   // Date: 3/17/2002
   // Current revision: 6
   // Last modified: 4/12/2004
   // By: Jane Doe
   // Reviewers: Alice, Bill, Cindy

   // class code goes here

}
```

如果使用注解的话，首先要定义一个注解类型，语法如下：

```java
@interface ClassPreamble {
   String author();
   String date();
   int currentRevision() default 1;
   String lastModified() default "N/A";
   String lastModifiedBy() default "N/A";
   // Note use of array
   String[] reviewers();
}
```

注解的定义和接口的定义有些像，`interface`前面多了个`@`。

上面这个注解的定义中包括了注解类型和元素声明，其中 default 值是可选的。

定义好注解之后，我们可以像下面这样使用它：

```java
@ClassPreamble (
   author = "John Doe",
   date = "3/17/2002",
   currentRevision = 6,
   lastModified = "4/12/2004",
   lastModifiedBy = "Jane Doe",
   // Note array notation
   reviewers = {"Alice", "Bob", "Cindy"}
)
public class Generation3List extends Generation2List {

// class code goes here

}
```
>Note：为了让 Java Doc 包含注解`@ClassPreamble`的信息，我们需要在定义`@ClassPreamble`时为它再加上注解`@Documented`：

>```java
>// import this to use @Documented
>import java.lang.annotation.*;

>@Documented
>@interface ClassPreamble {

>   // Annotation element definitions
   
>}
>```

## 预置的注解类型
Java API 中预定义了一些注解类型，一些用于编译器，还有一些用于其它注解。

### Java使用的注解类型
`java.lang`中预置的注解有：`@Deprecated`, `@Override`, 和 `@SuppressWarnings`。

**`@Deprecated`**注解表示被标记的元素是deprecated的，不建议再使用的元素。当程序使用了被`@Deprecated`注解的方法、类或域时，编译器会生成一个warning信息。当一个元素是deprecated，在Javadoc中也要使用@deprecated标记，Doc中的*d*小写，注解中是大写的*D*：

```java
 // Javadoc comment follows
    /**
     * @deprecated
     * explanation of why it was deprecated
     */
    @Deprecated
    static void deprecatedMethod() { }
}

```
**`@Override`**注解告诉编译器，该方法是重写自父类的方法：

```java
// mark method as a superclass method
   // that has been overridden
   @Override 
   int overriddenMethod() { }
```
用`@Override`的好处是如果这个方法没用正确重写父类的方法时，编译器会产生一个 error 提示。

**`@SuppressWarnings`**注解提示编译器不用产生指定的 warning 信息，如调用一个 deprecated 方法时，编译器会生成一个 waring 提示，使用@SuppressWarnings注解后，这个 waring 就被抑制了：

```java
// use a deprecated method and tell 
   // compiler not to generate a warning
   @SuppressWarnings("deprecation")
    void useDeprecatedMethod() {
        // deprecation warning
        // - suppressed
        objectOne.deprecatedMethod();
    }
```

**`@SafeVarargs`**用于方法或构造器时，断言可变参数不会产生潜在的危险，由可变参数使用而产生的 warning 会被 suppressed。

**`@FunctionalInterface`**是Java 8引入的，表示函数式接口。

### 注解的注解
用于其它注解的注解，被称为元注解（meta-annotations），下面设计`java.lang.annotation`包里的相关注解。

**`@Retention`**指定了注解的保留方式：

- RetentionPolicy.SOURCE – 被标记的注解只保留在源码级别，会被编译器忽略。
- RetentionPolicy.CLASS – 被标记的注解保留至编译时，会被JVM忽略。
- RetentionPolicy.RUNTIME – 被标记的注解被JVM保留，会用于运行时环境。

**`@Documented`**注解信息会被 Javadoc 工具写入文档。

**`@Target`**起限制作用，表示注解用于描述哪种 Java 元素：

- ElementType.ANNOTATION_TYPE 用于描述注解
- ElementType.CONSTRUCTOR 用于描述构造器
- ElementType.FIELD 用于描述域
- ElementType.LOCAL_VARIABLE 用于描述局部变量
- ElementType.METHOD 用于描述方法的注解
- ElementType.PACKAGE 用于描述包
- ElementType.PARAMETER 用于描述参数
- ElementType.TYPE 用于描述一个类的任何元素

**`@Inherited`**表示该注解是可以被继承的，只能用于声明类。

**`@Repeatable`**在Java 8引入，表示被标记的注解可以被重复用于同一个声明。

----
扩展阅读：

- [Type Annotations and Pluggable Type Systems](https://docs.oracle.com/javase/tutorial/java/annotations/type_annotations.html)
- [Repeating Annotations](https://docs.oracle.com/javase/tutorial/java/annotations/repeating.html)
- [Java中的注解是如何工作的？](http://www.importnew.com/10294.html)


