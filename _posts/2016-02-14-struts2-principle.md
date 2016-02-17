---
title:  "Strus2工作原理"
date:   2016-02-17 17:40:00 +0800
tags: [Strus2]
---

# Strus2工作原理
---
以[Struts2Example](/assets/file/Struts2Example.rar)为例。

### 入口
---
Struts 2使用`Servlet`的`javax.servlet.Filter`拦截特性。对特定请求就行拦截处理直接对`ServletResponse`进行响应。典型配置是：

	  <filter>
		<filter-name>struts2</filter-name>
		<filter-class>org.apache.struts2.dispatcher.ng.filter.StrutsPrepareAndExecuteFilter</filter-class>
	  </filter>
	  
	  <filter-mapping>
		<filter-name>struts2</filter-name>
		<url-pattern>/*</url-pattern>
	  </filter-mapping>

等于一个标准的`Servlet`配置。拦截全部请求（`/*`）。

### 初始化
---
对于全局特性，比如默认`interceptor`，action配置，全局对象在服务器启动时（即`Filter初始化`）。
我们一步一步看：
初始化入口（`org.apache.struts2.dispatcher.ng.filter.StrutsPrepareAndExecuteFilter#init`）：


 	public void init(FilterConfig filterConfig) throws ServletException {
        InitOperations init = new InitOperations();
        try {
            FilterHostConfig config = new FilterHostConfig(filterConfig);
            init.initLogging(config);
            Dispatcher dispatcher = init.initDispatcher(config);
            init.initStaticContentLoader(config, dispatcher);

            prepare = new PrepareOperations(filterConfig.getServletContext(), dispatcher);
            execute = new ExecuteOperations(filterConfig.getServletContext(), dispatcher);
			this.excludedPatterns = init.buildExcludedPatternsList(dispatcher);

            postInit(dispatcher, filterConfig);
        } finally {
            init.cleanup();
        }

    }

主要步骤是创建全局对象`Dispatcher`。`Dispatcher`主要完成一些全局的对象操作转发（全局配置对象的管理对象`ConfigurationManager`的加载），准备工作（`prepare`）。
然后是`Dispatcher`对`ConfigurationManager`的初始化：

 	public Dispatcher org.apache.struts2.dispatcher.ng.InitOperations#initDispatcher( HostConfig filterConfig ) {
        Dispatcher dispatcher = createDispatcher(filterConfig);
        dispatcher.init();
        return dispatcher;
    }

	public void org.apache.struts2.dispatcher.Dispatcher#init() {

    	if (configurationManager == null) {
    		configurationManager = createConfigurationManager(BeanSelectionProvider.DEFAULT_BEAN_NAME);
    	}

        try {
            init_DefaultProperties(); // [1]
            init_TraditionalXmlConfigurations(); // [2]
            init_LegacyStrutsProperties(); // [3]
            init_CustomConfigurationProviders(); // [5]
            init_FilterInitParameters() ; // [6]
            init_AliasStandardObjects() ; // [7]

            Container container = init_PreloadConfiguration();
            container.inject(this);
            init_CheckConfigurationReloading(container);
            init_CheckWebLogicWorkaround(container);

            if (!dispatcherListeners.isEmpty()) {
                for (DispatcherListener l : dispatcherListeners) {
                    l.dispatcherInitialized(this);
                }
            }
        } catch (Exception ex) {
            if (LOG.isErrorEnabled())
                LOG.error("Dispatcher initialization failed", ex);
            throw new StrutsException(ex);
        }
    }

`Dispatcher#init`方法([1]-[7])加载（其实并不是立即加载，提供`ContainerProvider`缓加载机制）。比如：
 
- ` init_DefaultProperties()` ： 加载`org\apache\struts2\default.properties`，里面包含默认全局配置：编码，主题，是否开发模式等
- `init_TraditionalXmlConfigurations`：加载`"struts-default.xml,struts-plugin.xml,struts.xml"`，包含默认以及定制的拦截器，package，result等
- `init_LegacyStrutsProperties`：配置`Locale object`。
- `init_CustomConfigurationProviders`：加载定制`ConfigurationProvider`（使用`configProviders`配置）。
- `init_FilterInitParameters`：加载初始化参数
- `init_AliasStandardObjects`：初始化默认bean类型到`ContainerBuilder`。比如`ObjectFactory`等。
下面就行正式加载：

		private Container org.apache.struts2.dispatcher.Dispatcher#init_PreloadConfiguration() {
	        Configuration config = configurationManager.getConfiguration();
	        Container container = config.getContainer();
	
	        boolean reloadi18n = Boolean.valueOf(container.getInstance(String.class, StrutsConstants.STRUTS_I18N_RELOAD));
	        LocalizedTextUtil.setReloadBundles(reloadi18n);
	
	        return container;
	    }
		 public synchronized Configuration getConfiguration() {
	        if (configuration == null) {
	            setConfiguration(createConfiguration(defaultFrameworkBeanName));
	            try {
	                configuration.reloadContainer(getContainerProviders());
	            } catch (ConfigurationException e) {
	                setConfiguration(null);
	                throw new ConfigurationException("Unable to load configuration.", e);
	            }
	        } else {
	            conditionalReload();
	        }
	
	        return configuration;
	    }
		
			public synchronized List<PackageProvider> com.opensymphony.xwork2.config.impl.DefaultConfiguration#reloadContainer(List<ContainerProvider> providers) throws ConfigurationException {
	        packageContexts.clear();
	        loadedFileNames.clear();
	        List<PackageProvider> packageProviders = new ArrayList<PackageProvider>();
	
	        ContainerProperties props = new ContainerProperties();
	        ContainerBuilder builder = new ContainerBuilder();
	        for (final ContainerProvider containerProvider : providers)
	        {
	            containerProvider.init(this);
	        }
	      ...
	
	        ActionContext oldContext = ActionContext.getContext();
	        try {
	           ...
	            // Then process any package providers from the plugins
	            Set<String> packageProviderNames = container.getInstanceNames(PackageProvider.class);
	            if (packageProviderNames != null) {
	                for (String name : packageProviderNames) {
	                    PackageProvider provider = container.getInstance(PackageProvider.class, name);
	                    provider.init(this);
						provider.loadPackages();
                    	packageProviders.add(provider);
	                }
	            }
	
	            rebuildRuntimeConfiguration();
	        } finally {
	            if (oldContext == null) {
	                ActionContext.setContext(null);
	            }
	        }
	        return packageProviders;
	    }

3段代码分别调用前面加载的`ContainerProvider`的`init`或`loadPackages`（如果是也是`PackageProvider`子类的话）。我们看一下`init_TraditionalXmlConfigurations`所提供的`StrutsXmlConfigurationProvider`的加载过程，即我们最重要的框架的路由加载过程。
调用过程是`org.apache.struts2.config.StrutsXmlConfigurationProvider#loadPackages`->`com.opensymphony.xwork2.config.providers.XmlConfigurationProvider#loadPackages`->`com.opensymphony.xwork2.config.providers.XmlConfigurationProvider#addPackage`：
	
	 protected PackageConfig addPackage(Element packageElement) throws ConfigurationException {
        PackageConfig.Builder newPackage = buildPackageContext(packageElement);

        if (newPackage.isNeedsRefresh()) {
            return newPackage.build();
        }

        if (LOG.isDebugEnabled()) {
            LOG.debug("Loaded " + newPackage);
        }

        // add result types (and default result) to this package
        addResultTypes(newPackage, packageElement);

        // load the interceptors and interceptor stacks for this package
        loadInterceptors(newPackage, packageElement);

        // load the default interceptor reference for this package
        loadDefaultInterceptorRef(newPackage, packageElement);

        // load the default class ref for this package
        loadDefaultClassRef(newPackage, packageElement);

        // load the global result list for this package
        loadGlobalResults(newPackage, packageElement);

        // load the global exception handler list for this package
        loadGobalExceptionMappings(newPackage, packageElement);

        // get actions
        NodeList actionList = packageElement.getElementsByTagName("action");

        for (int i = 0; i < actionList.getLength(); i++) {
            Element actionElement = (Element) actionList.item(i);
            addAction(actionElement, newPackage);
        }

        // load the default action reference for this package
        loadDefaultActionRef(newPackage, packageElement);

        PackageConfig cfg = newPackage.build();
        configuration.addPackageConfig(cfg.getName(), cfg);
        return cfg;
    }

这段配置加载我们的包配置，即我们`struts.xml`中的：

 	 <package name="user" namespace="/User" extends="struts-default">
        <interceptors>
            <interceptor name="testFirst" class="com.mkyong.user.action.LogInterceptor"/>
        </interceptors>

        <action name="Login">
            <result>pages/login.jsp</result>
        </action>
        <action name="Welcome" class="com.mkyong.user.action.WelcomeUserAction">
            <result name="input">pages/login.jsp</result>
            <result name="SUCCESS">pages/welcome_user.jsp</result>
            <interceptor-ref name="testFirst"/>
            <interceptor-ref name="defaultStack"/>
        </action>
    </package>

`addPackage`依次加载到`PackageConfig`：

- `result-type`到`resultTypeConfigs`
- `interceptor`到`interceptorConfigs`
- `default-interceptor-ref`到`defaultInterceptorRef`
- `default-class-ref`
- `global-results`
- `global-exception-mappings`
- `action`到`actionConfigs`：

		protected void addAction(Element actionElement, PackageConfig.Builder packageContext) throws ConfigurationException {
	        String name = actionElement.getAttribute("name");
	        String className = actionElement.getAttribute("class");
	        String methodName = actionElement.getAttribute("method");
	        Location location = DomHelper.getLocationObject(actionElement);
	
	        if (location == null) {
	            if (LOG.isWarnEnabled()) {
	            LOG.warn("location null for " + className);
	            }
	        }
	        //methodName should be null if it's not set
	        methodName = (methodName.trim().length() > 0) ? methodName.trim() : null;
	
	       ...
	        Map<String, ResultConfig> results;
	        try {
	            results = buildResults(actionElement, packageContext);
	        } catch (ConfigurationException e) {
	            throw new ConfigurationException("Error building results for action " + name + " in namespace " + packageContext.getNamespace(), e, actionElement);
	        }
	
	        List<InterceptorMapping> interceptorList = buildInterceptorList(actionElement, packageContext);
	
	        List<ExceptionMappingConfig> exceptionMappings = buildExceptionMappings(actionElement, packageContext);
	
	        Set<String> allowedMethods = buildAllowedMethods(actionElement, packageContext);
	
	        ActionConfig actionConfig = new ActionConfig.Builder(packageContext.getName(), name, className)
	                .methodName(methodName)
	                .addResultConfigs(results)
	                .addInterceptors(interceptorList)
	                .addExceptionMappings(exceptionMappings)
	                .addParams(XmlHelper.getParams(actionElement))
	                .addAllowedMethod(allowedMethods)
	                .location(location)
	                .build();
	        packageContext.addActionConfig(name, actionConfig);
	
	        if (LOG.isDebugEnabled()) {
	            LOG.debug("Loaded " + (StringUtils.isNotEmpty(packageContext.getNamespace()) ? (packageContext.getNamespace() + "/") : "") + name + " in '" + packageContext.getName() + "' package:" + actionConfig);
	        }
	    }

加载`methodName`、`interceptorList`等到`ActionConfig`，再到`actionConfigs`。

回到前面的`com.opensymphony.xwork2.config.impl.DefaultConfiguration#reloadContainer`，之后执行`rebuildRuntimeConfiguration`，将`packageContexts`信息复制到`runtimeConfiguration`。之后我们就需要使用`runtimeConfiguration`获取`PackageConfig`信息。

### 准备（prepare）
每一个请求（`request`）发起，struts 2有2部操作：`prepare`,`execute`。先看`prepare`。
	
	public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain) throws IOException, ServletException {

        HttpServletRequest request = (HttpServletRequest) req;
        HttpServletResponse response = (HttpServletResponse) res;

        try {
            prepare.setEncodingAndLocale(request, response);
            prepare.createActionContext(request, response);
            prepare.assignDispatcherToThread();
			if ( excludedPatterns != null && prepare.isUrlExcluded(request, excludedPatterns)) {
				chain.doFilter(request, response);
			} else {
				request = prepare.wrapRequest(request);
				ActionMapping mapping = prepare.findActionMapping(request, response, true);
				if (mapping == null) {
					boolean handled = execute.executeStaticResourceRequest(request, response);
					if (!handled) {
						chain.doFilter(request, response);
					}
				} else {
					execute.executeAction(request, response, mapping);
				}
			}
        } finally {
            prepare.cleanupRequest(request);
        }
    }

`setEncodingAndLocale`就相对简单些，主要设置`encoding and locale`。在`org.apache.struts2.dispatcher.Dispatcher#prepare`中。
下面就是ActionContext：

	 public ActionContext createActionContext(HttpServletRequest request, HttpServletResponse response) {
	        ActionContext ctx;
	        Integer counter = 1;
	        Integer oldCounter = (Integer) request.getAttribute(CLEANUP_RECURSION_COUNTER);
	        if (oldCounter != null) {
	            counter = oldCounter + 1;
	        }
	        
	        ActionContext oldContext = ActionContext.getContext();
	        if (oldContext != null) {
	            // detected existing context, so we are probably in a forward
	            ctx = new ActionContext(new HashMap<String, Object>(oldContext.getContextMap()));
	        } else {
	            ValueStack stack = dispatcher.getContainer().getInstance(ValueStackFactory.class).createValueStack();
	            stack.getContext().putAll(dispatcher.createContextMap(request, response, null, servletContext));
	            ctx = new ActionContext(stack.getContext());
	        }
	        request.setAttribute(CLEANUP_RECURSION_COUNTER, counter);
	        ActionContext.setContext(ctx);
	        return ctx;
	    }

		public Map<String,Object> createContextMap(HttpServletRequest request, HttpServletResponse response,
            ActionMapping mapping, ServletContext context) {

	        // request map wrapping the http request objects
	        Map requestMap = new RequestMap(request);
	
	        // parameters map wrapping the http parameters.  ActionMapping parameters are now handled and applied separately
	        Map params = new HashMap(request.getParameterMap());
	
	        // session map wrapping the http session
	        Map session = new SessionMap(request);
	
	        // application map wrapping the ServletContext
	        Map application = new ApplicationMap(context);
	
	        Map<String,Object> extraContext = createContextMap(requestMap, params, session, application, request, response, context);
	
	        if (mapping != null) {
	            extraContext.put(ServletActionContext.ACTION_MAPPING, mapping);
	        }
	        return extraContext;
	    }	

在这里讲`request`,`request.getParameterMap`,`session`,`ServletContext`封装为map放到`ActionContext`对象的`context`对象中。然后放置到`ThreadLocal`变量中（Struts很多地方使用`ThreadLocal`传参）。
然后把全局的`Dispatcher`方法放到`ThreadLocal`变量中。
下面是`prepare.findActionMapping(request, response, true)`构造一个url查找的简单的`ActionMapping`对象。
### 实施（`execute`）：
执行步骤：
	
	 public void org.apache.struts2.dispatcher.ng.ExecuteOperations#executeAction(HttpServletRequest request, HttpServletResponse response, ActionMapping mapping) throws ServletException {
	        dispatcher.serviceAction(request, response, servletContext, mapping);
	   }

	public void serviceAction(HttpServletRequest request, HttpServletResponse response, ServletContext context,
                              ActionMapping mapping) throws ServletException {

        Map<String, Object> extraContext = createContextMap(request, response, mapping, context);

        // If there was a previous value stack, then create a new copy and pass it in to be used by the new Action
        ValueStack stack = (ValueStack) request.getAttribute(ServletActionContext.STRUTS_VALUESTACK_KEY);
        boolean nullStack = stack == null;
        if (nullStack) {
            ActionContext ctx = ActionContext.getContext();
            if (ctx != null) {
                stack = ctx.getValueStack();
            }
        }
        if (stack != null) {
            extraContext.put(ActionContext.VALUE_STACK, valueStackFactory.createValueStack(stack));
        }

        UtilTimerStack.push(timerKey);
        String namespace = mapping.getNamespace();
        String name = mapping.getName();
        String method = mapping.getMethod();

        Configuration config = configurationManager.getConfiguration();
        ActionProxy proxy = config.getContainer().getInstance(ActionProxyFactory.class).createActionProxy(
                namespace, name, method, extraContext, true, false);

        request.setAttribute(ServletActionContext.STRUTS_VALUESTACK_KEY, proxy.getInvocation().getStack());

        // if the ActionMapping says to go straight to a result, do it!
       
         proxy.execute();
      

        // If there was a previous value stack then set it back onto the request
        if (!nullStack) {
            request.setAttribute(ServletActionContext.STRUTS_VALUESTACK_KEY, stack);
        }
        ...
    }

基本步骤是构早一个前面一样的包含request，session,application的Map。然后创建一个Action执行代理类`ActionProxy`：
	
	public ActionProxy createActionProxy(String namespace, String actionName, String methodName, Map<String, Object> extraContext, boolean executeResult, boolean cleanupContext) {
	        
	        ActionInvocation inv = new DefaultActionInvocation(extraContext, true);
	        container.inject(inv);
	        return createActionProxy(inv, namespace, actionName, methodName, executeResult, cleanupContext);
	    }

		public ActionProxy createActionProxy(ActionInvocation inv, String namespace, String actionName, String methodName, boolean executeResult, boolean cleanupContext) {
		
		        DefaultActionProxy proxy = new DefaultActionProxy(inv, namespace, actionName, methodName, executeResult, cleanupContext);
		        container.inject(proxy);
		        proxy.prepare();
		        return proxy;
		    }

首先创建一个`ActionInvocation`（Action执行上下文，包含action信息，extraContext,interceptors等）。然后创建`DefaultActionProxy`，最后`prepare`：

	protected void prepare() {
        String profileKey = "create DefaultActionProxy: ";
        try {
            UtilTimerStack.push(profileKey);
            config = configuration.getRuntimeConfiguration().getActionConfig(namespace, actionName);

            if (config == null && unknownHandlerManager.hasUnknownHandlers()) {
                config = unknownHandlerManager.handleUnknownAction(namespace, actionName);
            }
            if (config == null) {
                throw new ConfigurationException(getErrorMessage());
            }

            resolveMethod();

            if (!config.isAllowedMethod(method)) {
                throw new ConfigurationException("Invalid method: " + method + " for action " + actionName);
            }

            invocation.init(this);

        } finally {
            UtilTimerStack.pop(profileKey);
        }
    }

首先从我们前面得到的`RuntimeConfiguration`获取`ActionConfig`信息。然年委托`invocation`初始化：
	
 	public void init(ActionProxy proxy) {
        this.proxy = proxy;
        Map<String, Object> contextMap = createContextMap();

        // Setting this so that other classes, like object factories, can use the ActionProxy and other
        // contextual information to operate
        ActionContext actionContext = ActionContext.getContext();

        if (actionContext != null) {
            actionContext.setActionInvocation(this);
        }

        createAction(contextMap);

        if (pushAction) {
            stack.push(action);
            contextMap.put("action", action);
        }

        invocationContext = new ActionContext(contextMap);
        invocationContext.setName(proxy.getActionName());

        // get a new List so we don't get problems with the iterator if someone changes the list
        List<InterceptorMapping> interceptorList = new ArrayList<InterceptorMapping>(proxy.getConfig().getInterceptors());
        interceptors = interceptorList.iterator();
    }

获取action类以及当前action的`InterceptorMapping`。
下面就是实施：调用过程，`com.opensymphony.xwork2.DefaultActionProxy#execute`->`com.opensymphony.xwork2.DefaultActionInvocation#invoke`：
	
	public String com.opensymphony.xwork2.DefaultActionInvocation() throws Exception {
        String profileKey = "invoke: ";
        try {
            UtilTimerStack.push(profileKey);

            if (executed) {
                throw new IllegalStateException("Action has already executed");
            }

            if (interceptors.hasNext()) {
                final InterceptorMapping interceptor = (InterceptorMapping) interceptors.next();
                String interceptorMsg = "interceptor: " + interceptor.getName();
                UtilTimerStack.push(interceptorMsg);
                try {
                                resultCode = interceptor.getInterceptor().intercept(DefaultActionInvocation.this);
                            }
                finally {
                    UtilTimerStack.pop(interceptorMsg);
                }
            } else {
                resultCode = invokeActionOnly();
            }

            // this is needed because the result will be executed, then control will return to the Interceptor, which will
            // return above and flow through again
            if (!executed) {
                if (preResultListeners != null) {
                    for (Object preResultListener : preResultListeners) {
                        PreResultListener listener = (PreResultListener) preResultListener;

                        String _profileKey = "preResultListener: ";
                        try {
                            UtilTimerStack.push(_profileKey);
                            listener.beforeResult(this, resultCode);
                        }
                        finally {
                            UtilTimerStack.pop(_profileKey);
                        }
                    }
                }

                // now execute the result, if we're supposed to
                if (proxy.getExecuteResult()) {
                    executeResult();
                }

                executed = true;
            }

            return resultCode;
        }
        finally {
            UtilTimerStack.pop(profileKey);
        }
    }

此处的逻辑很明显，递归调用`InterceptorMapping`，直到`Interceptor`有返回值或者interceptors执行完就作为一个`resultCode`返回。关于`Interceptor`执行原理见后面。
下面根据不同的resulttype获取不同的`com.opensymphony.xwork2.Result`来执行该返回码：

	private void executeResult() throws Exception {
        result = createResult();

        String timerKey = "executeResult: " + getResultCode();
        try {
            UtilTimerStack.push(timerKey);
            if (result != null) {
                result.execute(this);
            } else if (resultCode != null && !Action.NONE.equals(resultCode)) {
                throw new ConfigurationException("No result defined for action " + getAction().getClass().getName()
                        + " and result " + getResultCode(), proxy.getConfig());
            } else {
                if (LOG.isDebugEnabled()) {
                    LOG.debug("No result returned for action " + getAction().getClass().getName() + " at " + proxy.getConfig().getLocation());
                }
            }
        } finally {
            UtilTimerStack.pop(timerKey);
        }
    }

	public Result createResult() throws Exception {

        if (explicitResult != null) {
            Result ret = explicitResult;
            explicitResult = null;

            return ret;
        }
        ActionConfig config = proxy.getConfig();
        Map<String, ResultConfig> results = config.getResults();

        ResultConfig resultConfig = null;

        try {
            resultConfig = results.get(resultCode);
        } catch (NullPointerException e) {
            // swallow
        }
        
        if (resultConfig == null) {
            // If no result is found for the given resultCode, try to get a wildcard '*' match.
            resultConfig = results.get("*");
        }

        if (resultConfig != null) {
            try {
                return objectFactory.buildResult(resultConfig, invocationContext.getContextMap());
            } catch (Exception e) {
                LOG.error("There was an exception while instantiating the result of type " + resultConfig.getClassName(), e);
                throw new XWorkException(e, resultConfig);
            }
        } else if (resultCode != null && !Action.NONE.equals(resultCode) && unknownHandlerManager.hasUnknownHandlers()) {
            return unknownHandlerManager.handleUnknownResult(invocationContext, proxy.getActionName(), proxy.getConfig(), resultCode);
        }
        return null;
    }

根据前面加载的`ActionConfig`和`resultCode`获取特定的`ResultConfig`，默认有配置在`struts-default.xml`：

	<result-types>
        <result-type name="chain" class="com.opensymphony.xwork2.ActionChainResult"/>
        <result-type name="dispatcher" class="org.apache.struts2.dispatcher.ServletDispatcherResult" default="true"/>
        <result-type name="freemarker" class="org.apache.struts2.views.freemarker.FreemarkerResult"/>
        <result-type name="httpheader" class="org.apache.struts2.dispatcher.HttpHeaderResult"/>
        <result-type name="redirect" class="org.apache.struts2.dispatcher.ServletRedirectResult"/>
        <result-type name="redirectAction" class="org.apache.struts2.dispatcher.ServletActionRedirectResult"/>
        <result-type name="stream" class="org.apache.struts2.dispatcher.StreamResult"/>
        <result-type name="velocity" class="org.apache.struts2.dispatcher.VelocityResult"/>
        <result-type name="xslt" class="org.apache.struts2.views.xslt.XSLTResult"/>
        <result-type name="plainText" class="org.apache.struts2.dispatcher.PlainTextResult" />
    </result-types>

本项目使用默认的`dispatcher`。不生成`ServletDispatcherResult`实例处理。
### Interceptor工作原理
---
`struts-default.xml`提供一批默认`interceptor`：
	
	<default-interceptor-ref name="defaultStack"/>
	
	 <interceptor-stack name="defaultStack">
        <interceptor-ref name="exception"/>
        <interceptor-ref name="alias"/>
        <interceptor-ref name="servletConfig"/>
        <interceptor-ref name="i18n"/>
        <interceptor-ref name="prepare"/>
        <interceptor-ref name="chain"/>
        <interceptor-ref name="scopedModelDriven"/>
        <interceptor-ref name="modelDriven"/>
        <interceptor-ref name="fileUpload"/>
        <interceptor-ref name="checkbox"/>
        <interceptor-ref name="multiselect"/>
        <interceptor-ref name="staticParams"/>
        <interceptor-ref name="actionMappingParams"/>
        <interceptor-ref name="params">
          <param name="excludeParams">dojo\..*,^struts\..*</param>
        </interceptor-ref>
        <interceptor-ref name="conversionError"/>
        <interceptor-ref name="validation">
            <param name="excludeMethods">input,back,cancel,browse</param>
        </interceptor-ref>
        <interceptor-ref name="workflow">
            <param name="excludeMethods">input,back,cancel,browse</param>
        </interceptor-ref>
        <interceptor-ref name="debugging"/>
    </interceptor-stack>

我们以验证interceptor（包括validation和workflow）来说明：
`org.apache.struts2.interceptor.validation.AnnotationValidationInterceptor`，实现具体的action验证操作、

	 @Override
    protected String com.opensymphony.xwork2.validator.ValidationInterceptor#doIntercept(ActionInvocation invocation) throws Exception {
        doBeforeInvocation(invocation);
        
        return invocation.invoke();
    }

	protected void doBeforeInvocation(ActionInvocation invocation) throws Exception {
        Object action = invocation.getAction();
        ActionProxy proxy = invocation.getProxy();

        //the action name has to be from the url, otherwise validators that use aliases, like
        //MyActio-someaction-validator.xml will not be found, see WW-3194
        String context = proxy.getActionName();
        String method = proxy.getMethod();

        if (log.isDebugEnabled()) {
            log.debug("Validating "
                    + invocation.getProxy().getNamespace() + "/" + invocation.getProxy().getActionName() + " with method "+ method +".");
        }
        

        if (declarative) {
           if (validateAnnotatedMethodOnly) {
               actionValidatorManager.validate(action, context, method);
           } else {
               actionValidatorManager.validate(action, context);
           }
       }    
        
        if (action instanceof Validateable && programmatic) {
            // keep exception that might occured in validateXXX or validateDoXXX
            Exception exception = null; 
            
            Validateable validateable = (Validateable) action;
            if (LOG.isDebugEnabled()) {
                LOG.debug("Invoking validate() on action "+validateable);
            }
            
            try {
                PrefixMethodInvocationUtil.invokePrefixMethod(
                                invocation, 
                                new String[] { VALIDATE_PREFIX, ALT_VALIDATE_PREFIX });
            }
            catch(Exception e) {
                // If any exception occurred while doing reflection, we want 
                // validate() to be executed
                if (LOG.isWarnEnabled()) {
                    LOG.warn("an exception occured while executing the prefix method", e);
                }
                exception = e;
            }
            
            
            if (alwaysInvokeValidate) {
                validateable.validate();
            }
            
            if (exception != null) { 
                // rethrow if something is wrong while doing validateXXX / validateDoXXX 
                throw exception;
            }
        }
    }

 首先注解解析，我们不考虑。我们的`com.mkyong.user.action.WelcomeUserAction`继承的`com.opensymphony.xwork2.ActionSupport`。而`ActionSupport`继承于`Validateable`，执行`validateable.validate();`。执行实际的校验。然后继续递归响应。
而`DefaultWorkflowInterceptor`实现验证拦截处理：
	
	 protected String doIntercept(ActionInvocation invocation) throws Exception {
	        Object action = invocation.getAction();
	
	        if (action instanceof ValidationAware) {
	            ValidationAware validationAwareAction = (ValidationAware) action;
	
	            if (validationAwareAction.hasErrors()) {
	                if (LOG.isDebugEnabled()) {
	                    LOG.debug("Errors on action " + validationAwareAction + ", returning result name 'input'");
	                }
	
	                String resultName = inputResultName;
	
	                if (action instanceof ValidationWorkflowAware) {
	                    resultName = ((ValidationWorkflowAware) action).getInputResultName();
	                }
	
	                InputConfig annotation = action.getClass().getMethod(invocation.getProxy().getMethod(), EMPTY_CLASS_ARRAY).getAnnotation(InputConfig.class);
	                if (annotation != null) {
	                    if (!annotation.methodName().equals("")) {
	                        Method method = action.getClass().getMethod(annotation.methodName());
	                        resultName = (String) method.invoke(action);
	                    } else {
	                        resultName = annotation.resultName();
	                    }
	                }
	
	
	                return resultName;
	            }
	        }
	
	        return invocation.invoke();
	    }

如果验证有错误`validationAwareAction.hasErrors()`，直接返回`resultName`（默认`com.opensymphony.xwork2.Action#INPUT`）。如此便拦截了后面的拦截以及action的执行。