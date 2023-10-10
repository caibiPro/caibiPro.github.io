---
title: Mybatis Plus 实现支持null字段全量更新
author: mingqing
date: 2023-10-09 13:21:00 +0800
categories: [ORM, Mybatis]
tags: [mybatis, sql injection, wrapper]
math: true
mermaid: true
---

## 1. 全局/注解设置更新策略

字段的更新策略包括：

- `FieldStrategy.IGNORED`: 对入参的实体所有字段不做任何判断，原貌更新

- `FieldStrategy.NOT_NULL`: 当字段为空时不会加入到更新的SQL语句中

- `FieldStrategy.NOT_EMPTY`：特定针对字符串类型，在NOT_NULL基础上增加了非空判断
- `FieldStrategy.DEFAULT`:  默认设置，一般只用在注解中。在全局中设置等同与`NOT_NULL`，在注解中设置代表跟随全局设置
- `FieldStrategy.NEVER`: 无论更新字段是否为空都不会加入到SQL语句中

```xml
# 全局
mybatis-plus:
 global-config:
  db-config:
   update-strategy: not_null
   insert-strategy: not_null

# 注解
@TableField(value = "name", updateStrategy=FieldStrategy.IGNORED)
private String name;
```

## 2. 使用 UpdateWrapper 更新

通过`baseMapper`的`UpdateWrapper`将实体类的指定字段设置为`null`。

```java
userMapper.update(user,
    Wrappers.<User>lambdaUpdate()
        .set(User::getId, null)
        .set(User::getName, null)
        .eq(User::getId, user.getId())
);
```

很显然字段较少时手动调用Wrapper还能接受，但是当存在较少字段需要手动设置某些字段，大批量字段按另外一套规则来设置的情况时，上述方法均不具备普遍性。

## 3.  统一管理Wrapper类的更新字段

编写工具类通过静态工厂法生成统一管理更新字段的Wrapper，结合全局更新策略来达到相对定制化的更新模式，具体考虑如下：

- 公共的字段创建时间，更新时间，逻辑删除字段等等排除在Wrapper类，由自动填充功能负责附加SQL语句
- 不需要做NULL值判断的字段加入到Wrapper类中，由Wrapper类设置值
- 除上述以外的字段要么直接忽略，要么会在入参的实体类对象在全局策略下附加到SQL语句中

```java
public class WrappersFactory {

  private final static List<String> ignoreList = new ArrayList<>();

  // 公共字段以及需要保持NULL检测更新的字段
  static {
    ignoreList.add("id");
    ignoreList.add("age");
  }

  // 将entity中的字段划分为两部分：
  // 1）需要保持NULL检测的字段： 即entity中对应字段为空时，不加入SQL全量更新语句中，或者由自动填充功能来附加相应的SET语句 
  // 2）不需要保持NULL检测的字段： 这些字段都会通过Wrapper的set方法来反映到SQL语句中
  public static <T> LambdaUpdateWrapper<T> updateWithNullField(T entity) {
    UpdateWrapper<T> updateWrapper = new UpdateWrapper<>();
    // 获取实体的全部字段 -> 对应的表列名
    List<Field> allFields = TableInfoHelper.getAllFields(entity.getClass());
    // 获取字段值
    MetaObject metaObject = SystemMetaObject.forObject(entity);

    // 遍历所有不在ignoreList中的字段，添加到Wrapper中
    for (Field field : allFields) {
      if (!ignoreList.contains(field.getName())) {
        Object value = metaObject.getValue(field.getName());
        updateWrapper.set(StringUtils.camelToUnderline(field.getName()), value);
      }
    }
    return updateWrapper.lambda();
  }
}
```

使用方法如下：

```java
// 1. user + wrapper
userService.update(user, WrappersFactory.updateWithNullField(user).eq(User::getId, user.getId()));
-----------------------------------------------------------------------------------
==> Preparing: UPDATE user SET name=?, age=?, address=?, name=?,address=? WHERE (id = ?)
==> Parameters: zard1991(String), 31(Integer), 海南(String), zard1991(String), 海南(String), 1710941416926949377(Long)
<== Updates: 1

 // 2. 仅仅靠wrapper
userService.update(WrappersFactory.updateWithNullField(user).eq(User::getId, user.getId()));
-----------------------------------------------------------------------------------
==> Preparing: UPDATE user SET name=?,address=? WHERE (id = ?)
==> Parameters: zard1991(String), null, 1710941416926949377(Long)
<== Updates: 1
```

## 4. 使用注入器完全自定义更新字段

### 扩展AbstractMethod

具体方法怎么写可以参考`BaseMapper`中的已有方法进行更改，主要是定义相应的枚举对象，定义好完整的SQL语句，并把需要判断是否添加的字段通过`%s`预留位置，根据`tableInfo`的情况将相应的值添加。

可以看到这段代码的逻辑中有一行`fieldFill`判断，为`update`或者`insert_update`时不进行`if`包裹。我们能可以利用这个特性。直接将需要的非公共字段全部设置为`FieldFill.UPDATE`即可。

``` java
@Slf4j
public class UpdateWithNull extends AbstractMethod {
  @Override
  public MappedStatement injectMappedStatement(Class<?> mapperClass, Class<?> modelClass, TableInfo tableInfo) {
    // 在update方法的基础上修改
    // UPDATE("update", "根据 whereEntity 条件，更新记录", "<script>\nUPDATE %s %s %s %s\n</script>"),
    SqlMethod sqlMethod = SqlMethod.UPDATE;

    final List<TableFieldInfo> fieldList = tableInfo.getFieldList();
    for (final TableFieldInfo tableFieldInfo : fieldList) {
      final Class<? extends TableFieldInfo> aClass = tableFieldInfo.getClass();
      try {
        final Field fieldFill = aClass.getDeclaredField("fieldFill");
        fieldFill.setAccessible(true);
        fieldFill.set(tableFieldInfo, FieldFill.UPDATE);
      } catch (final NoSuchFieldException e) {
        log.error("获取fieldFill失败", e);
      } catch (final IllegalAccessException e) {
        log.error("设置fieldFill失败", e);
      }
    }

    String sql = String.format(sqlMethod.getSql(), tableInfo.getTableName(),
        sqlSet(true, true, tableInfo, true, ENTITY, ENTITY_DOT),
        sqlWhereEntityWrapper(true, tableInfo));
    SqlSource sqlSource = languageDriver.createSqlSource(configuration, sql, modelClass);
    return addUpdateMappedStatement(mapperClass, modelClass, sqlMethod.getMethod(), sqlSource);
  }
}
```

### 构造继承BaseMapper的自定义Mapper基类

```java
public interface CommonMapper <T> extends BaseMapper<T> {

  int updateWithNull(@Param(Constants.ENTITY) T entity, @Param(Constants.WRAPPER) Wrapper<T> updateWrapper);
}
```

### 将自定义的基类方法添加到注入器的方法列表中

别忘了将自定义的注入类加入到IOC容器中

```java
public class CustomSqlInjector extends AbstractSqlInjector {
  @Override
  public List<AbstractMethod> getMethodList() {
    return Stream.of(
        new Insert(),
        new Delete(),
        new DeleteByMap(),
        new DeleteById(),
        new DeleteBatchByIds(),
        new Update(),
        new UpdateWithNull(),
        new UpdateById(),
        new SelectById(),
        new SelectBatchByIds(),
        new SelectByMap(),
        new SelectOne(),
        new SelectCount(),
        new SelectMaps(),
        new SelectMapsPage(),
        new SelectObjs(),
        new SelectList(),
        new SelectPage()
    ).collect(toList());
  }
}
```
