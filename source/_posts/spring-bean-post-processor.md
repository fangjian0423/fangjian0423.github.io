title: Spring内部的BeanPostProcessor接口总结
date: 2017-06-20 19:30:30
tags:
- spring
- java
- springboot
categories: springboot

----------------

Spring内部提供了一个BeanPostProcessor接口，这个接口的作用在于对于新构造的实例可以做一些自定义的修改。比如如何构造、属性值的修改、构造器的选择等等。

只要我们实现了这个接口，便可以对构造的bean进行自定义的修改。

<!--more-->

BeanPostProcessor接口还有一些子接口的定义：

![](http://7x2wh6.com1.z0.glb.clouddn.com/bean-post-processor.png)

## InstantiationAwareBeanPostProcessor

InstantiationAwareBeanPostProcessor接口继承自BeanPostProcessor接口。多出了3个方法：


```java
// postProcessBeforeInstantiation方法的作用在目标对象被实例化之前调用的方法，可以返回目标实例的一个代理用来代替目标实例
// beanClass参数表示目标对象的类型，beanName是目标实例在Spring容器中的name
// 返回值类型是Object，如果返回的是非null对象，接下来除了postProcessAfterInitialization方法会被执行以外，其它bean构造的那些方法都不再执行。否则那些过程以及postProcessAfterInitialization方法都会执行
Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) throws BeansException;

// postProcessAfterInstantiation方法的作用在目标对象被实例化之后并且在属性值被populate之前调用
// bean参数是目标实例(这个时候目标对象已经被实例化但是该实例的属性还没有被设置)，beanName是目标实例在Spring容器中的name
// 返回值是boolean类型，如果返回true，目标实例内部的返回值会被populate，否则populate这个过程会被忽视
boolean postProcessAfterInstantiation(Object bean, String beanName) throws BeansException;

// postProcessPropertyValues方法的作用在属性中被设置到目标实例之前调用，可以修改属性的设置
// pvs参数表示参数属性值(从BeanDefinition中获取)，pds代表参数的描述信息(比如参数名，类型等描述信息)，bean参数是目标实例，beanName是目标实例在Spring容器中的name
// 返回值是PropertyValues，可以使用一个全新的PropertyValues代替原先的PropertyValues用来覆盖属性设置或者直接在参数pvs上修改。如果返回值是null，那么会忽略属性设置这个过程(所有属性不论使用什么注解，最后都是null)
PropertyValues postProcessPropertyValues(
    PropertyValues pvs, PropertyDescriptor[] pds, Object bean, String beanName)
    throws BeansException;
```

总结：

1. InstantiationAwareBeanPostProcessor接口继承BeanPostProcessor接口，它内部提供了3个方法，再加上BeanPostProcessor接口内部的2个方法，所以实现这个接口需要实现5个方法。InstantiationAwareBeanPostProcessor接口的主要作用在于目标对象的实例化过程中需要处理的事情，包括实例化对象的前后过程以及实例的属性设置
2. postProcessBeforeInstantiation方法是最先执行的方法，它在目标对象实例化之前调用，该方法的返回值类型是Object，我们可以返回任何类型的值。由于这个时候目标对象还未实例化，所以这个返回值可以用来代替原本该生成的目标对象的实例(比如代理对象)。如果该方法的返回值代替原本该生成的目标对象，后续只有postProcessAfterInitialization方法会调用，其它方法不再调用；否则按照正常的流程走
3. postProcessAfterInstantiation方法在目标对象实例化之后调用，这个时候对象已经被实例化，但是该实例的属性还未被设置，都是null。如果该方法返回false，会忽略属性值的设置；如果返回true，会按照正常流程设置属性值
4. postProcessPropertyValues方法对属性值进行修改(这个时候属性值还未被设置，但是我们可以修改原本该设置进去的属性值)。如果postProcessAfterInstantiation方法返回false，该方法不会被调用。可以在该方法内对属性值进行修改
5. 父接口BeanPostProcessor的2个方法postProcessBeforeInitialization和postProcessAfterInitialization都是在目标对象被实例化之后，并且属性也被设置之后调用的
6. Instantiation表示实例化，Initialization表示初始化。实例化的意思在对象还未生成，初始化的意思在对象已经生成

## SmartInstantiationAwareBeanPostProcessor

SmartInstantiationAwareBeanPostProcessor接口继承InstantiationAwareBeanPostProcessor接口。多出了3个方法：

```java
// 预测Bean的类型，返回第一个预测成功的Class类型，如果不能预测返回null
Class<?> predictBeanType(Class<?> beanClass, String beanName) throws BeansException;

// 选择合适的构造器，比如目标对象有多个构造器，在这里可以进行一些定制化，选择合适的构造器
// beanClass参数表示目标实例的类型，beanName是目标实例在Spring容器中的name
// 返回值是个构造器数组，如果返回null，会执行下一个PostProcessor的determineCandidateConstructors方法；否则选取该PostProcessor选择的构造器
Constructor<?>[] determineCandidateConstructors(Class<?> beanClass, String beanName) throws BeansException;

// 获得提前暴露的bean引用。主要用于解决循环引用的问题
// 只有单例对象才会调用此方法
Object getEarlyBeanReference(Object bean, String beanName) throws BeansException;
```

总结：

1. SmartInstantiationAwareBeanPostProcessor接口继承InstantiationAwareBeanPostProcessor接口，它内部提供了3个方法，再加上父接口的5个方法，所以实现这个接口需要实现8个方法。SmartInstantiationAwareBeanPostProcessor接口的主要作用也是在于目标对象的实例化过程中需要处理的事情。它是InstantiationAwareBeanPostProcessor接口的一个扩展。主要在Spring框架内部使用
2. predictBeanType方法用于预测Bean的类型，返回第一个预测成功的Class类型，如果不能预测返回null。主要在于BeanDefinition无法确定Bean类型的时候调用该方法来确定类型
3. determineCandidateConstructors方法用于选择合适的构造器，比如类有多个构造器，可以实现这个方法选择合适的构造器并用于实例化对象。该方法在postProcessBeforeInstantiation方法和postProcessAfterInstantiation方法之间调用，如果postProcessBeforeInstantiation方法返回了一个新的实例代替了原本该生成的实例，那么该方法会被忽略
4. getEarlyBeanReference主要用于解决循环引用问题。比如ReferenceA实例内部有ReferenceB的引用，ReferenceB实例内部有ReferenceA的引用。首先先实例化ReferenceA，实例化完成之后提前把这个bean暴露在ObjectFactory中，然后populate属性，这个时候发现需要ReferenceB。然后去实例化ReferenceB，在实例化ReferenceB的时候它需要ReferenceA的实例才能继续，这个时候就会去ObjectFactory中找出了ReferenceA实例，ReferenceB顺利实例化。ReferenceB实例化之后，ReferenceA的populate属性过程也成功完成，注入了ReferenceB实例。提前把这个bean暴露在ObjectFactory中，这个ObjectFactory获取的实例就是通过getEarlyBeanReference方法得到的

## BeanPostProcessor

BeanPostProcessor接口是最顶层的接口，接口定义：

```java
public interface BeanPostProcessor {
    // 初始化之前的操作
    Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException;
    // 初始化之后的操作
    Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException;
}
```

总结：

1. postProcessBeforeInitialization是指bean在初始化之前需要调用的方法
2. postProcessAfterInitialization是指bean在初始化之后需要调用的方法
3. postProcessBeforeInitialization和postProcessAfterInitialization方法被调用的时候。这个时候bean已经被实例化，并且所有该注入的属性都已经被注入，是一个完整的bean
4. 这2个方法的返回值可以是原先生成的实例bean，或者使用wrapper包装这个实例

## DestructionAwareBeanPostProcessor

DestructionAwareBeanPostProcessor接口继承BeanPostProcessor接口。多出了1个方法：

```java
void postProcessBeforeDestruction(Object bean, String beanName) throws BeansException;
```

该方法是bean在Spring在容器中被销毁之前调用

## MergedBeanDefinitionPostProcessor

DestructionAwareBeanPostProcessor接口继承BeanPostProcessor接口。多出了1个方法：

```java
void postProcessMergedBeanDefinition(RootBeanDefinition beanDefinition, Class<?> beanType, String beanName);
```

该方法是bean在合并Bean定义之后调用

## 总结

Spring内部对bean的构造已经形成了一套体系。如果我们想修改这套体系，只能使用Spring提供的BeanPostProcessor接口去处理。这样做的好处：

**遵循设计模式的开闭原则，对扩展开放，对修改关闭。 我们只需要实现接口进行扩展即可，不需要修改内部的源码**

下一篇，将分析Spring内置的一些BeanPostProcessor的功能。
