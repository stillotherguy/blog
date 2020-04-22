#概要

本文是从Spring AOP API的角度去阐述Spring AOP的一些“内涵”，抓住重点，结合一些Spring AOP API的实战例子，让我们可以更好更深入的了解它，碰到问题也可以得心应手的处理，写框架和中间件用到Spring AOP时也可以从容不迫。
#通知

在没有maven，手动copy jar包到classpath的年代，使用Spring AOP时可能会注意到一个叫aopalliance的jar包，这个jar包里定义了AOP的一些基础API，最重要的一个接口叫Advice（是一个标记接口），顾名思义，这就是AOP基本概念里的“通知”，而“通知”的职责是拦截目标方法并执行具体的切面逻辑，这个职责定义在了Advice的子接口MethodInterceptor中，现在来看下MethodInterceptor的方法签名：
```
public interface MethodInterceptor extends Interceptor {

    Object invoke(MethodInvocation invocation) throws Throwable;
}
```
此接口着重于在被拦截的方法前、后、碰到异常时等场景来写一些自定义的切面逻辑，拿我写过的一个打印进出口日志的切面为例，典型写法为：
```
public class LogMethodInterceptor implements MethodInterceptor {

    @Override
    public Object invoke(MethodInvocation methodInvocation) throws Throwable {
        // 打印入口日志
        printInput(...);
        Object retObj = null;
        try {
            // 继续向后执行责任链中的其他通知，类似Servlet Filter的FilterChain.doChain()，本文后面会有涉及
            return methodInvocation.proceed();
        } finally {
            // 打印出口日志
            printOutput(...);
        }
    }
}
```
典型的实现有偏应用的通知如：TransactionInterceptor（Spring事务通知）。偏通用的通知如：MethodValidationInterceptor（Spring方法入参验证通知）、ThrowsAdviceInterceptor（异常处理通知）、AfterReturningAdviceInterceptor（返回值处理通知）等等
#切点
光有拦截器还不够，拦截器只是实现了一些切面的逻辑，当使用Spring AspectJ切面时，我们会在方法上打上一些特定注解，来指定拦截某某包下的某某类的某某方法。或当使用Spring事务时，我们会在方法上打上@Transactional注解来表明当前方法需要声明式事务管理。这些配置的背后都是切点在发挥作用，针对Spring容器茫茫多的bean，使用切点逻辑对其进行匹配（例如Spring事务的场景就是匹配bean的所有方法，看其是否有@Transactional注解）来决定是否代理当前的bean对象，如果切点命中，则织入通知里的切面逻辑，此时就涉及到一个匹配过程。这个功能是哪些api在发挥作用？这时候有另外一个重要的接口，Pointcut
```
public interface Pointcut {
    
    /**
     * 类过滤
     */
    ClassFilter getClassFilter();

    /**
     * 方法匹配
     */
    MethodMatcher getMethodMatcher();
}

public interface ClassFilter {

    /*
     * 是否匹配目标类
     */
    boolean matches(Class<?> clazz);

    ClassFilter TRUE = TrueClassFilter.INSTANCE;
}

public interface MethodMatcher {
    
    /*
     * 是否匹配目标方法
     */
    boolean matches(Method method, @Nullable Class<?> targetClass);

    /*
     * 是否runtime匹配目标方法
     */
    boolean isRuntime();

    /*
     * 如果isRuntime为true，会调用此方法，把运行时入参传入来决定是否匹配当前方法
     */
    boolean matches(Method method, @Nullable Class<?> targetClass, Object... args);

    MethodMatcher TRUE = TrueMethodMatcher.INSTANCE;
}

```
接口方法很直观，可以通过实现此接口来指定过滤目标Class的逻辑以及匹配目标Method的逻辑。都两者都可以匹配上，则切入当前方法，执行通知逻辑。比如我们可以使用AntPathMatcher（实现*.*.Test类似写法的实现工具类）来切入到一组package下的目标类的目标方法，实现切入，Spring AOP提供了很多三者的工具实现类，如正则表达式切点（JdkRegexpMethodPointcut）、AspectJ切点（AspectJExpressionPointcut）、事务切点（TransactionAttributeSourcePointcut）等等
#通知+切点=？
将通知和切点整合在一起，就有了一个完整的切面，这个接口在Spring里叫Advisor
```
public interface Advisor {

    Advice getAdvice();
}

public interface PointcutAdvisor extends Advisor {

    Pointcut getPointcut();
}
```
从方法签名看出，Advisor只定义了返回通知的方法，返回切点的方法定义在了其子接口PointcutAdvisor。同样的，Spring也提供了很多默认实现，如：默认切面实现（DefaultPointcutAdvisor，写自定义切面时常常会用到）、AspectJ切面（AspectJPointcutAdvisor）、事务切面（TransactionAttributeSourceAdvisor）等等，现在切面的核心接口我们已经大致了解了，如何结合切面拿到一个真正的代理对象？这时候，配置、封装和解耦代理对象创建过程的代理工厂ProxyFactory闪亮登场了
#代理工厂
Spring AOP的代理工厂叫ProxyFactory，典型的工厂模式。通过一个小例子，可以直观的感受一下代理对象的创建过程，Spring AOP几乎所有场景的代理对象创建都是通过此api来实现的：
```
// 简单通知实现
public class SimpleAdvice implements MethodInterceptor {

    @Override
    public Object invoke(MethodInvocation arg0) throws Throwable {
        System.out.println("before invoke...");
        Object obj = arg0.proceed();
        System.out.println("after invoke...");
        return obj;
    }
}

// 简单切点实现
public class SimpleDynamicPointcut extends DynamicMethodMatcherPointcut {

    @Override
    public boolean matches(Method method, Class<?> klass, Object[] args) {
        // 当第一个方法参数值不为100时，匹配目标方法
        int x = (Integer) args[0];
        return (x != 100);
    }

    @Override
    public ClassFilter getClassFilter() {
        return new ClassFilter() {
            public boolean matches(Class<?> klass) {
                // 当目标类是Foo子类时，匹配目标类
                return Foo.class.isAssignableFrom(klass);
            }
        };
    }

}

public class Main {

    public static void main(String[] args) {
        // 定义自定义切面
        Advisor advisor = new DefaultPointcutAdvisor(new SimpleDynamicPointcut(), new SimpleAdvice());
        // 创建代理工厂
        ProxyFactory pf = new ProxyFactory();
        // 配置代理工厂
        // 配置目标类
        pf.setTargetClass(Foo.class);
        // 配置是否暴露代理对象到AopContext
        pf.setExposeProxy(false);
        // 配置代理方式为JDK Proxy
        pf.setProxyTargetClass(false);

        Foo foo = new Impl();
        // 设置目标对象
        pf.setTarget(foo);
        // 将切面放入工厂
        pf.addAdvisor(advisor);
        // 通过工厂方法拿到被代理对象
        Foo proxy = (Foo) pf.getProxy();
        proxy.foo(1000);
        proxy.foo(100);
    }
}
```
结合上面的例子发现，经过Spring AOP工厂对象，对其做简单配置，就可以得到一个代理对象。有两行代码需要我们特别关注：
## pf.setTarget(foo)

这里引申出两个问题：

  - 为什么要把原始对象设置到工厂对象里？

当在写切面逻辑时，我们可能会用到原始对象，所以当使用Spring AOP框架时，框架会自动设置原始对象给到我们，而当使用原生API时，则需要手动去指定，否则会拿不到原始对象
```
public class HelloMethodInterceptor implements MethodInterceptor {

    @Override
    public Object invoke(MethodInvocation methodInvocation) throws Throwable {
        // 拿到原始对象
        Object target = methodInvocation.getThis();
    }
}
```
- 是哪个接口在起作用？

TargetSource接口就是为我们提供原始对象的（进入上述`pf.setTarget(foo)`方法可以看到），Spring默认会从BeanFacory里去取到原始对象并设置到代理工厂里，常用的实现有：单例原始对象源SingletonTargetSource、延迟加载原始对象源LazyInitTargetSource、原型原始对象源PrototypeTargetSource、热替换原始对象源HotSwappableTargetSource
```
public interface TargetSource extends TargetClassAware {

    /**
     * 原始对象类型
     */
    Class<?> getTargetClass();

    /**
     * 原始对象是否immutable
     */
    boolean isStatic();

    /**
     * 获得原始对象
     */
    Object getTarget() throws Exception;

    /**
     * 释放原始对象
     */
    void releaseTarget(Object target) throws Exception;
}

```
## Foo proxy = (Foo) pf.getProxy()

查看getProxy方法的源码之后可以发现ProxyFactory先是委托AopProxyFactory（默认实现为DefaultAopProxyFactory）这个接口的createAopProxy方法创建一个AopProxy（有JDK和Cglib两种实现）实例，然后调用AopProxy接口的getProxy方法拿到被代理对象
```
public interface AopProxyFactory {

    /**
     * 创建代理对象，特别注意此时的入参AdvisedSupport即为ProxyFactory本身，ProxyFactory继承了这个类
     */
    AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException;

}

public class DefaultAopProxyFactory implements AopProxyFactory, Serializable {

    @Override
    public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {
        if (config.isOptimize() || config.isProxyTargetClass() || hasNoUserSuppliedProxyInterfaces(config)) {
            // 如果采用优化模式或根据原始对象类型判断策略或目标类型没有实现任何接口，则根据原始对象类型类决定使用哪种代理方式
            Class<?> targetClass = config.getTargetClass();
            if (targetClass == null) {
                throw new AopConfigException("TargetSource cannot determine target class: " +
                        "Either an interface or a target is required for proxy creation.");
            }
            if (targetClass.isInterface() || Proxy.isProxyClass(targetClass)) {
                // 如果目标类型是接口或被jdk代理过的类型，则创建jdk代理
                return new JdkDynamicAopProxy(config);
            }
            // 否则创建cglib代理
            return new ObjenesisCglibAopProxy(config);
        }
        else {
            // 创建jdk代理
            return new JdkDynamicAopProxy(config);
        }
    }
}

public interface AopProxy {

    /**
     * 创建代理对象
     */
    Object getProxy();

    /**
     * 创建代理对象
     */
	Object getProxy(ClassLoader classLoader);

}
```
现在接着来说说代理对象，又有哪些”幺蛾子“在里面？
#被代理过的对象
众所周知，spring aop有两种代理方式，Cglib与JDK Proxy，但被代理过的对象“偷偷地”实现了几个接口，拿JDK代理方式来举例，查看JdkDynamicAopProxy的getProxy方法可以发现一些端倪：
```
final class JdkDynamicAopProxy implements AopProxy, InvocationHandler, Serializable {
    public Object getProxy(@Nullable ClassLoader classLoader) {
        if (logger.isDebugEnabled()) {
            logger.debug("Creating JDK dynamic proxy: target source is " + this.advised.getTargetSource());
        }
        // 获得JDK代理所需要的代理接口，自动填充用户接口（即业务接口）和下文所述三个Spring AOP内部接口，使得代理对象被动实现这些接口
        Class<?>[] proxiedInterfaces = AopProxyUtils.completeProxiedInterfaces(this.advised, true);
        findDefinedEqualsAndHashCodeMethods(proxiedInterfaces);
        // 创建JDK代理对象
        return Proxy.newProxyInstance(classLoader, proxiedInterfaces, this);
   }
}
```
接下来重点介绍下代理对象被动实现的Spring AOP内部接口以及实现这些接口的意图：
- SpringProxy  
这个是一个标记接口，用来标记当前对象被Spring代理过，从AopUtils这个内部工具类里判断是否是Spring代理、JDK代理或Cglib代理的工具方法实现就可以看到其用途：
    ```
        // 是否是Spring AOP代理
        public static boolean isAopProxy(@Nullable Object object) {
            return (object instanceof SpringProxy &&
                    (Proxy.isProxyClass(object.getClass()) || ClassUtils.isCglibProxyClass(object.getClass())));
        }
        // 是否是Spring AOP JDK Proxy代理
        public static boolean isJdkDynamicProxy(@Nullable Object object) {
            return (object instanceof SpringProxy && Proxy.isProxyClass(object.getClass()));
        }
        // 是否是Spring AOP Cglib代理
        public static boolean isCglibProxy(@Nullable Object object) {
            return (object instanceof SpringProxy && ClassUtils.isCglibProxy(object));
        }
    ```
    
- Advised  
代理对象实现的这个接口，我认为是Spring AOP的精华所在，他让我们在用Spring AOP时可以获得”肖申克的救赎“，也就是说，当我们对被代理过的对象不满意，我们可以有second chance，在对象已被代理的情况下无需进行二次代理来实现一些原始对象替换、切面管理等功能（一般情况下，最佳的位置是在BeanPostProcessor的实现方法里），首先我们来看这个接口定义了哪些方法（由于方法比较多，挑重点的说一下）  
    
    ####配置被代理对象的原始对象  
    可以对代理对象的原始对象进行”偷梁换柱“
    ```
        void setTargetSource(TargetSource targetSource);

        TargetSource getTargetSource();
    ```  
    ####切面的管理
    这里可以看出Spring AOP的强大，我们不仅可以直接（如AspectJ注解配置方式或`<aop:config>`标签配置方式）或间接（如Spring事务）配置切面，还可以在运行时拿到Spring AOP代理对象（一般是通过BeanPostProcessor或InitializingBean），将其强制转换为Advised对象，调用如下方法对切面进行动态的管理（如果想关闭这个特性，可以在创建ProxyFactory对象时，调用“setFrozen(true)”即可，此时会锁定代理对象当前的切面配置，不允许对其做变更）
    ```
        Advisor[] getAdvisors();
    
        void addAdvisor(Advisor advisor) throws AopConfigException;
    
        void addAdvisor(int pos, Advisor advisor) throws AopConfigException;
    
        boolean removeAdvisor(Advisor advisor);
    
        void removeAdvisor(int index) throws AopConfigException;
    ```
    ####AOP基础配置
    在使用Spring AOP时，我们可以指定两个配置。即：`<aop:aspectj-autoproxy proxy-target-class="true" expose-proxy="true"/>`，这些配置会被指定给ProxyFactory（ProxyFactory也实现了Advised接口），当创建好代理对象后，这些配置又会被复制到被代理对象里：
    ```
        /**
         * 是否冻结切面管理，此为内部配置，不可修改，只可在创建ProxyFactory时被指定，并被copy到当前代理对象
         */
        boolean isFrozen();
    
        /**
         * 是否根据目标类型决定代理方式，Spring AOP XML或Java Config可配置
         */
        boolean isProxyTargetClass();
    
        /**
         * 配置是否暴露代理到AopContext，Spring AOP XML或Java Config可配置
         */
        void setExposeProxy(boolean exposeProxy);
    
        /**
         * 返回是否暴露代理配置
         */
        boolean isExposeProxy();
    ```
- DecoratingProxy  
针对对象被多次代理过的场景，拿到代理对象的最终目标类型，此特性需要Spring最低版本为4.3或以上

现在来深入看下被代理对象的被代理方法内部执行逻辑，探索代理对象和切面是怎么相互协作的。
#切面与代理对象的协作

还是拿Spring AOP JDK Proxy代理对象对应的InvocationHandler实现里的源码来说明（对应的Cglib实现类为CglibAopProxy）：
```
final class JdkDynamicAopProxy implements AopProxy, InvocationHandler, Serializable {
    ...

    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        MethodInvocation invocation;
        Object oldProxy = null;
        boolean setProxyContext = false;

        TargetSource targetSource = this.advised.targetSource;
        Object target = null;

        try {
            // 略去无关紧要的逻辑
            ...

            Object retVal;

            if (this.advised.exposeProxy) {
                // 暴露当前代理到线程上下文
                oldProxy = AopContext.setCurrentProxy(proxy);
                setProxyContext = true;
            }

            // 拿到被代理源对象
            target = targetSource.getTarget();
            Class<?> targetClass = (target != null ? target.getClass() : null);

            // 搜集所有切面，以当前目标类和方法为上下文执行所有切面对应的切点内部的类过滤ClassFilter和方法匹配MethodMatcher逻辑，拿到匹配后的切面集合
            List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);

            // 如果切面为空，则直接反射调用目标方法
            if (chain.isEmpty()) {
                Object[] argsToUse = AopProxyUtils.adaptArgumentsIfNecessary(method, args);
                retVal = AopUtils.invokeJoinpointUsingReflection(target, method, argsToUse);
            }
            else {
                // 构建责任链，类似Servlet Filter的FilterChain
                invocation = new ReflectiveMethodInvocation(proxy, target, method, args, targetClass, chain);
                // 驱动责任链执行
                retVal = invocation.proceed();
            }

            // 省略无关紧要的逻辑
            ...
        }
        finally {
            if (target != null && !targetSource.isStatic()) {
                // 释放原始对象
                targetSource.releaseTarget(target);
            }
            if (setProxyContext) {
                AopContext.setCurrentProxy(oldProxy);
            }
        }
    }

    ...
}

```
#总结

最后，对Spring AOP进行一个小的总结，来看清它的本质。大致可以分为几步：
- 注册切面到容器
    - 通过`<aop:config />`这种old school的方式
    - 通过AspectJ注解@Aspect
    - 通过注册实现Advisor接口的bean到Spring容器，此方式一般用的比较少
- 收集容器里实现了Advisor的切面对象集合并结合AspectJ切面集合（由于AspectJ切面没有实现Advisor接口而是使用了@Aspect注解，所以需要特殊处理），将二者合并返回，针对返回的切面集合，以当前bean对象为上下文对其进行切点匹配，得到符合切点条件的切面集合
- 使用ProxyFacotory API配置、创建并返回代理对象

那么，上述的核心逻辑在哪些代理里实现的呢？

当我们配置启用Spring AOP时，我们可以查看解析`<aop:aspectj-autoproxy />`标签的具体实现类，AopNamespaceHandler，最终发现它注册了一个类名为AnnotationAwareAspectJAutoProxyCreator的Spring AOP基础设施bean到容器里，由于Spring大量使用了模板方法模式，所以沿着类继承结构可以发现最核心的就是AbstractAutoProxyCreator这个抽象类（典型的模板方法模式，核心逻辑在父类，子类只是扩展实现扫描容器里所有的AspectJ切面，在上述步骤2里合并返回所有切面集合）
```
public abstract class AbstractAutoProxyCreator extends ProxyProcessorSupport
        implements SmartInstantiationAwareBeanPostProcessor, BeanFactoryAware {
        
    // bean实例化钩子方法，如果返回结果不为null，则跳过默认bean实例化过程  
    @Override
    public Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) throws BeansException {
        // 省略非重点逻辑
        ...

        // 拿到原始对象
        TargetSource targetSource = getCustomTargetSource(beanClass, beanName);
        if (targetSource != null) {
            if (StringUtils.hasLength(beanName)) {
                this.targetSourcedBeans.add(beanName);
            }

            // 对应上述步骤2，搜集所有切面集合，以当前beanClass为上下文，通过上述切点API得到匹配当前对象的切面集合，子类会扩展实现收集以AspectJ注解方式配置的切面
            Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(beanClass, beanName, targetSource);
            // 对应上述步骤3，通过ProxyFactory api来得到代理对象
            Object proxy = createProxy(beanClass, beanName, specificInterceptors, targetSource);
            // 放入cache
            this.proxyTypes.put(cacheKey, proxy.getClass());
            // 返回代理对象，跳过默认bean实例化逻辑
            return proxy;
        }

        return null;
    }
}
```
看到这里，我们就对Spring AOP的本质有了一个清晰的认识。下面来讨论一下高级特性
#高级特性

- second chance

如之前所说，当拿到一个被代理过的对象，它并没有被织入过我想要的切面逻辑，此时我需要一个second chance。如上文所说，每个被代理对象都会实现一个Advised的接口，最佳的拦截位置是BeanPostProcessor接口（个人感觉InitializingBean在特定场景也适用），结合例子来看下应该怎么做：
```
public class AbstractAdvisingBeanPostProcessor implements BeanPostProcessor {

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) {
        if (bean instanceof AopInfrastructureBean || this.advisor == null) {
            // 忽略实现了AOP基础设施bean标记接口的bean
            return bean;
        }

        // 如果是代理对象
        if (bean instanceof Advised) {
            Advised advised = (Advised) bean;
            // 织入定制切面
            advised.addAdvisor(new CustomAdvisor(...));
        }
        
        return bean;
    }

}
```
- Introduction

Introduction可以在不改动目标类定义的情况下，为目标类添加新的属性以及行为。在Spring中，为目标对象添加新的属性和行为必须声明相应的接口以及相应的实现。这样，再通过特定的拦截器将新的接口定义以及实现类中的逻辑附加到目标对象之上。之后，目标对象（确切地说是目标对象的代理对象）就拥有了新的状态和行为。这个特定的拦截器就是org.springframework. aop.IntroductionInterceptor，之前讨论过的通知（即Advice），都是per-class的，在Spring AOP中，Introduction就是唯一的一种per-instance型Advice，与per-class类型的Advice不同，per-instance类型的Advice不会在目标类所有对象实例之间共享，而是会为不同的实例对象保存它们各自的状态以及相关逻辑。下面结合一个实战例子来看一下：
```
// 待introduce的接口
public interface IsModified {
    boolean isModified();
}

// IntroductionInterceptor是一个标记接口，DelegatingIntroductionInterceptor实现了它，提供将此拦截器实现的接口和切面逻辑织入到代理对象中的功能
public class IsModifiedMixin extends DelegatingIntroductionInterceptor implements IsModified {
    
    // 非per-class切面，所以此处是线程安全的
    private boolean isModified = false;
    
    // 待introduce接口的实现
    @Override
    public boolean isModified() {
        return isModified;
    }
    
    public Object invoke(MethodInvocation invocation) throws Throwable {

        if (!isModified) {
            if ((invocation.getMethod().getName().startsWith("set"))
                    && (invocation.getArguments().length == 1)) {

                Method getter = getGetter(invocation.getMethod());
                // 主要意义是，如果当前代理对象被调用过任意的setter方法，则标记其为被修改过
                if (getter != null) {
                    Object newVal = invocation.getArguments()[0];
                    Object oldVal = getter.invoke(invocation.getThis(), null);
                    
                    if((newVal == null) && (oldVal == null)) {
                        isModified = false;
                    } else if((newVal == null) && (oldVal != null)) {
                        isModified = true;
                    } else if((newVal != null) && (oldVal == null)) {
                        isModified = true;
                    } else {
                        isModified = (!newVal.equals(oldVal));
                    }
                }

            }
        }

        return super.invoke(invocation);
    }

    private Method getGetter(Method setter) {
        String getterName = setter.getName().replaceFirst("set", "get");
        try {
            return setter.getDeclaringClass().getMethod(getterName, (Class<?>[]) null);
        } catch (NoSuchMethodException ex) {
            return null;
        }
    }
}

public class TargetBean {

    private String name;
    
    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }
}

public class IntroductionExample {

    public static void main(String[] args) {
        TargetBean target = new TargetBean();
        target.setName("Ethan");
        
        // 构造introduction切面
        IntroductionAdvisor advisor = new IsModifiedAdvisor();
        //构造代理工厂
        ProxyFactory pf = new ProxyFactory();
        pf.setTarget(target);
        pf.addAdvisor(advisor);
        pf.setOptimize(true);
        // 拿到代理对象
        TargetBean proxy = (TargetBean) pf.getProxy();
        IsModified proxyInterface = (IsModified) proxy;

        System.out.println("is target bean " + (proxy instanceof TargetBean));
        System.out.println("is ismodified " + (proxy instanceof IsModified));
        System.out.println("changed? " + proxyInterface.isModified());

        proxy.setName("Zhang");
        System.out.println("changed? " + proxyInterface.isModified());
        proxy.setName("Li");
        System.out.println("changed? " + proxyInterface.isModified());
    }
}
```
运行结果：

is target bean true

is ismodified true

changed? false

changed? true

changed? true

Spring @DeclareParents功能使用的就是Spring AOP Introduction API，只不过区别上面例子，它是直接使用了例子中IsModifiedMixin的父类DelegatingIntroductionInterceptor，为此类指定一个待introduce的delegate对象（即使用@DeclareParents注解时为defaultImpl指定的类型所实例化的对象。此处略去例子，可以尝试一下）
- expose-proxy

特定的场景会用到暴露代理到AopContext的特性，需要配置`<aop:aspectj-autoproxy expose-proxy="true"/>`来启用此特性，还是结合例子说一下：
```
public interface Foo {
    
    @Transactional(propagation = Propagation.REQUIRED)
    void foo1();
    
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    void foo2();
}

public class Impl implements Foo {

	@Override
	public void foo1() {
        //this.foo1();
        Foo foo = (Foo) AopContext.currentProxy();
        foo.foo1();
	}

	@Override
	public void foo2() {
        // 省略实现
        ...
	}

}
```
当在foo1方法里直接使用this.foo2()来调用foo2方法时，是不会走事务切面的，这时候foo2方法要求的在方法开始执行前开启新事务的事务配置就会失效，此时需要通过AopContext API拿到当前this对象对应的代理对象，再调用foo2方法，即可避免此类问题。
#参考资料：

<https://docs.spring.io/spring/docs/4.3.0.RELEASE/spring-framework-reference/html/>
<https://docs.spring.io/spring/docs/4.3.0.RELEASE/javadoc-api>  
<https://www.javalobby.org//java/forums/t45691.html>

