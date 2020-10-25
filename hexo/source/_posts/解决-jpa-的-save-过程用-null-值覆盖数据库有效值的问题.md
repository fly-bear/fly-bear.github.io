---
title: 解决 jpa 的 save 过程用 null 值覆盖数据库有效值的问题
tags:
  - Java
  - jpa
  - postgreSQL
originContent: ''
categories: []
toc: true
date: 2019-07-19 16:20:12
---

## 问题背景

最近在项目中使用 jpa 操作数据库的时候，遇到了一个问题。

我所使用的数据库是 postgreSQL，通过定义实体类添加`@Table`注解指定表，定义接口继承`CrudRepository`类，将实体类作为参数传入，即可调用默认的 save 方法，jpa 会根据主键将实体进行插入或更新操作。

但是，如果用于更新的实体中含有未赋值的属性，即值为 null 的情况时，数据库中对应字段的值会被 null 覆盖。

我先是在 Google 查询了解决方案，有网友提出可以在实体类加上`@DynamicUpdate`和`@DynamicInsert`两个注解，序列化时即可忽略为 null 的值。我尝试了这种方法，启动单元测试进行了一次更新操作，却发现数据库的数据依然被 null 覆盖。

为了寻找原因，我查看了这两个注解的源码，它们是在 hebernate-core 里定义的。使用 idea 的全局搜索找到了注解的实现代码,还没来得及看具体实现，却发现它被加上了`@Deprecated`注解。虽然理论上对使用是无影响的，但既然作者不推荐了，还是放弃使用吧。

## 自己动手
实在不想在苦苦寻找方法了，决定自己造个轮子，实现 update 的功能。逻辑非常简单，每次 save 之前先从数据库查找出对应数据存入实体A，将含有更新数据的实体B中的非空属性赋值给实体A，最后用实体 A 存回数据库。具体代码如下：

```
	/**
     * 防止 jpa 将为 null 的属性更新（因为注解DynamicUpdate无效）
     *
     * @param entity 包含需要更新字段的实体
     * @param oldEntity 从数据库获取的旧实体
     * @return 最终往数据库插入的实体
     */
     
    public static <T> T updateEntity(T entity,T oldEntity){
        Class clazz = entity.getClass();
        Field[] fields = clazz.getDeclaredFields();
        for (Field field : fields){
            try {
                PropertyDescriptor pd = new PropertyDescriptor(field.getName(), clazz);
                Method readMethod = pd.getReadMethod();
                Method writeMethod= pd.getWriteMethod();
                if (readMethod.invoke(entity)!=null){   //不为null 的覆盖
                    writeMethod.invoke(oldEntity,readMethod.invoke(entity));
                }
            }catch (Exception e){
                log.error(e.getMessage());
            }
        }
        return oldEntity;
    }
```

利用反射+泛型写了个通用方法，将新旧实体作为参数传入，反射取出新实体中非 null 的值赋给旧实体，最后将旧实体返回。将方法封装成工具类方便调用。

在代码中，每次更新数据库前调用一下工具，用返回的实体作为 save 对象，经测试，成功解决 null 覆盖问题。

## 现成工具类

实际上，通过寻找，我确实发现了现成的工具。

大家都知道 `BeanUtil` 类的 `copyProperties` 方法可以拷贝两个实体的同名变量，而通过控制传入参数，就可以实现只拷贝非空属性。

`BeanUtil` 中有个 `getNullPropertyNames` 方法：

```
	public static String[] getNullPropertyNames (Object source) {
		final BeanWrapper src = new BeanWrapperImpl(source);
		java.beans.PropertyDescriptor[] pds = src.getPropertyDescriptors();

		Set<String> emptyNames = new HashSet<String>();
		for(java.beans.PropertyDescriptor pd : pds) {
			Object srcValue = src.getPropertyValue(pd.getName());
			if (srcValue == null) emptyNames.add(pd.getName());
		}
		String[] result = new String[emptyNames.size()];
		return emptyNames.toArray(result);
	}
```

此方法返回实体的所有非 null 字段，将其作为参数传入 `copyProperties` ，即可实现指定复制。代码如下：

```
	public void update(ClassA source, Integer id ){
		ClassA target = classDao.findById(id);  
		BeanUtils.copyProperties(source,target,getNullPropertyNames(source)); 
		classDao.update(target );
}
```