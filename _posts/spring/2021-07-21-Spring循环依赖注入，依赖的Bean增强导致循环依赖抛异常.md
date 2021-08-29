---
layout:     post
title:      "Spring循环依赖注入，依赖的Bean增强导致循环依赖抛异常"
subtitle:   "spring依赖注入"
date:       2021-07-21
author:     "CHuiL"
header-img: "/img/bg/spring.jpg"
tags:
    - spring
---

### spring 通过缓存解决循环依赖

Spring解决循环依赖的原理，其实就是依靠对象创建的中间状态。我们一个对象实例的创建，其实是可以分为实例化，然后在初始化。实例化就是分配了内存空间，引用已经指向了该内存的地址，该对象可以正常被引用了。不过此时还没有初始化，也就是内存空间中的数据还没有初始化。 

```
class A {
    B b;
    public A() {
        b = new B();
    }
}

class B {
    A a;
    public B() {
        a = new A();
    }
}
```
面对上面这段代码，我们运行起来会一直循环调用AB的构造，导致内存溢出。但是如果我们不依靠构造函数来注入依赖，而是先实例化出来，在set进去其实就可以实现循环依赖了。
```
public class Main {

    public static void main(String[] args) throws Exception {
        A a = new A();
        B b = new B();
        a.set(b);
        b.set(a);
    }
    
    class A {
            B b;
            public A() {
            }
            public setB(B b){
                this.b = b;
            }
        }
        
        class B {
            A a;
            public B() {
            }
            public setA(A a){
                this.a = a;
            }
            
        }
}
```

而spring循环依赖解决的思路，其实基本就是上面的思路。比如我们通过@Autowired的方式来注入AB。
```
@Service
public class A {
    @Autowired
    private B b;
}

@Service
public class B {
    @Autowired
    private A a;
}
```
Spring在构造A的时候，会先实例化A，然后将A的实例放入到缓存中，然后依赖注入B。在依赖注入B的过程中，也会创建B。在实例化B之后，也会将B放入缓存中，此时注入B的依赖，这个时候在获取B的依赖A的时候，通过缓存可以拿到B的实例引用A（此时A还没初始化完成），然后注入进去，最终返回B的实例。  

而A将B注入到自己的依赖之后，A也完成了初始化。  
此刻AB就已经创建完成，循环的依赖也注入成功了。  

基本原理其实就是这么简单。但是spring中使用了三层缓存来解决这个问题。 主要有多方面的考虑，其中就有对增强类（代理类）实例的考虑。  


具体源码实现。  

```
public class DefaultSingletonBeanRegistry extends SimpleAliasRegistry implements SingletonBeanRegistry {
	...
	// 从上至下 分表代表这“三级缓存”
	private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256); //一级缓存
	private final Map<String, Object> earlySingletonObjects = new HashMap<>(16); // 二级缓存
	private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<>(16); // 三级缓存
	...
	
	/** Names of beans that are currently in creation. */
	// 这个缓存也十分重要：它表示bean创建过程中都会在里面呆着~
	// 它在Bean开始创建时放值，创建完成时会将其移出~
	private final Set<String> singletonsCurrentlyInCreation = Collections.newSetFromMap(new ConcurrentHashMap<>(16));

	/** Names of beans that have already been created at least once. */
	// 当这个Bean被创建完成后，会标记为这个 注意：这里是set集合 不会重复
	// 至少被创建了一次的  都会放进这里~~~~
	private final Set<String> alreadyCreated = Collections.newSetFromMap(new ConcurrentHashMap<>(256));
}
```
流程
![image](/chuil/img/spring/static-auto-1.png)

而前面提到的类增强，也就是我们在创建一个Bean的时候，如果有注入@Async的注解，那么我们得到的应该是一个代理Bean实例。  

而这在上面的实例流程就出现了一个问题，还是以前面的AB为例子。加入A是增强类，但是起初A实例化后，此时还是原生的类对象，放入缓存的也是原生的类对象。B在注入的A依赖也是原生的对象实例。  

在B创建完成之后，B依赖的A就是原生的实例。  

而在A初始化完成之后，后续会执行`exposedObject = initializeBean(beanName, exposedObject, mbd);`这么一个操作，该操作扫描该类是否有@Async注解，有的话会增强该类，并返回该类的代理类对象。  

而这是返回的A实例就与B中的实例对象不同了。而这就会导致构建时抛出循环依赖的错误。  

```
protected Object doCreateBean( ... ){
	...
	boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences && isSingletonCurrentlyInCreation(beanName));
	if (earlySingletonExposure) {
		addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
	}
	...

	// populateBean这一句特别的关键，它需要给A的属性赋值，所以此处会去实例化B~~
	// 而B我们从上可以看到它就是个普通的Bean（并不需要创建代理对象），实例化完成之后，继续给他的属性A赋值，而此时它会去拿到A的早期引用
	// 也就在此处在给B的属性a赋值的时候，会执行到上面放进去的Bean A流程中的getEarlyBeanReference()方法  从而拿到A的早期引用~~
	// 执行A的getEarlyBeanReference()方法的时候，会执行自动代理创建器，但是由于A没有标注事务，所以最终不会创建代理，so B合格属性引用会是A的**原始对象**
	// 需要注意的是：@Async的代理对象不是在getEarlyBeanReference()中创建的，是在postProcessAfterInitialization创建的代理
	// 从这我们也可以看出@Async的代理它默认并不支持你去循环引用，因为它并没有把代理对象的早期引用提供出来~~~（注意这点和自动代理创建器的区别~）

	// 结论：此处给A的依赖属性字段B赋值为了B的实例(因为B不需要创建代理，所以就是原始对象)
	// 而此处实例B里面依赖的A注入的仍旧为Bean A的普通实例对象（注意  是原始对象非代理对象）  注：此时exposedObject也依旧为原始对象
	populateBean(beanName, mbd, instanceWrapper);
	
	// 标注有@Async的Bean的代理对象在此处会被生成~~~ 参照类：AsyncAnnotationBeanPostProcessor
	// 所以此句执行完成后  exposedObject就会是个代理对象而非原始对象了
	exposedObject = initializeBean(beanName, exposedObject, mbd);
	
	...
	// 这里是报错的重点~~~
	if (earlySingletonExposure) {
		// 上面说了A被B循环依赖进去了，所以此时A是被放进了二级缓存的，所以此处earlySingletonReference 是A的原始对象的引用
		// （这也就解释了为何我说：如果A没有被循环依赖，是不会报错不会有问题的   因为若没有循环依赖earlySingletonReference =null后面就直接return了）
		Object earlySingletonReference = getSingleton(beanName, false);
		if (earlySingletonReference != null) {
			// 上面分析了exposedObject 是被@Aysnc代理过的对象， 而bean是原始对象 所以此处不相等  走else逻辑
			if (exposedObject == bean) {
				exposedObject = earlySingletonReference;
			}
			// allowRawInjectionDespiteWrapping 标注是否允许此Bean的原始类型被注入到其它Bean里面，即使自己最终会被包装（代理）
			// 默认是false表示不允许，如果改为true表示允许，就不会报错啦。这是我们后面讲的决方案的其中一个方案~~~
			// 另外dependentBeanMap记录着每个Bean它所依赖的Bean的Map~~~~
			else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
				// 我们的Bean A依赖于B，so此处值为["b"]
				String[] dependentBeans = getDependentBeans(beanName);
				Set<String> actualDependentBeans = new LinkedHashSet<>(dependentBeans.length);

				// 对所有的依赖进行一一检查~	比如此处B就会有问题
				// “b”它经过removeSingletonIfCreatedForTypeCheckOnly最终返返回false  因为alreadyCreated里面已经有它了表示B已经完全创建完成了~~~
				// 而b都完成了，所以属性a也赋值完成儿聊 但是B里面引用的a和主流程我这个A竟然不相等，那肯定就有问题(说明不是最终的)~~~
				// so最终会被加入到actualDependentBeans里面去，表示A真正的依赖~~~
				for (String dependentBean : dependentBeans) {
					if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
						actualDependentBeans.add(dependentBean);
					}
				}
	
				// 若存在这种真正的依赖，那就报错了~~~  则个异常就是上面看到的异常信息
				if (!actualDependentBeans.isEmpty()) {
					throw new BeanCurrentlyInCreationException(beanName,
							"Bean with name '" + beanName + "' has been injected into other beans [" +
							StringUtils.collectionToCommaDelimitedString(actualDependentBeans) +
							"] in its raw version as part of a circular reference, but has eventually been " +
							"wrapped. This means that said other beans do not use the final version of the " +
							"bean. This is often the result of over-eager type matching - consider using " +
							"'getBeanNamesOfType' with the 'allowEagerInit' flag turned off, for example.");
				}
			}
		}
	}
	...
}
```


这里貌似并不是所有生成代理类对象的Bean都会有这个问题，@Async的代理生成实在初始化完成之后的。但是如果代理类的生成可以在初始化之前，就可以避免这个问题。因为三级代理是一个ObjectFatory，在这个工厂类中，他会先遍历出所有的post-processors来生产代理类，这样在B一开始获取A时就已经是代理类了，后面A在创建完成时，也是同样的一个代理类。

## 参考

- [使用@Async异步注解导致该Bean在循环依赖时启动报BeanCurrentlyInCreationException异常的根本原因分析，以及提供解决方案【享学Spring】](https://cloud.tencent.com/developer/article/1497689)
- [一文告诉你Spring是如何利用“三级缓存“巧妙解决Bean的循环依赖问题的【享学Spring】](https://blog.csdn.net/f641385712/article/details/92801300)