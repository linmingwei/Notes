# Spring Bean的运行实现
## getBean()的流程
首先Spring会去AbstractBeanFatory这个类里面调用doGetBean()这个方法来获取bean。如果没有获取预加载的bean，则会去父工厂去获取，如果还没有获取这个被依赖的bean，则解析并创建这个bean。  
doGetBean()源码:  
```java
protected <T> T doGetBean(
    final String name, final Class<T> requiredType, final Object[] args, boolean typeCheckOnly)
    throws BeansException {

final String beanName = transformedBeanName(name);
Object bean;

// 检查单例缓存池中是否注册了这个bean
Object sharedInstance = getSingleton(beanName);
if (sharedInstance != null && args == null) {
    if (logger.isDebugEnabled()) {
        if (isSingletonCurrentlyInCreation(beanName)) {
            logger.debug("Returning eagerly cached instance of singleton bean '" + beanName +
                    "' that is not fully initialized yet - a consequence of a circular reference");
        }
        else {
            logger.debug("Returning cached instance of singleton bean '" + beanName + "'");
        }
    }
    bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
}

else {
    // Fail if we're already creating this bean instance:
    // We're assumably within a circular reference.
    if (isPrototypeCurrentlyInCreation(beanName)) {
        throw new BeanCurrentlyInCreationException(beanName);
    }

    // 检查beanfactory中是否有这个bean，如果没有递归获取父beanfactory检查有没有这个bean
    BeanFactory parentBeanFactory = getParentBeanFactory();
    if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
        // Not found -> check parent.
        String nameToLookup = originalBeanName(name);
        if (args != null) {
            // Delegation to parent with explicit args.
            return (T) parentBeanFactory.getBean(nameToLookup, args);
        }
        else {
            // No args -> delegate to standard getBean method.
            return parentBeanFactory.getBean(nameToLookup, requiredType);
        }
    }

    if (!typeCheckOnly) {
        markBeanAsCreated(beanName);
    }

    try {
        final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
        checkMergedBeanDefinition(mbd, beanName, args);

        // 检查当前bean是否存在循环依赖
        String[] dependsOn = mbd.getDependsOn();
        if (dependsOn != null) {
            for (String dependsOnBean : dependsOn) {
                if (isDependent(beanName, dependsOnBean)) {
                    throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                            "Circular depends-on relationship between '" + beanName + "' and '" + dependsOnBean + "'");
                }
                //先向容器注册依赖的bean
                registerDependentBean(dependsOnBean, beanName);
                //如果依赖的bean也存在依赖其他bean的场景，通过递归将其依赖的bean创建注册。其核心逻辑就是在bean实例化之前先解决其依赖其他bean的实例化问题，最后解决所有////bean的依赖问题
                getBean(dependsOnBean);
            }
        }

        //bean的作用域有： singleton prototype request session application
        if (mbd.isSingleton()) {
            sharedInstance = getSingleton(beanName, new ObjectFactory<Object>() {
                @Override
                public Object getObject() throws BeansException {
                    try {
                        return createBean(beanName, mbd, args);
                    }
                    catch (BeansException ex) {
                        
                        //如果出现异常，销毁可能实例化的bean
                        destroySingleton(beanName);
                        throw ex;
                    }
                }
            });
            bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
        }

        else if (mbd.isPrototype()) {
            // It's a prototype -> create a new instance.
            Object prototypeInstance = null;
            try {
                //在threadlocal中设置这个bean名称标志位，防止同意线程中重复创建此bean
                beforePrototypeCreation(beanName);
                //创建多实例bean
                prototypeInstance = createBean(beanName, mbd, args);
            }
            finally {
                //清除标志位
                afterPrototypeCreation(beanName);
            }
            bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
        }

        else {
            //scope为request session等的场景
            String scopeName = mbd.getScope();
            final Scope scope = this.scopes.get(scopeName);
            if (scope == null) {
                throw new IllegalStateException("No Scope registered for scope name '" + scopeName + "'");
            }
            try {
                Object scopedInstance = scope.get(beanName, new ObjectFactory<Object>() {
                    @Override
                    public Object getObject() throws BeansException {
                        beforePrototypeCreation(beanName);
                        try {
                            return createBean(beanName, mbd, args);
                        }
                        finally {
                            afterPrototypeCreation(beanName);
                        }
                    }
                });
                bean = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
            }
            catch (IllegalStateException ex) {
                throw new BeanCreationException(beanName,
                        "Scope '" + scopeName + "' is not active for the current thread; consider " +
                        "defining a scoped proxy for this bean if you intend to refer to it from a singleton",
                        ex);
            }
        }
    }
    catch (BeansException ex) {
        //如果创建某个bean失败，则会清除已创建的bean集合
        cleanupAfterBeanCreationFailure(beanName);
        throw ex;
    }
}

// Check if required type matches the type of the actual bean instance.
if (requiredType != null && bean != null && !requiredType.isAssignableFrom(bean.getClass())) {
    try {
        return getTypeConverter().convertIfNecessary(bean, requiredType);
    }
    catch (TypeMismatchException ex) {
        if (logger.isDebugEnabled()) {
            logger.debug("Failed to convert bean '" + name + "' to required type [" +
                    ClassUtils.getQualifiedName(requiredType) + "]", ex);
        }
        throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
    }
}
return (T) bean;
}
```
如果当前bean是Factory Bean，getBean的时候不直接返回当前bean，而是调用其工厂方法获取Bean实例，下面是getObjectForBeanInstance()源码
```java 
protected Object getObjectForBeanInstance(
        Object beanInstance, String name, String beanName, RootBeanDefinition mbd) {

    // factorybean的名称以“&”开头，如果这个bean以&开头，但是不是factorybean则会报错
    if (BeanFactoryUtils.isFactoryDereference(name) && !(beanInstance instanceof FactoryBean)) {
        throw new BeanIsNotAFactoryException(transformedBeanName(name), beanInstance.getClass());
    }

    
    //如果bean不是factorybean，或者以&开头，直接返回
    if (!(beanInstance instanceof FactoryBean) || BeanFactoryUtils.isFactoryDereference(name)) {
        return beanInstance;
    }

    Object object = null;
    if (mbd == null) {
        //从FactoryBeanRegistrySupport的单例map中获取实例
        object = getCachedObjectForFactoryBean(beanName);
    }
    if (object == null) {
        // Return bean instance from factory.
        FactoryBean<?> factory = (FactoryBean<?>) beanInstance;
        // Caches object obtained from FactoryBean if it is a singleton.
        if (mbd == null && containsBeanDefinition(beanName)) {
            mbd = getMergedLocalBeanDefinition(beanName);
        }
        boolean synthetic = (mbd != null && mbd.isSynthetic());
        //从FactoryBean中获取实例
        object = getObjectFromFactoryBean(factory, beanName, !synthetic);
    }
    return object;
}

```
getBean() 流程总结：
    1. 对bean名称进行解析，如果以&作为前缀，将&去掉来获取真实bean名称
    2. 尝试从单例缓存池获取bean
    3. 如果从单例缓存池获取不到bean,获取父工厂递归执行getBean()
    4. 检查bean依赖，如果这个bean依赖其他bean，需要先把依赖的bean创建注册，然后运用递归解决循环依赖的问题
    5. 根据不同的作用域创建不同的bean
       1. singleton：getSingleton(beanName,objectFactory),如果出现异常，销毁可能初始化的bean
       2. 在threadlocal设置标志位然后创建bean
       3. request session等。。。
    6. 

## createBean()的流程

Bean可能是多种多样的，有代理Bean、Factory Bean、自定义Bean等，这些Bean的加载，执行顺序以及结果等都是不同的。Spring是如何支持创建多种多样的bean的？  
在AbstractAutowireCapableBeanFactory中的creatBean()方法源码中：  
```java
protected Object createBean(String beanName, RootBeanDefinition mbd, Object[] args) throws BeanCreationException {
    if (logger.isDebugEnabled()) {
        logger.debug("Creating instance of bean '" + beanName + "'");
    }
    RootBeanDefinition mbdToUse = mbd;

    //先从beandefinition获取beanclass为bean实例化做准备，复制出来一个新的beandefinition，防止多线程修改原beanDefinition
    Class<?> resolvedClass = resolveBeanClass(mbd, beanName);
    if (resolvedClass != null && !mbd.hasBeanClass() && mbd.getBeanClassName() != null) {
        mbdToUse = new RootBeanDefinition(mbd);
        mbdToUse.setBeanClass(resolvedClass);
    }

    // Prepare method overrides.
    try {
        mbdToUse.prepareMethodOverrides();
    }
    catch (BeanDefinitionValidationException ex) {
        throw new BeanDefinitionStoreException(mbdToUse.getResourceDescription(),
                beanName, "Validation of method overrides failed", ex);
    }

    try {
        // Give BeanPostProcessors a chance to return a proxy instead of the target bean instance.
        //如果某些bean添加了实例化前的后置处理器，并且这些后置处理器已经实例化当前bean，则不用创建bean实例
        Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
        if (bean != null) {
            return bean;
        }
    }
    catch (Throwable ex) {
        throw new BeanCreationException(mbdToUse.getResourceDescription(), beanName,
                "BeanPostProcessor before instantiation of bean failed", ex);
    }
    //正式创建bean实例
    Object beanInstance = doCreateBean(beanName, mbdToUse, args);
    if (logger.isDebugEnabled()) {
        logger.debug("Finished creating instance of bean '" + beanName + "'");
    }
    return beanInstance;
}

```

接下来看看正常的bean是如何被创建的：
```java
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final Object[] args) {
    // Instantiate the bean.
    BeanWrapper instanceWrapper = null;
    if (mbd.isSingleton()) {
        //如果这个bean是单例的，并且在单例map中已经含有名为beanName的Bean实例，则不需要实例化当前bean，如果单例map中不存在当前bean，实例化bean并将其添加到单例map中
        instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
    }
    if (instanceWrapper == null) {
        //创建bean实例
        instanceWrapper = createBeanInstance(beanName, mbd, args);
    }
    final Object bean = (instanceWrapper != null ? instanceWrapper.getWrappedInstance() : null);
    Class<?> beanType = (instanceWrapper != null ? instanceWrapper.getWrappedClass() : null);

    //调用这个bean的 merge bean来定义后置处理器方法， 例如检查自动注入时的成员变量
    synchronized (mbd.postProcessingLock) {
        if (!mbd.postProcessed) {
            applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
            mbd.postProcessed = true;
        }
    }

    // Eagerly cache singletons to be able to resolve circular references
    // even when triggered by lifecycle interfaces like BeanFactoryAware.
    //尽早创建bean实例，并解决循环依赖的问题
    boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
            isSingletonCurrentlyInCreation(beanName));
    if (earlySingletonExposure) {
        if (logger.isDebugEnabled()) {
            logger.debug("Eagerly caching bean '" + beanName +
                    "' to allow for resolving potential circular references");
        }
        addSingletonFactory(beanName, new ObjectFactory<Object>() {
            @Override
            public Object getObject() throws BeansException {
                return getEarlyBeanReference(beanName, mbd, bean);
            }
        });
    }

    // Initialize the bean instance.
    //此时bean已经被创建，但其属性并没有被赋值，所以要准备当前bean 中与属性相关的数据
    Object exposedObject = bean;
    try {
        populateBean(beanName, mbd, instanceWrapper);
        if (exposedObject != null) {
            exposedObject = initializeBean(beanName, exposedObject, mbd);
        }
    }
    catch (Throwable ex) {
        if (ex instanceof BeanCreationException && beanName.equals(((BeanCreationException) ex).getBeanName())) {
            throw (BeanCreationException) ex;
        }
        else {
            throw new BeanCreationException(mbd.getResourceDescription(), beanName, "Initialization of bean failed", ex);
        }
    }

    if (earlySingletonExposure) {
        Object earlySingletonReference = getSingleton(beanName, false);
        if (earlySingletonReference != null) {
            if (exposedObject == bean) {
                exposedObject = earlySingletonReference;
            }
            else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
                String[] dependentBeans = getDependentBeans(beanName);
                Set<String> actualDependentBeans = new LinkedHashSet<String>(dependentBeans.length);
                for (String dependentBean : dependentBeans) {
                    if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
                        actualDependentBeans.add(dependentBean);
                    }
                }
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

    // Register bean as disposable.
    //如果当前bean生命周期不是多例（包含单例，request等范围的bean），也就是说需要spring管理bean的生命周期，会把bean的destory方法注册到spring上下文中
    try {
        registerDisposableBeanIfNecessary(beanName, bean, mbd);
    }
    catch (BeanDefinitionValidationException ex) {
        throw new BeanCreationException(mbd.getResourceDescription(), beanName, "Invalid destruction signature", ex);
    }

    return exposedObject;
}

```


