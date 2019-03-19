## Spring IOC容器

### IOC容器加载bean步骤：定位 -> 解析 -> 注册 

> IOC容器 -> `Map<String, BeanDefinition>`
>
> 参考资料：https://www.cnblogs.com/yuanfuqiang/p/5834496.html

`org.springframework.context.support.AbstractApplicationContext#refresh`

```java
@Override
public void refresh() throws BeansException, IllegalStateException {
    synchronized (this.startupShutdownMonitor) {
        // Prepare this context for refreshing.
        // 调用容器准备刷新的方法，同时给容器设置同步标识
        prepareRefresh();

        // Tell the subclass to refresh the internal bean factory.
        // 告诉子类启动refreshBeanFactory()方法，Bean定义资源文件的载入从子类的refreshBeanFactory()方法启动
        ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

        // Prepare the bean factory for use in this context.
        // 为BeanFactory配置容器特性，例如类加载器、事件处理器等
        prepareBeanFactory(beanFactory);

        try {
            // Allows post-processing of the bean factory in context subclasses.
            // 为容器的某些子类指定特殊的BeanPost事件处理器
            postProcessBeanFactory(beanFactory);

            // Invoke factory processors registered as beans in the context.
            // 调用所有注册的BeanFactoryPostProcessor的Bean
            invokeBeanFactoryPostProcessors(beanFactory);

            // Register bean processors that intercept bean creation.
            // 为BeanFactory注册BeanPost事件处理器
            // BeanPostProcessor是Bean后置处理器，用于监听容器触发的事件
            registerBeanPostProcessors(beanFactory);

            // Initialize message source for this context.
            // 初始化信息源，和国际化有关
            initMessageSource();

            // Initialize event multicaster for this context.
            // 初始化容器事件传播器
            initApplicationEventMulticaster();

            // Initialize other special beans in specific context subclasses.
            // 调用子类的某些特殊Bean初始化方法
            onRefresh();

            // Check for listener beans and register them.
            // 为事件传播器注册事件监听器
            registerListeners();

            // Instantiate all remaining (non-lazy-init) singletons.
            // 初始化所有剩余的单态Bean
            finishBeanFactoryInitialization(beanFactory);

            // Last step: publish corresponding event.
            // 初始化容器的生命周期事件处理器，并发布容器的生命周期事件 
            finishRefresh();
        }

        catch (BeansException ex) {
            if (logger.isWarnEnabled()) {
                logger.warn("Exception encountered during context initialization - " +
                            "cancelling refresh attempt: " + ex);
            }

            // Destroy already created singletons to avoid dangling resources.
            destroyBeans();

            // Reset 'active' flag.
            // 取消refresh操作，重置容器的同步标识
            cancelRefresh(ex);

            // Propagate exception to caller.
            throw ex;
        }

        finally {
            // Reset common introspection caches in Spring's core, since we
            // might not ever need metadata for singleton beans anymore...
            resetCommonCaches();
        }
    }
}
```

​	`refresh()`方法主要为IOC容器Bean的生命周期管理提供条件，IOC容器载入Bean定义资源文件从其子类容器的`refreshBeanFactory()`方法启动，所以整个refresh()中`ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();`这句代码后的都是注册容器的信息源和生命周期事件，载入过程就是从这句代码开始：

```java
protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
   // 这里使用了委派设计模式，父类定义了抽象的refreshBeanFactory()方法，具体实现调用子类容器的refreshBeanFactory()方法
   refreshBeanFactory();
   return getBeanFactory();
}
```

具体实现`refreshBeanFactory()`方法的是`org.springframework.context.support.AbstractRefreshableApplicationContext#refreshBeanFactory`

```java
@Override
protected final void refreshBeanFactory() throws BeansException {
   if (hasBeanFactory()) {
      destroyBeans();
      closeBeanFactory();
   }
   try {
      DefaultListableBeanFactory beanFactory = createBeanFactory();
      beanFactory.setSerializationId(getId());
      customizeBeanFactory(beanFactory);
      loadBeanDefinitions(beanFactory);
      synchronized (this.beanFactoryMonitor) {
         this.beanFactory = beanFactory;
      }
   }
   catch (IOException ex) {
      throw new ApplicationContextException("I/O error parsing bean definition source for " + getDisplayName(), ex);
   }
}
```

​	该方法加上`final`，也就是说此方法不可被重写。IOC容器的初始就是发生在这里，第一步是判断有无现有的工厂，有的话便销毁，否则就创建一个默认的bean工厂，也就是`DefaultListableBeanFactory`。	                     

​      `AbstractRefreshableApplicationContext`只定义了抽象的`loadBeanDefinitions()`方法，容器真正调用的是其子类`AbstractXmlApplicationContext#loadBeanDefinitions`，主要源码如下：

```java
@Override
protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) throws BeansException, IOException {
   // Create a new XmlBeanDefinitionReader for the given BeanFactory.
   XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);

   // Configure the bean definition reader with this context's
   // resource loading environment.
   beanDefinitionReader.setEnvironment(this.getEnvironment());
   beanDefinitionReader.setResourceLoader(this);
   beanDefinitionReader.setEntityResolver(new ResourceEntityResolver(this));

   // Allow a subclass to provide custom initialization of the reader,
   // then proceed with actually loading the bean definitions.
   initBeanDefinitionReader(beanDefinitionReader);
   loadBeanDefinitions(beanDefinitionReader);
}
```

​	第一行首先定义一个reader，追踪源码发现最终读取过程也正是由`XmlBeanDefinitionReader#loadBeanDefinitions`完成的，具体源码如下：

```java
public int loadBeanDefinitions(EncodedResource encodedResource) throws BeanDefinitionStoreException {
   Assert.notNull(encodedResource, "EncodedResource must not be null");
   if (logger.isTraceEnabled()) {
      logger.trace("Loading XML bean definitions from " + encodedResource);
   }

   Set<EncodedResource> currentResources = this.resourcesCurrentlyBeingLoaded.get();
   if (currentResources == null) {
      currentResources = new HashSet<>(4);
      this.resourcesCurrentlyBeingLoaded.set(currentResources);
   }
   if (!currentResources.add(encodedResource)) {
      throw new BeanDefinitionStoreException(
            "Detected cyclic loading of " + encodedResource + " - check your import definitions!");
   }
   try {
      InputStream inputStream = encodedResource.getResource().getInputStream();
      try {
         InputSource inputSource = new InputSource(inputStream);
         if (encodedResource.getEncoding() != null) {
            inputSource.setEncoding(encodedResource.getEncoding());
         }
         // 执行加载Bean方法 
         return doLoadBeanDefinitions(inputSource, encodedResource.getResource());
      }
      finally {
         inputStream.close();
      }
   }
   catch (IOException ex) {
      throw new BeanDefinitionStoreException(
            "IOException parsing XML document from " + encodedResource.getResource(), ex);
   }
   finally {
      currentResources.remove(encodedResource);
      if (currentResources.isEmpty()) {
         this.resourcesCurrentlyBeingLoaded.remove();
      }
   }
}

/**
	 * Actually load bean definitions from the specified XML file.
	 * @param inputSource the SAX InputSource to read from
	 * @param resource the resource descriptor for the XML file
	 * @return the number of bean definitions found
	 * @throws BeanDefinitionStoreException in case of loading or parsing errors
	 * @see #doLoadDocument
	 * @see #registerBeanDefinitions
	 */
protected int doLoadBeanDefinitions(InputSource inputSource, Resource resource)
    throws BeanDefinitionStoreException {

    try {
        Document doc = doLoadDocument(inputSource, resource);
        int count = registerBeanDefinitions(doc, resource);
        if (logger.isDebugEnabled()) {
            logger.debug("Loaded " + count + " bean definitions from " + resource);
        }
        return count;
    }
    catch (BeanDefinitionStoreException ex) {
        throw ex;
    }
    catch (SAXParseException ex) {
        throw new XmlBeanDefinitionStoreException(resource.getDescription(),
                                                  "Line " + ex.getLineNumber() + " in XML document from " + resource + " is invalid", ex);
    }
    catch (SAXException ex) {
        throw new XmlBeanDefinitionStoreException(resource.getDescription(),
                                                  "XML document from " + resource + " is invalid", ex);
    }
    catch (ParserConfigurationException ex) {
        throw new BeanDefinitionStoreException(resource.getDescription(),
                                               "Parser configuration exception parsing XML from " + resource, ex);
    }
    catch (IOException ex) {
        throw new BeanDefinitionStoreException(resource.getDescription(),
                                               "IOException parsing XML document from " + resource, ex);
    }
    catch (Throwable ex) {
        throw new BeanDefinitionStoreException(resource.getDescription(),
                                               "Unexpected exception parsing XML document from " + resource, ex);
    }
}
```

​	可以看到，Spring采用documentLoader将资源转换成了`Document`。该方法从特定XML文件中实际载入Bean定义资源的方法，接下来调用`registerBeanDefinitions()`启动Spring IOC容器对Bean定义的解析过程，`registerBeanDefinitions()`源码如下：

```java
public int registerBeanDefinitions(Document doc, Resource resource) throws BeanDefinitionStoreException {
   // 得到BeanDefinitionDocumentReader来对xml格式的BeanDefinition解析  
   BeanDefinitionDocumentReader documentReader = createBeanDefinitionDocumentReader();
   // 获得容器中注册的Bean数量  
   int countBefore = getRegistry().getBeanDefinitionCount();
   // 解析过程入口，这里使用了委派模式，BeanDefinitionDocumentReader只是个接口，
   // 具体的解析实现过程有实现类DefaultBeanDefinitionDocumentReader完成 
   documentReader.registerBeanDefinitions(doc, createReaderContext(resource));
   // 统计解析的Bean数量  
   return getRegistry().getBeanDefinitionCount() - countBefore;
}

// 创建BeanDefinitionDocumentReader对象，解析Document对象 
protected BeanDefinitionDocumentReader createBeanDefinitionDocumentReader() {
    return BeanUtils.instantiateClass(this.documentReaderClass);
}
```

​	`DefaultBeanDefinitionDocumentReader`解析`Document`的代码如下：

```java
// DefaultBeanDefinitionDocumentReader#doRegisterBeanDefinitions
protected void doRegisterBeanDefinitions(Element root) {
   // Any nested <beans> elements will cause recursion in this method. In
   // order to propagate and preserve <beans> default-* attributes correctly,
   // keep track of the current (parent) delegate, which may be null. Create
   // the new (child) delegate with a reference to the parent for fallback purposes,
   // then ultimately reset this.delegate back to its original (parent) reference.
   // this behavior emulates a stack of delegates without actually necessitating one.
   BeanDefinitionParserDelegate parent = this.delegate;
   this.delegate = createDelegate(getReaderContext(), root, parent);

   if (this.delegate.isDefaultNamespace(root)) {
      String profileSpec = root.getAttribute(PROFILE_ATTRIBUTE);
      if (StringUtils.hasText(profileSpec)) {
         String[] specifiedProfiles = StringUtils.tokenizeToStringArray(
               profileSpec, BeanDefinitionParserDelegate.MULTI_VALUE_ATTRIBUTE_DELIMITERS);
         // We cannot use Profiles.of(...) since profile expressions are not supported
         // in XML config. See SPR-12458 for details.
         if (!getReaderContext().getEnvironment().acceptsProfiles(specifiedProfiles)) {
            if (logger.isDebugEnabled()) {
               logger.debug("Skipped XML bean definition file due to specified profiles [" + profileSpec +
                     "] not matching: " + getReaderContext().getResource());
            }
            return;
         }
      }
   }

   // 在解析Bean定义之前，进行自定义的解析，增强解析过程的可扩展性  
   preProcessXml(root);
   // 从Document的根元素开始进行Bean定义的Document对象 
   parseBeanDefinitions(root, this.delegate);
   // 在解析Bean定义之后，进行自定义的解析，增加解析过程的可扩展性
   postProcessXml(root);

   this.delegate = parent;
}
```

​	分析源码发现，Spring中对<property>、元素中配置的Array、List、Set、Map、Prop/<bean>等各种集合子元素都是在`org.springframework.beans.factory.xml.BeanDefinitionParserDelegate`中实现的，其源码片段：

```java
// 解析元素入口
// 1
@Nullable
public BeanDefinitionHolder parseBeanDefinitionElement(Element ele) {
   return parseBeanDefinitionElement(ele, null);
}

// 2
@Nullable
public BeanDefinitionHolder parseBeanDefinitionElement(Element ele, @Nullable BeanDefinition containingBean) {
    //获取<Bean>元素中的id属性值 
    String id = ele.getAttribute(ID_ATTRIBUTE);
    //获取<Bean>元素中的name属性值 
    String nameAttr = ele.getAttribute(NAME_ATTRIBUTE);
    //获取<Bean>元素中的alias属性值 
    List<String> aliases = new ArrayList<>();
    if (StringUtils.hasLength(nameAttr)) {
        String[] nameArr = StringUtils.tokenizeToStringArray(nameAttr, MULTI_VALUE_ATTRIBUTE_DELIMITERS);
        aliases.addAll(Arrays.asList(nameArr));
    }

    String beanName = id;
    //如果<Bean>元素中没有配置id属性时，将别名中的第一个值赋值给beanName 
    if (!StringUtils.hasText(beanName) && !aliases.isEmpty()) {
        beanName = aliases.remove(0);
        if (logger.isTraceEnabled()) {
            logger.trace("No XML 'id' specified - using '" + beanName +
                         "' as bean name and " + aliases + " as aliases");
        }
    }

    if (containingBean == null) {
        checkNameUniqueness(beanName, aliases, ele);
    }

    // 执行具体的解析，并封装成Beandefinition
    // 详细对<Bean>元素中配置的Bean定义进行解析的地方 
    AbstractBeanDefinition beanDefinition = parseBeanDefinitionElement(ele, beanName, containingBean);
    if (beanDefinition != null) {
        if (!StringUtils.hasText(beanName)) {
            try {
                if (containingBean != null) {
                    beanName = BeanDefinitionReaderUtils.generateBeanName(
                        beanDefinition, this.readerContext.getRegistry(), true);
                }
                else {
                    beanName = this.readerContext.generateBeanName(beanDefinition);
                    // Register an alias for the plain bean class name, if still possible,
                    // if the generator returned the class name plus a suffix.
                    // This is expected for Spring 1.2/2.0 backwards compatibility.
                    String beanClassName = beanDefinition.getBeanClassName();
                    if (beanClassName != null &&
                        beanName.startsWith(beanClassName) && beanName.length() > beanClassName.length() &&
                        !this.readerContext.getRegistry().isBeanNameInUse(beanClassName)) {
                        aliases.add(beanClassName);
                    }
                }
                if (logger.isTraceEnabled()) {
                    logger.trace("Neither XML 'id' nor 'name' specified - " +
                                 "using generated bean name [" + beanName + "]");
                }
            }
            catch (Exception ex) {
                error(ex.getMessage(), ele);
                return null;
            }
        }
        String[] aliasesArray = StringUtils.toStringArray(aliases);
        return new BeanDefinitionHolder(beanDefinition, beanName, aliasesArray);
    }

    return null;
}

// 3
@Nullable
public AbstractBeanDefinition parseBeanDefinitionElement(
    Element ele, String beanName, @Nullable BeanDefinition containingBean) {

    this.parseState.push(new BeanEntry(beanName));
    //这里只读取<Bean>元素中配置的class名字，然后载入到BeanDefinition中去 
    //只是记录配置的class名字，不做实例化，对象的实例化在依赖注入时完成
    String className = null;
    if (ele.hasAttribute(CLASS_ATTRIBUTE)) {
        className = ele.getAttribute(CLASS_ATTRIBUTE).trim();
    }
    //如果<Bean>元素中配置了parent属性，则获取parent属性的值 
    String parent = null;
    if (ele.hasAttribute(PARENT_ATTRIBUTE)) {
        parent = ele.getAttribute(PARENT_ATTRIBUTE);
    }

    try {
        //根据<Bean>元素配置的class名称和parent属性值创建BeanDefinition 
        //为载入Bean定义信息做准备 
        AbstractBeanDefinition bd = createBeanDefinition(className, parent);
		//对当前的<Bean>元素中配置的一些属性进行解析和设置，如配置的单态(singleton)属性等	
        parseBeanDefinitionAttributes(ele, beanName, containingBean, bd);
        bd.setDescription(DomUtils.getChildElementValueByTagName(ele, DESCRIPTION_ELEMENT));
		
        //对<Bean>元素的meta(元信息)属性解析
        parseMetaElements(ele, bd);
        //对<Bean>元素的lookup-method属性解析
        parseLookupOverrideSubElements(ele, bd.getMethodOverrides());
        //对<Bean>元素的replaced-method属性解析
        parseReplacedMethodSubElements(ele, bd.getMethodOverrides());

        //解析<Bean>元素的构造方法设置
        parseConstructorArgElements(ele, bd);
        //解析<Bean>元素的<property>设置
        parsePropertyElements(ele, bd);
        //解析<Bean>元素的qualifier属性 
        parseQualifierElements(ele, bd);

        //为当前解析的Bean设置所需的资源和依赖对象 
        bd.setResource(this.readerContext.getResource());
        bd.setSource(extractSource(ele));

        return bd;
    }
    catch (ClassNotFoundException ex) {
        error("Bean class [" + className + "] not found", ele, ex);
    }
    catch (NoClassDefFoundError err) {
        error("Class that bean class [" + className + "] depends on not found", ele, err);
    }
    catch (Throwable ex) {
        error("Unexpected failure during bean definition parsing", ele, ex);
    }
    finally {
        this.parseState.pop();
    }

    return null;
}
```

​	上面方法中对一些配置如元信息（meta）、qualifier等的解析，我们在Spring配置中使用的也不多。我们在使用<bean>元素时，使用最多的还是<property>属性，接下来还是具体看看<property>怎么解析的：

```java
// BeanDefinitionParserDelegate
/**
 * Parse property sub-elements of the given bean element.
 */
public void parsePropertyElements(Element beanEle, BeanDefinition bd) {
   NodeList nl = beanEle.getChildNodes();
   for (int i = 0; i < nl.getLength(); i++) {
      Node node = nl.item(i);
      if (isCandidateElement(node) && nodeNameEquals(node, PROPERTY_ELEMENT)) {
         parsePropertyElement((Element) node, bd);
      }
   }
}

	/**
	 * Parse a property element.
	 */
public void parsePropertyElement(Element ele, BeanDefinition bd) {
    String propertyName = ele.getAttribute(NAME_ATTRIBUTE);
    if (!StringUtils.hasLength(propertyName)) {
        error("Tag 'property' must have a 'name' attribute", ele);
        return;
    }
    this.parseState.push(new PropertyEntry(propertyName));
    try {
        if (bd.getPropertyValues().contains(propertyName)) {
            error("Multiple 'property' definitions for property '" + propertyName + "'", ele);
            return;
        }
        Object val = parsePropertyValue(ele, bd, propertyName);
        PropertyValue pv = new PropertyValue(propertyName, val);
        parseMetaElements(ele, pv);
        pv.setSource(extractSource(ele));
        bd.getPropertyValues().addPropertyValue(pv);
    }
    finally {
        this.parseState.pop();
    }
}

@Nullable
public Object parsePropertyValue(Element ele, BeanDefinition bd, @Nullable String propertyName) {
    String elementName = (propertyName != null ?
                          "<property> element for property '" + propertyName + "'" :
                          "<constructor-arg> element");

    // Should only have one child element: ref, value, list, etc.
    NodeList nl = ele.getChildNodes();
    Element subElement = null;
    for (int i = 0; i < nl.getLength(); i++) {
        Node node = nl.item(i);
        if (node instanceof Element && !nodeNameEquals(node, DESCRIPTION_ELEMENT) &&
            !nodeNameEquals(node, META_ELEMENT)) {
            // Child element is what we're looking for.
            if (subElement != null) {
                error(elementName + " must not contain more than one sub-element", ele);
            }
            else {
                subElement = (Element) node;
            }
        }
    }

    boolean hasRefAttribute = ele.hasAttribute(REF_ATTRIBUTE);
    boolean hasValueAttribute = ele.hasAttribute(VALUE_ATTRIBUTE);
    if ((hasRefAttribute && hasValueAttribute) ||
        ((hasRefAttribute || hasValueAttribute) && subElement != null)) {
        error(elementName +
              " is only allowed to contain either 'ref' attribute OR 'value' attribute OR sub-element", ele);
    }

    if (hasRefAttribute) {
        String refName = ele.getAttribute(REF_ATTRIBUTE);
        if (!StringUtils.hasText(refName)) {
            error(elementName + " contains empty 'ref' attribute", ele);
        }
        RuntimeBeanReference ref = new RuntimeBeanReference(refName);
        ref.setSource(extractSource(ele));
        return ref;
    }
    else if (hasValueAttribute) {
        TypedStringValue valueHolder = new TypedStringValue(ele.getAttribute(VALUE_ATTRIBUTE));
        valueHolder.setSource(extractSource(ele));
        return valueHolder;
    }
    else if (subElement != null) {
        return parsePropertySubElement(subElement, bd);
    }
    else {
        // Neither child element nor "ref" or "value" attribute found.
        error(elementName + " must specify a ref or value", ele);
        return null;
    }
}
@Nullable
public Object parsePropertySubElement(Element ele, @Nullable BeanDefinition bd, @Nullable String defaultValueType) {
    if (!isDefaultNamespace(ele)) {
        return parseNestedCustomElement(ele, bd);
    }
    else if (nodeNameEquals(ele, BEAN_ELEMENT)) {
        BeanDefinitionHolder nestedBd = parseBeanDefinitionElement(ele, bd);
        if (nestedBd != null) {
            nestedBd = decorateBeanDefinitionIfRequired(ele, nestedBd, bd);
        }
        return nestedBd;
    }
    else if (nodeNameEquals(ele, REF_ELEMENT)) {
        // A generic reference to any name of any bean.
        String refName = ele.getAttribute(BEAN_REF_ATTRIBUTE);
        boolean toParent = false;
        if (!StringUtils.hasLength(refName)) {
            // A reference to the id of another bean in a parent context.
            refName = ele.getAttribute(PARENT_REF_ATTRIBUTE);
            toParent = true;
            if (!StringUtils.hasLength(refName)) {
                error("'bean' or 'parent' is required for <ref> element", ele);
                return null;
            }
        }
        if (!StringUtils.hasText(refName)) {
            error("<ref> element contains empty target attribute", ele);
            return null;
        }
        RuntimeBeanReference ref = new RuntimeBeanReference(refName, toParent);
        ref.setSource(extractSource(ele));
        return ref;
    }
    else if (nodeNameEquals(ele, IDREF_ELEMENT)) {
        return parseIdRefElement(ele);
    }
    else if (nodeNameEquals(ele, VALUE_ELEMENT)) {
        return parseValueElement(ele, defaultValueType);
    }
    else if (nodeNameEquals(ele, NULL_ELEMENT)) {
        // It's a distinguished null value. Let's wrap it in a TypedStringValue
        // object in order to preserve the source location.
        TypedStringValue nullHolder = new TypedStringValue(null);
        nullHolder.setSource(extractSource(ele));
        return nullHolder;
    }
    else if (nodeNameEquals(ele, ARRAY_ELEMENT)) {
        return parseArrayElement(ele, bd); // array 元素
    }
    else if (nodeNameEquals(ele, LIST_ELEMENT)) {
        return parseListElement(ele, bd); // list 元素
    }
    else if (nodeNameEquals(ele, SET_ELEMENT)) {
        return parseSetElement(ele, bd); // set 元素
    }
    else if (nodeNameEquals(ele, MAP_ELEMENT)) {
        return parseMapElement(ele, bd); // map 元素
    }
    else if (nodeNameEquals(ele, PROPS_ELEMENT)) {
        return parsePropsElement(ele); // props 元素
    }
    else {
        error("Unknown property sub-element: [" + ele.getNodeName() + "]", ele);
        return null;
    }
}
```

> 注意：在解析<bean>元素的过程中并没有实例化bean对象，只是创建了bean对象的定义类`Beandefinition`，将bean元素的配置信息写入，当依赖注入时才实例化具体的bean对象。

### 依赖注入（实例化Bean）

依赖注入在以下两种情况下发生：

 * 用户第一次通过getBean方法向IOC容器索要bean时，IOC容器触发依赖注入；
 * 当用户在Bean定义资源中为<bean>元素配置了`lazy-init`属性。

`BeanFactory`接口定义了Spring IOC容器的基本规范，其中定义了几个getBean方法，就是用户向容器中获取Bean的方法，通过分析发现，他的具体实现是在`AbstractBeanFactory`中，具体代码如下：

```java
// AbstractBeanFactory#doGetBean
protected <T> T doGetBean(final String name, @Nullable final Class<T> requiredType,
			@Nullable final Object[] args, boolean typeCheckOnly) throws BeansException {

    //如果指定的是别名，将别名转换为规范的Bean名称 
    final String beanName = transformedBeanName(name);
    Object bean;

    //先从缓存中取是否已经有被创建过的单态类型的Bean，对于单态模式的Bean整个IoC容器中只创建一次，不需要重复创建 
    Object sharedInstance = getSingleton(beanName);
    if (sharedInstance != null && args == null) {
        if (logger.isTraceEnabled()) {
            if (isSingletonCurrentlyInCreation(beanName)) {
                logger.trace("Returning eagerly cached instance of singleton bean '" + beanName +
                             "' that is not fully initialized yet - a consequence of a circular reference");
            }
            else {
                logger.trace("Returning cached instance of singleton bean '" + beanName + "'");
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

        // Check if bean definition exists in this factory.
        BeanFactory parentBeanFactory = getParentBeanFactory();
        if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
            // Not found -> check parent.
            String nameToLookup = originalBeanName(name);
            if (parentBeanFactory instanceof AbstractBeanFactory) {
                return ((AbstractBeanFactory) parentBeanFactory).doGetBean(
                    nameToLookup, requiredType, args, typeCheckOnly);
            }
            else if (args != null) {
                // Delegation to parent with explicit args.
                return (T) parentBeanFactory.getBean(nameToLookup, args);
            }
            else if (requiredType != null) {
                // No args -> delegate to standard getBean method.
                return parentBeanFactory.getBean(nameToLookup, requiredType);
            }
            else {
                return (T) parentBeanFactory.getBean(nameToLookup);
            }
        }

        if (!typeCheckOnly) {
            markBeanAsCreated(beanName);
        }

        try {
            final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
            checkMergedBeanDefinition(mbd, beanName, args);

            // Guarantee initialization of beans that the current bean depends on.
            String[] dependsOn = mbd.getDependsOn();
            if (dependsOn != null) {
                for (String dep : dependsOn) {
                    if (isDependent(beanName, dep)) {
                        throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                                                        "Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
                    }
                    registerDependentBean(dep, beanName);
                    try {
                        getBean(dep);
                    }
                    catch (NoSuchBeanDefinitionException ex) {
                        throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                                                        "'" + beanName + "' depends on missing bean '" + dep + "'", ex);
                    }
                }
            }

            // Create bean instance.
            if (mbd.isSingleton()) {
                sharedInstance = getSingleton(beanName, () -> {
                    try {
                        return createBean(beanName, mbd, args);
                    }
                    catch (BeansException ex) {
                        // Explicitly remove instance from singleton cache: It might have been put there
                        // eagerly by the creation process, to allow for circular reference resolution.
                        // Also remove any beans that received a temporary reference to the bean.
                        destroySingleton(beanName);
                        throw ex;
                    }
                });
                bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
            }

            else if (mbd.isPrototype()) {
                // It's a prototype -> create a new instance.
                Object prototypeInstance = null;
                try {
                    beforePrototypeCreation(beanName);
                    prototypeInstance = createBean(beanName, mbd, args);
                }
                finally {
                    afterPrototypeCreation(beanName);
                }
                bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
            }

            else {
                String scopeName = mbd.getScope();
                final Scope scope = this.scopes.get(scopeName);
                if (scope == null) {
                    throw new IllegalStateException("No Scope registered for scope name '" + scopeName + "'");
                }
                try {
                    Object scopedInstance = scope.get(beanName, () -> {
                        beforePrototypeCreation(beanName);
                        try {
                            return createBean(beanName, mbd, args);
                        }
                        finally {
                            afterPrototypeCreation(beanName);
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
            cleanupAfterBeanCreationFailure(beanName);
            throw ex;
        }
    }

    // Check if required type matches the type of the actual bean instance.
    if (requiredType != null && !requiredType.isInstance(bean)) {
        try {
            T convertedBean = getTypeConverter().convertIfNecessary(bean, requiredType);
            if (convertedBean == null) {
                throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
            }
            return convertedBean;
        }
        catch (TypeMismatchException ex) {
            if (logger.isTraceEnabled()) {
                logger.trace("Failed to convert bean '" + name + "' to required type '" +
                             ClassUtils.getQualifiedName(requiredType) + "'", ex);
            }
            throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
        }
    }
    return (T) bean;
}
```

>  通过上面对向IoC容器获取Bean方法的分析，我们可以看到在Spring中，如果Bean定义的单态模式(Singleton)，则容器在创建之前先从缓存中查找，以确保整个容器中只存在一个实例对象。如果Bean定义的是原型模式(Prototype)，则容器每次都会创建一个新的实例对象。除此之外，Bean定义还可以扩展为指定其生命周期范围。	



​	上面的源码只是定义了根据Bean定义的模式，采取的不同创建Bean实例对象的策略，具体的Bean实例对象的创建过程由实现了BeanFactory接口的匿名内部类的createBean方法完成，BeanFactory使用委派模式，具体的Bean实例创建过程交由其实现类AbstractAutowireCapableBeanFactory完成，我们继续分析`AbstractAutowireCapableBeanFactory`的createBean方法的源码，理解其创建Bean实例的具体实现过程。

```java
// AbstractAutowireCapableBeanFactory
@Override
protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
      throws BeanCreationException {

   if (logger.isTraceEnabled()) {
      logger.trace("Creating instance of bean '" + beanName + "'");
   }
   RootBeanDefinition mbdToUse = mbd;

   // Make sure bean class is actually resolved at this point, and
   // clone the bean definition in case of a dynamically resolved Class
   // which cannot be stored in the shared merged bean definition.
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
      Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
      if (bean != null) {
         return bean;
      }
   }
   catch (Throwable ex) {
      throw new BeanCreationException(mbdToUse.getResourceDescription(), beanName,
            "BeanPostProcessor before instantiation of bean failed", ex);
   }

   try {
      Object beanInstance = doCreateBean(beanName, mbdToUse, args);
      if (logger.isTraceEnabled()) {
         logger.trace("Finished creating instance of bean '" + beanName + "'");
      }
      return beanInstance;
   }
   catch (BeanCreationException | ImplicitlyAppearedSingletonException ex) {
      // A previously detected exception with proper bean creation context already,
      // or illegal singleton state to be communicated up to DefaultSingletonBeanRegistry.
      throw ex;
   }
   catch (Throwable ex) {
      throw new BeanCreationException(
            mbdToUse.getResourceDescription(), beanName, "Unexpected exception during bean creation", ex);
   }
}
```

在`AbstractAutowireCapableBeanFactory#doCreateBean`中有两个重要的方法

 * createBeanInstance 根据指定的初始化策略，使用静态工厂、工厂方法或者容器的自动装配特性生成java实例对象；

 * populateBean 对Bean属性的依赖注入进行处理。

   ​	





## Spring多数据源配置

### 配置文件

在项目中新建配置文件 MultiDataSourceConfig.java

```java
package com.example.springweb.config;

import org.springframework.boot.jdbc.DataSourceBuilder;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Primary;

import javax.sql.DataSource;

/**
 * 多数据源配置
 *
 * @Author zhanglin
 * @Date 2019/2/19 22:50
 * @Verson 1.0
 */
@Configuration
public class MultiDataSourceConfig {

    @Bean
    @Primary // 设置主数据源
    public DataSource masterDataSource() {
        DataSourceBuilder dataSourceBuilder = DataSourceBuilder.create();
        DataSource dataSource = dataSourceBuilder
                .driverClassName("com.mysql.cj.jdbc.Driver")
                .url("jdbc:mysql://localhost:3306/test?serverTimezone=GMT&characterEncoding=utf-8")
                .username("root")
                .password("")
                .build();
        return dataSource;
    }

    @Bean
    public DataSource salveDataSource() {
        DataSourceBuilder dataSourceBuilder = DataSourceBuilder.create();
        DataSource dataSource = dataSourceBuilder
                .driverClassName("com.mysql.cj.jdbc.Driver")
                .url("jdbc:mysql://localhost:3306/test2?serverTimezone=GMT&characterEncoding=utf-8")
                .username("root")
                .password("")
                .build();
        return dataSource;
    }
}
```

### 多数据源的使用

* 手写Connection方式

  ```java
  package com.example.springweb.dao;
  
  import com.example.springweb.domain.User;
  import org.springframework.beans.factory.annotation.Qualifier;
  import org.springframework.stereotype.Repository;
  
  import javax.sql.DataSource;
  import java.sql.Connection;
  import java.sql.PreparedStatement;
  import java.sql.SQLException;
  
  /**
   * @Author zhanglin
   * @Date 2019/2/19 22:59
   * @Verson 1.0
   */
  @Repository
  public class UserDao {
  
      private final DataSource masterDataSource;
  
      private final DataSource salveDataSource;
  
      // 构造器方式注入
      public UserDao(@Qualifier("masterDataSource") DataSource masterDataSource,
                     @Qualifier("salveDataSource") DataSource salveDataSource) {
          this.masterDataSource = masterDataSource;
          this.salveDataSource = salveDataSource;
      }
  
      public boolean save(User user) {
          boolean result = false;
          Connection connection = null;
          try {
              connection = salveDataSource.getConnection();
              connection.setAutoCommit(false); // 设置非自动提交
              PreparedStatement statement = connection.prepareStatement(
                      "INSERT INTO user(username, password) VALUES (?, ?)");
              statement.setString(1, user.getUsername());
              statement.setString(2, user.getPassword());
              result = statement.executeUpdate() > 0;
          } catch (SQLException e) {
              e.printStackTrace();
          } finally {
              if (connection != null) {
                  try {
                      connection.commit(); // 在非自动提交的情况下，需要手动commit
                      connection.close();
                  } catch (SQLException e1) {
                      e1.printStackTrace();
                  }
              }
          }
  
          return result;
      }
  }
  ```

### 思考：当配合jpa、mybatis时如何使用多数据源？



## Spring定时任务

### 1、基于注解的定时任务

 * 在需要开启定时任务的类上面增加`@EnableScheduling`注解
 * 在需要开启定时任务的方法上面增加注解`@Scheduled(cron = "0/5 * * * * ?")`。







## Spring Boot 工程改造成多模块以及打包（Spring Boot版本：2.1.3.RELEASE）

### 改造成多模块：计划三个模块，web，domain，repository

* 新建web模块，把所有代码copy到web模块

* 新建repository模块，把web模块中关于repository中的代码copy到repository中，并在web模块中添加repository依赖

* 新建domain模块，把web模块中关于domain中的代码copy到repository中，并在repository添加domain的依赖。这样，在web模块中就间接依赖domain模块

  

### 打包

* 把主工程的build插件移动到web模块下的pom.xml文件中，如下

  ```xml
  <build>
      <plugins>
          <plugin>
              <groupId>org.springframework.boot</groupId>
              <artifactId>spring-boot-maven-plugin</artifactId>
              <configuration>
                  <!-- 主类（手工添加） -->
                  <mainClass>com.example.springweb.SpringwebApplication</mainClass>
              </configuration>
          </plugin>
      </plugins>
  </build>
  ```

* 打成jar包-

  * 执行打包命令：`mvn -Dmaven.test.skip -U clean package`
* 打成war包
  * 在web模块的pom.xml文件中添加`<packaging>war</packaging>`
  * 执行命令：`mvn -Dmaven.test.skip -U clean package`

### 启动

* 常规启动：在jar目录执行命令->`java -jar web-0.0.1-SNAPSHOT.jar --Dserver.port=0`

  > 注意： --Dserver.port=0，启动一个随机端口

* 目录方式启动（该方式解决老旧的jar不支持Spring Boot的情况）：

  * 将jar文件解压
  * 进入解压的目录，执行命令`java org.springframework.boot.loader.JarLauncher`
  * 如果是war包，则命令变为`java org.springframework.boot.loader.WarLauncher`





## Spring Boot Validation

### 1、使用自带的注解验证

* 1）在pom.xml中增加依赖

  ```xml
  <dependency>
     <groupId>org.springframework.boot</groupId>
     <artifactId>spring-boot-starter-validation</artifactId>
  </dependency>
  ```

 * 2）在实体类上增加注解。如@NotBlank，@Max等，已定义注解有如下：
   * @AssertFalse
   * @AssertTrue
   * @DecimalMax
   * @DecimalMin
   * @Digits
   * @Email
   * @Future
   * @FutureOrPresent
   * @Max
   * @Min
   * @Negative
   * @NegativeOrZero
   * @NotBlank
   * @NotEmpty
   * @NotNull
   * @Null
   * @Past
   * @PastOrPresent
   * @Pattern
   * @Positive
   * @PositiveOrZero
   * @Size
 * 3）在Controller的参数里面增加@Valid注解

### 2、自定义验证注解

 * 1） 创建注解 `@ValidCardNumber`

   ```java
   package com.example.sourcelearn.validation.constraints;
   
   import com.example.sourcelearn.validation.ValidCardNumberConstraintValidator;
   
   import javax.validation.Constraint;
   import javax.validation.Payload;
   import java.lang.annotation.Documented;
   import java.lang.annotation.Repeatable;
   import java.lang.annotation.Retention;
   import java.lang.annotation.Target;
   
   import static java.lang.annotation.ElementType.FIELD;
   import static java.lang.annotation.RetentionPolicy.RUNTIME;
   
   @Target({ FIELD }) // 字段
   @Retention(RUNTIME)
   @Repeatable(ValidCardNumber.List.class)
   @Documented
   @Constraint(validatedBy = { ValidCardNumberConstraintValidator.class })
   public @interface ValidCardNumber {
   
       String message() default "{javax.validation.constraints.card.number.message}";
   
       Class<?>[] groups() default { };
   
       Class<? extends Payload>[] payload() default { };
   
       @Target({ FIELD })
       @Retention(RUNTIME)
       @Documented
       public @interface List {
           ValidCardNumber[] value();
       }
   }
   ```

* 2) 创建注解实现类  `ValidCardNumberConstraintValidator`

  ```java
  package com.example.sourcelearn.validation;
  
  import com.example.sourcelearn.validation.constraints.ValidCardNumber;
  import org.apache.commons.lang3.ArrayUtils;
  import org.apache.commons.lang3.StringUtils;
  
  import javax.validation.ConstraintValidator;
  import javax.validation.ConstraintValidatorContext;
  
  /**
   * @Author zhanglin
   * @Date 2019/3/10 11:01
   * @Verson 1.0
   */
  public class ValidCardNumberConstraintValidator
          implements ConstraintValidator<ValidCardNumber, String> {
  
      @Override
      public void initialize(ValidCardNumber constraintAnnotation) {
  
      }
  
      /**
       * 验证规则：GUPAO-开头，数字结尾
       * @param value
       * @param context
       * @return
       */
      @Override
      public boolean isValid(String value, ConstraintValidatorContext context) {
          // 这里的分割尽量不要用String#split
          // 这里用的Apache的分割
          String[] parts = StringUtils.split(value, "-");
  
          if (ArrayUtils.getLength(parts) != 2) {
              return false;
          }
  
          boolean isValidPrex = "GUPAO".equals(parts[0]);
          boolean isValidSuffix = StringUtils.isNumeric(parts[1]);
  
          return isValidPrex && isValidSuffix;
      }
  }
  ```

* 3） 在POJO类的字段上面增加注解

  ```java
  package com.example.sourcelearn.domain;
  
  import com.example.sourcelearn.validation.constraints.ValidCardNumber;
  import lombok.Data;
  import lombok.ToString;
  
  import javax.validation.constraints.NotBlank;
  
  
  @Data
  @ToString
  public class User {
      private Integer id;
  
      //@NotNull(message = "姓名不能为空")
      @NotBlank(message = "姓名不能为空")
      private String name;
  
      // 必须以 GUPAO-开头，数字结尾
      @ValidCardNumber
      private String cardNumber;
  
      public User() {
          //this(1, "小马哥");
      }
  
      public User(Integer id, String name) {
          this.id = id;
          this.name = name;
      }
  
  }
  ```

* 国际化：因为spring boot validation用的是hibernate，所以在resource文件下增加国际化资源配置文件`ValidationMessages.properties`和`ValidationMessages.properties`

  ```properties
  javax.validation.constraints.card.number.message=cardNumer must start with GUPAO,and must end with number
  ```

  > 注意：这里的键为`@ValidCardNumber` 中定义的message







## Spring Cloud Config

> Spring boot 版本：2.1.3.RELEASE
>
> Spring cloud版本：Greenwich.SR1

### Server端

 * 1、加入maven依赖

   ```xml
   <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-web</artifactId>
   </dependency>
   <!-- config server依赖 -->
   <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-config-server</artifactId>
   </dependency>
   <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-actuator</artifactId>
   </dependency>
   ```

* 2、在启动类上增加注解`@EnableConfigServer`

* 3、在application.properties增加配置

  ```properties
  spring.application.name=config-server
  
  server.port=9100
  
  ## 暴露所有端点
  # Spring boot 2.0以后方式
  management.endpoints.web.exposure.include=*
  # Spring boot 2.0以前方式
  # management.security.enabled=true
  
  # git配置中心地址
  # 这里也可以用http
  spring.cloud.config.server.git.uri=\
    file:///C:/JAVA/Idea_Workspace/spring-cloud-learn/config
  ```

### Client端

 * 1、加入依赖

   ```xml
   <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-actuator</artifactId>
   </dependency>
   <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-web</artifactId>
   </dependency>
   <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-config-client</artifactId>
   </dependency>
   ```

* 2、增加`bootstrap.properties`文件，文件里面添加git配置

  ```properties
  ## 配置中心地址
  spring.cloud.config.uri=http://localhost:9100
  ## {application}
  spring.cloud.config.name=ht
  ## {profile}
  spring.cloud.config.profile=dev
  ## {label} 如果是git配置，则代表分支名称
  spring.cloud.config.label=master
  ```

  application.properties配置内容：

  ```properties
  spring.application.name=config-client
  
  server.port=9101
  
  # 暴露所有端点
  management.endpoints.web.exposure.include=*
  ```

* 3、`@RefreshScope`：当配置文件修改时，使该注解作用的类、方法下的配置项生效。如果不加该注解，则对应的配置不会时时生效。

  ```java
  @RestController
  @RefreshScope
  public class EchoController {
  
      @Value("${my.name}")
      private String myName;
  
      @GetMapping("/echo/env")
      public Map<String, Object> env() {
          Map<String, Object> result = new HashMap<>();
          result.put("name", myName);
  
          return result;
      }
  }
  ```

* 4、设置自动刷新配置功能

  ```java
  package com.example.springcloudconfigclient;
  
  import org.springframework.beans.factory.annotation.Autowired;
  import org.springframework.boot.SpringApplication;
  import org.springframework.boot.autoconfigure.SpringBootApplication;
  import org.springframework.cloud.context.refresh.ContextRefresher;
  import org.springframework.core.env.Environment;
  import org.springframework.scheduling.annotation.EnableScheduling;
  import org.springframework.scheduling.annotation.Scheduled;
  
  import java.util.Set;
  
  @SpringBootApplication
  @EnableScheduling // 启用定时任务，一定要加上该注解
  public class SpringCloudConfigClientApplication {
  
      // 采用构造方法注入ContextRefresher，刷新配置项
  	private final ContextRefresher contextRefresher;
  	private final Environment environment;
  
  	@Autowired
  	public SpringCloudConfigClientApplication(
  			ContextRefresher contextRefresher,
  			Environment environment) {
  		this.contextRefresher = contextRefresher;
  		this.environment = environment;
  	}
  
  	public static void main(String[] args) {
  		SpringApplication.run(SpringCloudConfigClientApplication.class, args);
  	}
  
  	/**
  	 * 采用定时任务自动刷新配置
  	 */
  	@Scheduled(cron = "0/5 * * * * ?")
  	public void autoRefreshConfig() {
  		Set<String> refreshedConfigs = contextRefresher.refresh();
  
  		if (!refreshedConfigs.isEmpty()) {
  			refreshedConfigs.forEach(property -> {
  				System.out.printf(
  						"Thread %s配置项已更新，Key %s，Value %s",
  						Thread.currentThread().getName(),
  						property,
  						environment.getProperty(property)
  				);
  			});
  		}
  	}
  }
  
  ```

  



## Spring Cloud服务注册中心

### Eureka

#### 服务端

 * 在启动程序上加上注解`@EnableEurekaServer`

   ```java
   package com.example.springcloudeurekaserver;
   
   import org.springframework.boot.SpringApplication;
   import org.springframework.boot.autoconfigure.SpringBootApplication;
   import org.springframework.cloud.netflix.eureka.server.EnableEurekaServer;
   
   @SpringBootApplication
   @EnableEurekaServer // eureka 服务注解
   public class SpringCloudEurekaServerApplication {
   
      public static void main(String[] args) {
         SpringApplication.run(SpringCloudEurekaServerApplication.class, args);
      }
   
   }
   ```

* 配置文件，如果是单机，需求取消服务自我注册

  ```properties
  ### 单机配置
  spring.application.name=eureka-server
  server.port=9090
  ## 取消服务器自我注册
  eureka.client.register-with-eureka=false
  eureka.client.fetch-registry=false
  ## eureka server服务URL
  eureka.client.service-url.defaultZone=\
    http://localhost:${server.port}/eureka/
  ```







### Nacos







## Java基础知识

### 定时任务表达式 cron详解

cron表达式格式：`* * * * * ?`

 * `*`秒（0~59） 例如：0/5表示每5秒
 * `*`分（0~59）
 * `*`时（0~23）
 * `*`日（0~31）的某天
 * `*`月（0~11）
 * `?`周（可填1~7，或SUN,MON,TUE,WED,THU,FRI,SAT）