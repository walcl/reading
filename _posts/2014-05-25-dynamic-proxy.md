---
layout: post
title: Spring AOP技术底层实现-动态代理
categories: Java
tag: dynamic_proxy
---

##1. XML文件语法的知识点

1. 对于XML没有提示的话，在Eclipse中搜索XML catalog设置。对于XML文件来说，xsd是规定XML的语法。我们来看一个例子：

```
<?xml version="1.0" encoding="UTF-8"?>

<beans xmlns="http://www.springframework.org/schema/beans"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:context="http://www.springframework.org/schema/context"
	xmlns:aop="http://www.springframework.org/schema/aop" xmlns:mvc="http://www.springframework.org/schema/mvc"
	xmlns:tx="http://www.springframework.org/schema/tx"
	xsi:schemaLocation="http://www.springframework.org/schema/beans
           http://www.springframework.org/schema/beans/spring-beans-3.0.xsd
           http://www.springframework.org/schema/mvc
           http://www.springframework.org/schema/mvc/spring-mvc-3.0.xsd  
           http://www.springframework.org/schema/context
           http://www.springframework.org/schema/context/spring-context-3.0.xsd
           http://www.springframework.org/schema/aop
           http://www.springframework.org/schema/aop/spring-aop-3.0.xsd
           http://www.springframework.org/schema/tx
           http://www.springframework.org/schema/tx/spring-tx-3.0.xsd">

</beans>
```

这里面首先是XML的版本，然后是XML的编码方式（如果有乱码，说明Eclipse的编码不对）。接下来是关键：

> xmlns中的ns是namespace，和C++相同，是代表变量在某个命名空间中的可以使用的语法。xmlns:xsi代表是以csi开头的命名空间的可以使用的语法。xsi:schemaLocation是这些命名空间存在的物理位置。拿第一个为例子，没有前缀的标签的命名空间是http://www.springframework.org/schema/beans，它的物理位置是xsi:schemaLocation的第一对，key为http://www.springframework.org/schema/beans，它的值（也就是实际存放的位置为）http://www.springframework.org/schema/beans/spring-beans-3.0.xsd。这样，如果你使用没有前缀的标签时，XML文件会知道这是一个没有前缀的bean，我需要找个xmlns这个命名空间，然后去找这个文件里面的语法规则。那么，它根据key找到物理地址是那个在网站上的xsd文件，如果你写的语法规则符合xsd文件，就不会报错；反之会出现红色标记。

如果你是在本地的话，一般也是这样配置的，但是在本地的话，我们可以把xsd文件down到本地的目录中，这样查询语法就在本地进行。所以这就是XML catalog的用处。至于设置的内容，只要把上面那个东西搞清楚了，分分钟秒杀之。


##2. 零碎的知识点

1. Spring的注入类型有三种，分别为set注入、构造方法注入、接口注入，但是后两种基本用不到，只需要记住set的原理即可。加入要进行set注入的属性名称为userInfo，那么Spring会调用bean的setUserInfo方法进行注入。这个方法虽然土，但是是业界标准。尽量不要违反。
2. bean的id或者name都可以，但是name可以有特殊字符。**一般均使用id**
3. bean中的默认scope为singleton（单例模式），就是这个属性无论被拿出来几次，都是同一个对象；还有是prototype（原型），每次拿spring会根据这个原型创建一个新的对象，所以拿出来的每个都是全新的、不同的对象
4. 对于自动装配来说，有2种是经常使用的。byName是在bean中找到和属性名称完全相同的bean进行注入；byType是找到和属性类型完全一致的bean进行注入（如果存在多个可能要报错的）。default的使用<beans>中定义的装配方式```default-autowire="byName"```，
5. lazy-init是在bean被get的时候才会初始化
6. @Autowired默认是byType的，如果有多个相同name的bean，就需要使用@Qualifier("name1")来指定使用name1的bean
7. @Resource是j2ee的特性，如果要使用，就需要加入j2ee/common-annotation.jar包才能使用。**默认按照byName注入，不存在则进行byType注入。**当然，也可以使用特定名称，比如```@Resource(name="name1")```
8. @Commopent的加入，使得不需要将将java bean配置再xml中。对于一个bean，我在类前面加上@Component，就会生成一个对象注入到容器中，后面使用的时候直接就可以用了。避免了在xml中配置的麻烦。只需要在xml中加入```context:component-scan base-package="com.sina" />```即可。它会扫描com.sina包下面的所有类，遇到component就会初始化一个对象加入到容器中，当别的类使用@Resource的时候，就会调用对应的组件。**建议每个组件均使用首字母小写的类名作为标识**
9. @Scope就是在bean的配置中改为注解，singleton为单例模式，prototype为原型模式。
10. @PostConstruct = init-method，@PreDestory = destroy-method
