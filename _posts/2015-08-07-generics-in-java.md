---
layout:     post
title:      "Java 泛型理解"
subtitle:   ""
date:       2015-08-07 12:00:00
author:     "Yomii"
catalog:    true
tags:
    - Java
---


泛型是Java 1.5 以后添加的功能，可以在类或方法上指定其需要的参数或返回值类型。Java原本不支持泛型，因此使用了擦除机制作为折中。

## 类的类型

Java将类的类型封装为接口Type， 包含ParameterizedType，GenericArrayType，TypeVariable和WildcardType四种类型的接口和Class这个直接子类。

其中，只有Class和ParameterizedType是明确类型的。TypeVariable和WildcardType是泛型类型， GenericArrayType两者皆有可能。

- `Class` 普通的类，如String
- `ParameterizedType` 类名带参数类型的类， 如 `ArrayList<String>`
- `TypeVariable` 可变类型, 如 `List<T>` 中的 T，在实际使用中传递给方法时，获取的就是TypeVariable
- `WildcardType` 通配符类型，包含上限或下限两种,如`List<? extend AbstractList>`和`List<? super ArrayList>`，不设置上限或者下限的话可以写成`List<?>`，是`List<? extend Object>`的简写

## 泛型的作用

### 1\. 增加数据结构的安全性

由于Java是强类型语言，而类似List的数据容器可以储存任何类型的对象，在实际调用时容易出现类型转换异常，泛型的加入规范了容器中应该保存的数据类型，减少的错误发生。

```java
//泛型限制仅对对象的引用有效，即等式的左边部分
ArrayList<String> stringList1 = new ArrayList<String>();
//等式右边只是在堆空间中申请内存,因此可以简写为如下
ArrayList<String> stringList2 = new ArrayList<>();
//反过来引用不使用泛型限定的情况下, 等式右边的泛型不会起到任何作用
//容器依然能存入任何对象
ArrayList stringList3 = new ArrayList<String>();
```

### 2\. 获取更精确的返回值和参数

有时调用类的方法中需要使用一些变量的父类或者接口中的通用方法，但根据传入对象的不同，也可能用到他们的子类特性，使用泛型就避免了将对象强转。如下例子:

```java
public static <T extends Comparable> T get(T t1,T t2) { //添加类型限定  
        if(t1.compareTo(t2)>=0);  
        return t1;  
    }
```

### 3\. 更利于抽象

当我们抽取一些行为到接口或者抽取一些特性到基类的时候，类中基本都会包含或者返回对象的引用，而这些对象可能有一些可配置状态或者自有实现，虽然与本类无关，但调用者希望明确这些对象是什么类型，这时就需要泛型。比如Retrofit中的这个接口,就是对类型的灵活使用

```java
package retrofit2;

import java.lang.annotation.Annotation;
import java.lang.reflect.ParameterizedType;
import java.lang.reflect.Type;

/**
 * Adapts a {@link Call} into the type of {@code T}. Instances are created by {@linkplain Factory a
 * factory} which is {@linkplain Retrofit.Builder#addCallAdapterFactory(Factory) installed} into
 * the {@link Retrofit} instance.
 */
public interface CallAdapter<T> {
  /**
   * Returns the value type that this adapter uses when converting the HTTP response body to a Java
   * object. For example, the response type for {@code Call<Repo>} is {@code Repo}. This type
   * is used to prepare the {@code call} passed to {@code #adapt}.
   * <p>
   * Note: This is typically not the same type as the {@code returnType} provided to this call
   * adapter's factory.
   */
  Type responseType();

  /**
   * Returns an instance of {@code T} which delegates to {@code call}.
   * <p>
   * For example, given an instance for a hypothetical utility, {@code Async}, this instance would
   * return a new {@code Async<R>} which invoked {@code call} when run.
   * <pre>{@code
   * @Override
   * public <R> Async<R> adapt(final Call<R> call) {
   *   return Async.create(new Callable<Response<R>>() {
   *     @Override
   *     public Response<R> call() throws Exception {
   *       return call.execute();
   *     }
   *   });
   * }
   * }</pre>
   */
  <R> T adapt(Call<R> call);

  /**
   * Creates {@link CallAdapter} instances based on the return type of {@linkplain
   * Retrofit#create(Class) the service interface} methods.
   */
  abstract class Factory {
    /**
     * Returns a call adapter for interface methods that return {@code returnType}, or null if it
     * cannot be handled by this factory.
     */
    public abstract CallAdapter<?> get(Type returnType, Annotation[] annotations,
        Retrofit retrofit);

    /**
     * Extract the upper bound of the generic parameter at {@code index} from {@code type}. For
     * example, index 1 of {@code Map<String, ? extends Runnable>} returns {@code Runnable}.
     */
    protected static Type getParameterUpperBound(int index, ParameterizedType type) {
      return Utils.getParameterUpperBound(index, type);
    }

    /**
     * Extract the raw class type from {@code type}. For example, the type representing
     * {@code List<? extends Runnable>} returns {@code List.class}.
     */
    protected static Class<?> getRawType(Type type) {
      return Types.getRawType(type);
    }
  }
}
```

### 泛型擦除机制

Java为了向前兼容不支持泛型的版本，会在编译检查过后将泛型擦除，因此，在编译完成后，参数类型转化为了原始类型，如果类型变量有限定，那么原始类型就用第一个边界的类型变量来替换。

如果Pair这样声明`public class Pair<T extends Serializable&Comparable>` ，那么原始类型就用Serializable替换，而编译器在必要的时要向Comparable插入强制类型转换。为了提高效率，应该将标签（tagging）接口（即没有方法的接口）放在边界限定列表的末尾。

然后在调用的地方，字节码中检查了申明类型并自动转换

拿一个例子来说明

```
public class Test {  
public static void main(String[] args) {  
ArrayList<Date> list=new ArrayList<Date>();  
list.add(new Date());  
Date myDate=list.get(0);  
}
```

```
public static void main(java.lang.String[]);  
Code:  
0: new #16 // class java/util/ArrayList  
3: dup  
4: invokespecial #18 // Method java/util/ArrayList."<init  
:()V  
7: astore_1  
8: aload_1  
9: new #19 // class java/util/Date  
12: dup  
13: invokespecial #21 // Method java/util/Date."<init>":()  

16: invokevirtual #22 // Method java/util/ArrayList.add:(L  
va/lang/Object;)Z  
19: pop  
20: aload_1  
21: iconst_0  
22: invokevirtual #26 // Method java/util/ArrayList.get:(I  
java/lang/Object;  
25: checkcast #19 // class java/util/Date  
28: astore_2  
29: return
```

看第22 ，它调用的是ArrayList.get()方法，方法返回值是Object，说明类型擦除了。然后第25，它做了一个checkcast操作，即检查类型#19， 在在上面找#19引用的类型， `9: new #19` 他是一个Date类型，即做Date类型的强转。所以不是在get方法里强转的，是在你调用的地方强转的。

## 泛型序列化的问题

在封装网络请求库时，将json数据格式反序列化为对象，我们的json转换工具比如Gson、fastjson通常需要该对象的Class对象，但如果将TypeVariable传入则会变为它擦除后的原始类型或者限定类型，转化后的结果会丢失信息。

针对这种情况，有以下3种解决方案。

1. **将具体类型传入并保存为变量**，比如Gson中的TypeToken和fastjson中的TypeReference就是保存转化的详细类型信息的包装类。

    缺点： 需要手动传入，json转化的行为无法进行抽取

2. **通过内部类+`getGenericSuperclass`方法获取父类ParameterizedType对象，然后调用`getActualTypeArguments`获取明确参数类型，规避了泛型擦除问题。** 注意：必须以子类或者内部类形式调用getGenericSuperclass方法，因为java没有提供获取自身ParameterizedType的方式，因此只能这样写。

    缺点： 类型无法向内传递，只能从最外层赋值泛型的地方获取，因为向内传递的是TypeVariable

3. **参考Retrofit的Service定义机制，从方法的Method对象中获取方法返回值类型对象，并保存为变量。需要json转换时将它包装并传入。**

    算不上缺点的缺点：获取数据的方法需要单独定义。这种方式可以抽取Json转换行为，单独定义Service，只提供参数并获取结果，也符合 view-dispatch-loader的分层原则。

以上三种解决方法可根据实际需求灵活使用，用起来最方便就好。
