## 一、前言
> 用过Spring Cloud的同学都知道在使用动态配置刷新的我们要配置一个@RefreshScope 在类上才可以实现对象属性的的动态更新，本着知其所以然的态度，晚上没事儿又把这个点回顾了一下，下面就来简单的说下自己的理解。

总览下，实现@RefreshScope 动态刷新的就需要以下几个：
- @ Scope   
- @RefreshScope
- RefreshScope       
- GenericScope   
- Scope
- ContextRefresher

## 二、@Scope
**一句话，@RefreshScope 能实现动态刷新全仰仗着@Scope 这个注解，这是为什么呢？**

@Scope 代表了Bean的作用域，我们来看下其中的属性：
```java
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Scope {

	/**
	 * Alias for {@link #scopeName}.
	 * @see #scopeName
	 */
	@AliasFor("scopeName")
	String value() default "";

	/**
	 *  singleton  表示该bean是单例的。(默认)
     *  prototype    表示该bean是多例的，即每次使用该bean时都会新建一个对象。
     *  request        在一次http请求中，一个bean对应一个实例。
     *  session        在一个httpSession中，一个bean对应一个实例
	 */
	@AliasFor("value")
	String scopeName() default "";

	/**
    *   DEFAULT			不使用代理。(默认)
	* 	NO				不使用代理，等价于DEFAULT。
	* 	INTERFACES		使用基于接口的代理(jdk dynamic proxy)。
	* 	TARGET_CLASS	使用基于类的代理(cglib)。
    */
	ScopedProxyMode proxyMode() default ScopedProxyMode.DEFAULT;

}

```

通过代码我们可以清晰的看到两个主要属性value 和 proxyMode，value就不多说了，大家平时经常用看看注解就可以。proxyMode 这个就有意思了，而这个就是@RefreshScope  实现的本质了。

**我们需要关心的就是ScopedProxyMode.TARGET_CLASS 这个属性，当ScopedProxyMode 为TARGET_CLASS 的时候会给当前创建的bean 生成一个代理对象，会通过代理对象来访问，每次访问都会创建一个新的对象。**

理解起来可能比较晦涩，那先来看下实现再回头来看这句话。

## 三、RefreshScope 的实现原理

1. 先来看下@RefreshScope
```java
@Target({ ElementType.TYPE, ElementType.METHOD })
@Retention(RetentionPolicy.RUNTIME)
@Scope("refresh")
@Documented
public @interface RefreshScope {
	/**
	 * @see Scope#proxyMode()
	 */
	ScopedProxyMode proxyMode() default ScopedProxyMode.TARGET_CLASS;

}
```

2. 可以看出，它使用就是 @Scope ，其内部就一个属性默认 ScopedProxyMode.TARGET_CLASS。知道了是通过Spring Scope 来实现的那就简单了，我们来看下Scope 这个接口
  
```java
public interface Scope {

	/**
	 * Return the object with the given name from the underlying scope,
	 * {@link org.springframework.beans.factory.ObjectFactory#getObject() creating it}
	 * if not found in the underlying storage mechanism.
	 * <p>This is the central operation of a Scope, and the only operation
	 * that is absolutely required.
	 * @param name the name of the object to retrieve
	 * @param objectFactory the {@link ObjectFactory} to use to create the scoped
	 * object if it is not present in the underlying storage mechanism
	 * @return the desired object (never {@code null})
	 * @throws IllegalStateException if the underlying scope is not currently active
	 */
	Object get(String name, ObjectFactory<?> objectFactory);

 
	@Nullable
	Object remove(String name);

 
	void registerDestructionCallback(String name, Runnable callback);

 
	@Nullable
	Object resolveContextualObject(String key);

	 
	@Nullable
	String getConversationId();

}

```

看下接口，我们只看Object get(String name, ObjectFactory<?> objectFactory); 这个方法帮助我们来创建一个新的bean ，也就是说，@RefreshScope 在调用 刷新的时候会使用此方法来给我们创建新的对象，这样就可以通过spring 的装配机制将属性重新注入了，也就实现了所谓的动态刷新。

3. 那它究竟是怎么处理老的对象，又怎么除法创建新的对象呢？

在开头我提过几个重要的类，而其中 RefreshScope extends GenericScope, GenericScope  implements Scope。

 所以通过查看代码，是GenericScope 实现了 Scope 最重要的  get(String name, ObjectFactory<?> objectFactory) 方法，在GenericScope 里面 包装了一个内部类 BeanLifecycleWrapperCache 来对加了 @RefreshScope 从而创建的对象进行缓存，使其在不刷新时获取的都是同一个对象。（这里你可以把 BeanLifecycleWrapperCache 想象成为一个大Map 缓存了所有@RefreshScope  标注的对象）

知道了对象是缓存的，所以在进行动态刷新的时候，只需要清除缓存，重新创建就好了。
来看代码，眼见为实，只留下关键方法：

```java
// ContextRefresher 外面使用它来进行方法调用 ============================== 我是分割线

	public synchronized Set<String> refresh() {
		Set<String> keys = refreshEnvironment();
		this.scope.refreshAll();
		return keys;
	}

// RefreshScope 内部代码  ============================== 我是分割线

	@ManagedOperation(description = "Dispose of the current instance of all beans in this scope and force a refresh on next method execution.")
	public void refreshAll() {
		super.destroy();
		this.context.publishEvent(new RefreshScopeRefreshedEvent());
	}


// GenericScope 里的方法 ============================== 我是分割线

	//进行对象获取，如果没有就创建并放入缓存
	@Override
	public Object get(String name, ObjectFactory<?> objectFactory) {
		BeanLifecycleWrapper value = this.cache.put(name,
				new BeanLifecycleWrapper(name, objectFactory));
		locks.putIfAbsent(name, new ReentrantReadWriteLock());
		try {
			return value.getBean();
		}
		catch (RuntimeException e) {
			this.errors.put(name, e);
			throw e;
		}
	}
	//进行缓存的数据清理
	@Override
	public void destroy() {
		List<Throwable> errors = new ArrayList<Throwable>();
		Collection<BeanLifecycleWrapper> wrappers = this.cache.clear();
		for (BeanLifecycleWrapper wrapper : wrappers) {
			try {
				Lock lock = locks.get(wrapper.getName()).writeLock();
				lock.lock();
				try {
					wrapper.destroy();
				}
				finally {
					lock.unlock();
				}
			}
			catch (RuntimeException e) {
				errors.add(e);
			}
		}
		if (!errors.isEmpty()) {
			throw wrapIfNecessary(errors.get(0));
		}
		this.errors.clear();
	}
```

   

通过观看源代码我们得知，我们截取了三个片段所得之，ContextRefresher 就是外层调用方法用的，GenericScope 里面的 get 方法负责对象的创建和缓存，destroy 方法负责再刷新时缓存的清理工作。当然spring n内部还进行很多其他有趣的处理，有兴趣的同学可以详细看一下。

## 四、总结
综上所述，来总结下@RefreshScope 实现流程

1. 需要动态刷新的类标注@RefreshScope  注解

2. @RefreshScope  注解标注了@Scope 注解，并默认了ScopedProxyMode.TARGET_CLASS; 属性，此属性的功能就是在创建一个代理，在每次调用的时候都用它来调用GenericScope get 方法来获取对象

3. 如属性发生变更会调用 ContextRefresher   refresh()  -》RefreshScope refreshAll()  进行缓存清理方法调用，并发送刷新事件通知 -》 GenericScope 真正的 清理方法destroy() 实现清理缓存

4. 在下一次使用对象的时候，会调用GenericScope  get(String name, ObjectFactory<?> objectFactory)  方法创建一个新的对象，并存入缓存中，此时新对象因为Spring 的装配机制就是新的属性了。

![](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9vc2NpbWcub3NjaGluYS5uZXQvb3NjbmV0L3VwLWFlNDE5YjJiYTk0Njc3ZGQxZWYxMzA3MGFlODVhYzU5MTk1LkpQRUc?x-oss-process=image/format,png)


>专注于分享技术干货文章的地方，内容涵盖java基础、中间件、分布式、apm监控方案、异常问题定位等技术栈。多年基础架构经验，擅长基础组件研发，分布式监控系统，热爱技术，热爱分享

![qrcode_for_gh_13314ac27929_258.jpg](https://imgconvert.csdnimg.cn/aHR0cHM6Ly91cGxvYWQtaW1hZ2VzLmppYW5zaHUuaW8vdXBsb2FkX2ltYWdlcy80NDA1NjYzLTZkYzRmNTQ3NmRhOGY2NTUuanBn?x-oss-process=image/format,png)