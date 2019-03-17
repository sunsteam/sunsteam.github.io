---
layout:     post
title:      "MyBatis转换对象、枚举插入数据库的处理"
subtitle:   ""
date:       2019-03-13 12:00:00
author:     "Yomii"
catalog:    true
tags:
    - Java
---

## 需求

1. 枚举类型插入数据库时，插入枚举的值而不是名称

```java
enum FlowType implements ValueEnum {

        /**
         * 充值
         */
        RECHARGE(1),
        /**
         * 消费
         */
        CONSUME(2),
        /**
         * 退款
         */
        REFUND(3);

        private int value;
        FlowType(int value) {
            this.value = value;
        }

        @Override
        @JsonValue
        public int getValue() {
            return value;
        }
    }

```

2. 对象类型插入数据库时，自动转换所有属性为 json 或字符串后插入

---

## 默认行为

MyBatis 提供 `BaseTypeHandler` 来处理不同类型的数据对象转换成数据库字段类型的操作。不同类型的处理器继承它并实现读写转换方法，然后通过 xml 或者注解配置处理器来处理未知类型的读写操作。

1. MyBatis 处理枚举类型时，自动插入枚举的名称或索引 (位置)，实际上是通过叫做 `EnumTypeHandler` 和 `EnumOrdinalTypeHandler` 的类完成的，他们是枚举类型的默认处理器。
以下是两个类的序列化和反序列化实现

2. 处理未知类型时，如果不指定 TypeHanlder，会导致编译错误出错

```java
public class EnumTypeHandler<E extends Enum<E>> extends BaseTypeHandler<E> {
  ....
  @Override
  public void setNonNullParameter(PreparedStatement ps, int i, E parameter, JdbcType jdbcType) throws SQLException {
    if (jdbcType == null) {
      ps.setString(i, parameter.name());
    } else {
      ps.setObject(i, parameter.name(), jdbcType.TYPE_CODE); // see r3589
    }
  }

  @Override
  public E getNullableResult(ResultSet rs, String columnName) throws SQLException {
    String s = rs.getString(columnName);
    return s == null ? null : Enum.valueOf(type, s);
  }
  ....
}
```

```java
public class EnumOrdinalTypeHandler<E extends Enum<E>> extends BaseTypeHandler<E> {
....
@Override
  public void setNonNullParameter(PreparedStatement ps, int i, E parameter, JdbcType jdbcType) throws SQLException {
    ps.setInt(i, parameter.ordinal());
  }

  @Override
  public E getNullableResult(ResultSet rs, String columnName) throws SQLException {
    int i = rs.getInt(columnName);
    if (rs.wasNull()) {
      return null;
    } else {
      try {
        return enums[i];
      } catch (Exception ex) {
        throw new IllegalArgumentException("Cannot convert" + i + "to" + type.getSimpleName() + "by ordinal value.", ex);
      }
    }
  }
....
}
```

---


## 解决方案

1. 对于枚举类型，实现 `ValueEnum` 接口的 getValue 方法，获取数值，然后针对 `ValueEnum` 接口写一个 TypeHandler 进行处理, 最后通过注解 `@MappedTypes({ValueEnum.class})` 将这个处理类作为 `ValueEnum` 类型的默认处理器

</br>

```java
/**
 * 值枚举转换处理器
 *
 * @param <E> 枚举类型
 *
 * @author by SunYuXing on 2019-03-01.
 */
@MappedTypes({ValueEnum.class})
public class ValueEnumTypeHandler<E extends Enum<?> & ValueEnum> extends BaseTypeHandler<ValueEnum> {

    private Class<E> type;

    public ValueEnumTypeHandler(Class<E> type) {
        if (type == null) {
            throw new IllegalArgumentException("Type argument cannot be null");
        }
        this.type = type;
    }

    @Override
    public void setNonNullParameter(PreparedStatement ps, int i, ValueEnum parameter, JdbcType jdbcType)
            throws SQLException {
        ps.setInt(i, parameter.getValue());
    }

    @Override
    public E getNullableResult(ResultSet rs, String columnName) throws SQLException {
        int code = rs.getInt(columnName);
        return rs.wasNull() ? null : codeOf(code);
    }

    @Override
    public E getNullableResult(ResultSet rs, int columnIndex) throws SQLException {
        int code = rs.getInt(columnIndex);
        return rs.wasNull() ? null : codeOf(code);
    }

    @Override
    public E getNullableResult(CallableStatement cs, int columnIndex) throws SQLException {
        int code = cs.getInt(columnIndex);
        return cs.wasNull() ? null : codeOf(code);
    }

    private E codeOf(int code){
        try {
            return codeOf(type, code);
        } catch (Exception ex) {
            throw new IllegalArgumentException("Cannot convert" + code + "to" + type.getSimpleName() + "by code value.", ex);
        }
    }

    private static <E extends Enum<?> & ValueEnum> E codeOf(Class<E> enumClass, int code) {
        E[] enumConstants = enumClass.getEnumConstants();
        for (E e : enumConstants) {
            if (e.getValue() == code) {
                return e;
            }
        }
        return null;
    }
}
```

</br>

1. 对于任意对象，转换为 json 储存进数据库，由于反序列化需要指定类型，因此只能匹配到具体类型，比如

</br>

```java
public class OrderData implements MetaEntity, Serializable {
    private static final long serialVersionUID = 1;

    /**
     * 用户名称
     */
    private String userName;

    /**
     * 场馆名称
     */
    private String gymName;

    /**
     * 赠送金额
     */
    private Integer donationAmount;

    /**
     * 排期 id
     */
    private String scheduleId;
}
```

</br>

对应的处理器，实际上在 mapper 中的 `javaType` 为 OrderData，因此只能映射 `OrderData.class` 做处理

</br>

```java
@MappedTypes({OrderData.class})
public class MetaEntityTypeHandler<T extends MetaEntity> extends BaseTypeHandler<T> {

    private static final Logger LOG = LoggerFactory.getLogger(MetaEntityTypeHandler.class);

    private Class<T> clazz;

    public MetaEntityTypeHandler(Class<T> clazz) {
        if (clazz == null) {
            throw new IllegalArgumentException("Type argument cannot be null");
        }
        this.clazz = clazz;
    }

    @Override
    public void setNonNullParameter(PreparedStatement ps, int i, MetaEntity parameter, JdbcType jdbcType) throws SQLException {
        ps.setString(i, JsonUtils.toJsonString(parameter));
    }

    @Override
    public T getNullableResult(ResultSet rs, String columnName) throws SQLException {
        return JsonUtils.parseObject(rs.getString(columnName), clazz);
    }

    @Override
    public T getNullableResult(ResultSet rs, int columnIndex) throws SQLException {
        return JsonUtils.parseObject(rs.getString(columnIndex), clazz);
    }

    @Override
    public T getNullableResult(CallableStatement cs, int columnIndex) throws SQLException {
        return JsonUtils.parseObject(cs.getString(columnIndex), clazz);
    }
}
```

</br>

  **需要在 yml 中配置 `mybatis` 的配置的包地址，否则不会扫描到注解**

```xml
  # mybatis-plus 配置
  mybatis:v
    type-handlers-package: com.comma.fit.productcenter.config
  mybatis-plus:
    mapper-locations: classpath:mapper/**/*.xml
    type-handlers-package: com.comma.fit.productcenter.config
```
