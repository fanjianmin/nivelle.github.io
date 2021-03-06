---
layout: post
title:  "springIOC之BeanFactory"
date:   2017-04-23 00:06:05
categories: spring
tags: spring
excerpt: spring之IOC
author: nivelle
---


* content
{:toc}

---

## IOC 基本原理和思想

整片文章以如下业务为例：

![image](http://7xpuj1.com1.z0.glb.clouddn.com/%E6%B5%8B%E8%AF%95%E7%B1%BB.png)


### ioc 基本概念

定义：全称是Inversion of control, 中文翻译为“控制反转”，别名叫做依赖注入(dependency Injection)

通常情况下，被注入对象会直接依赖于被依赖的对象。（我们常见的直接构造一个对象过来）。但是在IOC 场景中，二者是通过 Ioc Service Provider 来打交道，所有被注入对象和依赖对象现在有其统一管理。Ioc  Service Provider 在这里充当的就是通常说的IOC 容器角色。

---

![image](http://7xpuj1.com1.z0.glb.clouddn.com/ioc.png)

从被注入对象的角度看，与之前直接寻求依赖对象相比，依赖对象方式发生了反转，控制也从被注入对象转到了IOC service provider 哪里。

---

![image](http://7xpuj1.com1.z0.glb.clouddn.com/%E5%BD%A2%E8%B1%A1%E5%9B%BE.png)

被注入对象通知ioc service provider 提供具体服务的方式：

- 构造注入法:被注入对象可以通过其在构造方法中申明依赖对象的参数列表，让外部（通常是ioc容器）知道他需要依赖那些对象。ioc service provider 会检查被注入对象的构造方法，取得他需要的依赖对象列表，进而为其注入相应的对象。**优点**:对象构造完成之后就进入就绪状态，可以立马使用。但构造方法无法继续，对非必须参数处理不够灵活。

- setter注入法:当前对象只要为其依赖对象所对应的属性添加setter 方法，就可以通过setter方法将对应的依赖对象设置到被注入对象中。相对灵活，可以不像构造注入那样，可以在对象先构造完成再注入。** 优点**：setter方法可以被继承，允许设置默认值。
  
- 接口注入:被注入对象想要IOC Service Provider 为其注入依赖对象， 就必须实现某个接口。这个接口提供一个方法，用来为其注入依赖对象。

#### 总结：Ioc是一种可以帮助我们解耦各个业务对象间依赖关系的对象绑定方式。


### IOC Service Provider 基本原理

#### ioc Service Provider的基本职责

- 业务对象的构建管理。IOC 中，业务对象无需关心所依赖对象如何构建如何取得，这部分工作被IOC Service Provider 从客户端对象哪里剥离出来，以免这部分逻辑污染业务对象的实现。

- 业务对象间的依赖绑定。IOC Service Provider 通过结合之前构建和管理的所有业务对象，以及各个业务对象间可以识别的依赖关系，将这些对象所依赖的对象注入绑定，从而保证每个业务对象在使用的时候，可以处于就绪状态。

#### ioc Service Provider工作方式

- 直接编码方式
  在容器启动之前，我们就可以通过程序编码的方式将被注入对象和依赖对象注册到容器当中，并明确确认他们相互之间的依赖注入关系。
  
```
   IoContainer container = ...;
   container.register(FxNewsProvider.class,new FxNewProvider());
   container.register(IFxNewsListener.class,new FxNewListener());
   ...
   
   FxNewsProvider newsProvider = (FxNewsProvider)container.get(FxNewsProvider.class);
   newsProvider.getAndPersistNews();

```
- 配置文件方式
  普通文件，properties文件，xml文件等都可以成为管理依赖注入关系的载体。最常见的还是通过xml管理对象注册和对象间依赖关系。

```
 <bean id ="newsProvider" class="..FxNewsProvider">
      <property name="newsListener">
         <ref bean="djNewListener"/>
      </property>
      <property name="newsPersistener">
         <ref bean="djNewPersistener"/>
      </property>
 </bean>
  <bean id ="djNewListener" class="..impl.DowJonesNewsListener">
 </bean>
   <bean id ="newsPersistener" class="..impl.DowJonesNewsPersistener">
 </bean>

```
读取对象：

```
container.readConfigurationFiles(...);
FxNewsProvider newsProvider = (FxNewsProvider)container.get(FxNewsProvider.class);
newsProvider.getAndPersistNews();

```

- 元数据方式

我们可以直接在类中使用元数据信息来标注各个对象之间的依赖关系，然后由Guice框架根据这些注解所提供的信息将这些对象组装后，交给客户端对象使用。

```
public class FXNewsProvider
{
    private IFXNewsListener newsListener;
    private IFXNewsPersister newPersistener;
    @Inject
    public FXNewsProvider(IFXNewsListener listener,IFXNewsPersister newPersistener)
    {
        this.newsListener = listener;
        this.newPersistener = persistener
    }
    
}

```
通过@Inject ,我们指明需要IOC Service Provider 通过构造方法注入方式，为FXNewsProvider 注入其所依赖的对象。至于余下的依赖相关信息，在guice中由响应的Module来提供。
FXNewsProvider 所使用的Module实现
```
public class NewsBindingModule extends AbstractModule{
    @Override
    protected void configure(){
        bind(IFXNewsListener.class)
        .to(DowJonesNewsListener.class).in(Scopes.SINGLETON);
        bind(IFXNewsListener.class)
        .to(DowJonesNewsPersistener.class).in(Scopes.SINGLETON);
    }
}
```

从guice获取并使用最终绑定完成的FXNewsProvider

```
Injector injector = Guice.createInjector(new NewsBindingModule());
FXNewsProvider newsProvider = injector.getInstance(FXNewsProvider.class);
newsProvider.getAndPersistNews();

```

#### spring提供了两种容器类型
- [ beanFactory]  基础类型ioc  容器，提供完整ioc服务支持。默认采用延迟初始化策略（lazy-load）只有当客户端对象需要访问容器中的某个受管对象的时候，才对该受管对象进行初始化以及依赖注入操作。
- [application ]  在beanFactory的基础上构建，是相对比较高的容器实现，除了拥有beanfactory的所有支持，还提供了其他高级功能。applicationContext所管理的对象，在容器启动之后，默认全部初始化，容器启动时间较BeanFactory长一些。

#### spring的IOC 容器之 BeanFactory

![image](http://7xpuj1.com1.z0.glb.clouddn.com/springIOC%E5%AE%B9%E5%99%A8.png)
分为两部分理解：IOC 和容器（ioc 指的是ioc service provider 功能）此外还包括的功能aop,生命周期管理等。


beanFactory的定义
![image](http://7xpuj1.com1.z0.glb.clouddn.com/beanfactory%E6%8E%A5%E5%8F%A3%E5%AE%9A%E4%B9%89.png)


##### **beanFactory**作为 ioc service provider 的工作方式

1.使用BeanFactory 的xml 配置方式实现业务对象间的依赖

```
<beans>
   <bean id="djNewsProvider" class="..FXNewsProvider">
     <constructor-arg index="0">
       <ref bean="djNewsListener/>
     </construct-arg>
     
     <constructor-arg index="1">
        <ref bean="djNewsPersister/>
     </construct-arg>
     
</beans>

```

2.在beanFactory出现之前，我们在程序的入口main()方法中，通过如下方式实现实例化并调用方法：

```
FXNewsProvider newsProvider = new FXNewsProvider（）;
newsProvider.getAndPersistNews();

```
有beanFactory之后：

```
beanfactory container = new XmlBeanFactory(new classPathResource("文件配置路径"));
或者：beanfactory container = new ClassPathXmlApplicationContext(new classPathResource("文件配置路径"));
或者：beanfactory container = new FileSystemXmlApplicationContext(new classPathResource("文件配置路径"));
FXNewsProvider newsProvider  =(FXNewsProvider)container.getBean("djNewsProvider");
newsProvider.getAndPersistNews();

```

##### beanFactory 实现对象注册和依赖关系绑定的方式

- 硬编码方式(底层方式)

```
public static void main(String[] args)
{
	DefaultListableBeanFactory  beanRegistry = new DefaultListableBeanFactory();
	BeanFactory container = (BeanFactory)bindViaCode(beanRegistry);
	FXNewsProvider newsProvider  =(FXNewsProvider)container.getBean("djNewsProvider");
    newsProvider.getAndPersistNews();
}

public static BeanFactory bindViaCode(DefaultListableBeanFactory registry)
{
    AbstractBeanDefinition newsProvider = new RootBeanDefinition(FXNewsProvider.class,true);
    
    AbstractBeanDefinition newsListener = new RootBeanDefinition(DowJoneNewsListener.class,true);
    
    AbstractBeanDefinition newsPersister = new RootBeanDefinition(DowJoneNewsProvider.class,true);
    //将bean定义注册到容器中
    registry.registerBeanDefinition("djNewsProvider",newsProvider);
    registry.registerBeanDefinition("djListener",newsListener);
    registry.registerBeanDefinition("djPersister",newsPersister);
    //指定依赖关系
    //1.构造方式注入
    ConstructorArgumentValues  argValues =  new ConstructorArgumentValues();
    argValues.addIndexedAregumentValue(0,newsListener);
    argValues.addIndexedAregumentValue(1,newsPersister);
    newProvider.setConstructorArgumentValues(argValues);
    //2.setter方式注入
    MutablePropertyValues propertyValues = new MutablePropertyValues();
    propertyValues.addPropertyValue(new ropertyValue("newsListener",newsListener));
    propertyValues.addPropertyValue(new ropertyValue("newsPersistener",newsPersister));
    newsProvider.setPropertyValues(propertyValues);
    //绑定完成
    return (BeanFactory)registry;
}

```
![image](http://7xpuj1.com1.z0.glb.clouddn.com/QQ%E6%88%AA%E5%9B%BE20170510003635.png)

其中，BeanFactory接口定义如何访问内容管理器的bean方法，各个BeanFactory的具体实现类负责具体Bean的注册以及管理工作。BeanDefinitionRergistry接口定义抽象了bean的注册逻辑。通常，具体的BeanFactory类会实现这个接口来管理Bean的注册。

每个受管对象，在容器中都有一个BeanDefinition的实例与之对应，该实例保存对象的所有必要信息，包括对应的对象的class类型、是否是抽象类、构造方法参数以及其他属性

- 外部配置文件方式（properties和xml）

对于外部配置文件方式 ，ioc有统一的处理方式。通常，根据不同的外部配置文件格式，给出相应的beanDefinitionReader 实现类，由其相应的实现类负责将配置文件内容读出并映射到BeanDefinition，然后将映射后的BeanDefinition注册到BeanDefinitionRegistry,之后beanDefinitionRegistry完成bean的注册和加载。

一般处理方式如下：

```
BeanDefinitionRegistry beanRegistry = 实现类；（DefaultListableBeanFactory）

BeanDefinitionReader beanDefinitionReader = new BeanDefinitionReaderImpl(beanRegistry);

beanDefinitionReader.loadBeanDefinitions("配置文件路径");

```

- 因为properties方式不常用，所以重点关注下xml方式实现方式：

  xml配置采用DTD实现文档格式约束，2.X 引入XSD(xml schema Definition)。

```
 <beans>
   <bean id="djNewsProvider" class="..FXNewsProvider">
     <constructor-arg index="0">
       <ref bean="djNewsListener/>
     </construct-arg>
     
     <constructor-arg index="1">
        <ref bean="djNewsPersister/>
     </construct-arg>
    </bean>
   <bean id ="djNewListener" class="..impl.DowJonesNewsListener">
   </bean>
   <bean id ="newsPersistener" class="..impl.DowJonesNewsPersistener">
   </bean>
     
</beans>

```
接下来做的就是将配置文件加载到beanFactory实现中

```

public static void main(String[] args)
{
	DefaultListableBeanFactory  beanRegistry = new DefaultListableBeanFactory();
	BeanFactory container = (BeanFactory)bindViaXMLFile(beanRegistry);
	FXNewsProvider newsProvider  =(FXNewsProvider)container.getBean("djNewsProvider");
    newsProvider.getAndPersistNews();
}

public static BeanFactory bindViaXmlFile(BeanDefinitionRegistry registry)
{
	XmlBeanDefinitionReader reader = new XmlBeanDefinitionReader(registry);
	reader.loadBeanDefinitions("classpath:../news-config.xml");
	return (BeanFactory)registry;
	//或者
	return new XmlBeanFactory(new ClasspathResource("../news-config.xml"));
}

```

###### 与构造方法注入可以使用<construct-arg>注入配置对应，Spirng 为setter方法注入提供了<property>元素。


- 注解方式

如果通过注解方式为FxNewsProvider注入所需依赖，可以使用@Autowired以及@Component对相关类进行标记。


实现方式：
```
@Component
public class FXNewsProvider()

{
    @AutoWired
    private IFXNewsListener newsListener;
    @AutoWired
    private IFXNewsPersister newPersistener; 
    
    public FXNewsProvider(IFXNewsListener listener,IFXNewsPersister newPersistener)
    {
        this.newsListener = listener;
        this.newPersistener = persistener
    } 

}

@Component
pubcli class DowJonesNewsListener implements IFXNewsListener
{
    
}

@Component
pubcli class DowJonesNewsPersister implements IFXNewsPersister
{
    
}

```
其中 @AutoWired 告知Spring容器需要为当前容器注入那些依赖对象，@Component和classpath-scanning配合告知Spring容器需要实例化那些对象。

```
<context:component-scan base-package ="cn.spring21.project.base-package"/>

```


工作方式：

```

public static void main(String[] args)
{
	ApplicationContext ctx  = new ClassPathXmlApplicationContenxt("配置文件路径");
	FXNewsProvider newsProvider  =(FXNewsProvider)container.getBean("djNewsProvider");
    newsProvider.getAndPersistNews();
}

```


####  工厂方法与FactoryBean

##### 工厂方法

 接口编程强调对象可以通过申明接口来避免对特定接口实现类的过度耦合，但最终必须将申明依赖接口的对象与接口实现类关联起来。否则只是依赖一个不会干活的接口类。
  
 当依赖第三方库时，需要实例化并使用第三方库中的相关类，这时通过工厂方法实现解耦合。


```

public class Foo {
  
  private BarInterface barInterface;
  public Foo()
  {
    //barInterface = BarInterfaceFactory.getInstance();
    //或者
    //barInterface = new BarInterfaceFactory().getInstance();
  }

}

class BarInterface{
  
}


```

- 静态工厂方法

  假设某个第三方库发布了BarInterface ,为了向使用该接口的客户端对象屏蔽以后可能对BarInterface实现类的变动，提供了一个静态工厂方法实现类：

```

public class StaticBarInterfaceFactory {
  
  public static BarInterface getInstance(){
    
    return new BarInterfaceImpl();
  }

}


```

 将该静态工厂方法返回得到实现注入Foo,我们使用以下方式进行配置

 ```
 /<bean id ="foo" class="..Foo">
    <property name ="barInterface">
         <ref bean =" bar"/>
    </property>     
/</bean>

/<bean id ="bar" class="..StaticBarInterfaceFactory" factory-method="getInstance"/>

```

**由此看到**，我们为foo 注入的是调用工厂方法类后返回的实例而不是工厂方法类的实例。工厂方法类的类型和其返回类型没有必然的相同关系。



- 非静态工厂方法

```

  public class NonStaticBarInterfaceFactory {
  
  public  BarInterface getInstance(){
    
    return new BarInterfaceImpl();
  }

}

```

对应的配置文件：

```
 /<bean id ="foo" class="..Foo">
    <property name ="barInterface">
         <ref bean =" bar"/>
    </property>     
/</bean>

/<bean id ="barFactory" class="..NonStaticBarInterfaceFactory"></bean>
/<bean id ="bar" factory-ban="barFactory" factory-method="getInstance"/>


```
注意：现在使用factory-bean属性指定工厂方法所在工厂类实例，而不是通过class属性指定工厂方法所在类的类型。


##### FactoryBean

它是spring容器提供的一种可扩展容器对象实例化逻辑的接口，不要与容器名称BeanFactory相混淆。

当不愿使用xml或者第三方库不能直接注册到Spring容器的时候，可以实现org.springfamework.beans.factory.factoryBean接口

```

  T getObject() throws Exception;
  Class<?> getObjectType();
  boolean isSingleton();

```

使用实例

```
public class NextDayDateBean implements FactoryBean<Object>{

  @Override
  public Object getObject() throws Exception {
    
    return new DateTime().plusDays(1);
  }

  @Override
  public Class<?> getObjectType() {
    // TODO Auto-generated method stub
    return DateTime.class;
  }

  @Override
  public boolean isSingleton() {
    // TODO Auto-generated method stub
    return false;
  }

}


 /<bean id ="NextDayDateDisplayer" class="..NextDayDateDisplayer">
    <property name ="dateOfNextDay">
         <ref bean =" nextDayDate"/>
    </property>     
/</bean>

/<bean id ="nextDayDate" class="..NextDayDateBean"></bean>

NextDayDateDisplayer 定义如下：

public class NextDayDateDisplayer
{
 private DateTime dateOfNextDay;
}

```
以上表明：factoryBean类型的bean定义，通过正常的id引用，容器返回的是FactoryBean所“生产”的对象类型，而非FactoryBean实现本身。

 ---
文章内容出自：《spring揭秘》--王福强
