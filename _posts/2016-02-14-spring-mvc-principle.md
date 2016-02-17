---
title:  "Spring mvc工作原理"
date:   2016-02-14 17:20:00 +0800
tags: [Spring mvc]
---

### 需要解决的问题

Spring mvc按照MVC设计模式设计的框架，Model实体类传递给Controller，设置变量，返回给模板，渲染给客户端。  

目标是简化开发流程，不需要大量重复的代码，不需要像servlet每个请求都要配置一个servlet，并需要手工管理request，response。以及DI,aop的介入，可以定制大量操作，比如`HandlerMethodReturnValueHandler`，定制特定通用返回值。

### 处理流程

> 以[spring mvc json custom example](https://github.com/shaoyihe/spring_mvc)为例

1. DispatcherServlet处理
   
   `Spring MVC`的典型入口配置是在`web.xml`中这样定义:

 
	<servlet>
		<servlet-name>appServlet</servlet-name>
		<servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
		<init-param>
			<param-name>contextConfigLocation</param-name>
			<param-value>/WEB-INF/spring/appServlet/servlet-context.xml</param-value>
		</init-param>
	</servlet>
	<servlet-mapping>
		<servlet-name>appServlet</servlet-name>
		<url-pattern>/</url-pattern>
	</servlet-mapping>


​	`org.springframework.web.servlet.DispatcherServlet`继承于`javax.servlet.http.HttpServlet`，并没有什么特殊的，即该配置将所有请求委托与`DispatcherServlet`处理。  

	HandlerExecutionChain mappedHandler = getHandler(processedRequest);
	if (mappedHandler == null || mappedHandler.getHandler() == null) {
		noHandlerFound(processedRequest, response);
		return;
	}
	protected HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
		for (HandlerMapping hm : this.handlerMappings) {
			if (logger.isTraceEnabled()) {
				logger.trace(
						"Testing handler map [" + hm + "] in DispatcherServlet with name '" + getServletName() + "'");
			}
			HandlerExecutionChain handler = hm.getHandler(request);
			if (handler != null) {
				return handler;
			}
		}
		return null;
	}



当发起一个请求（`get`或者`post`）时，调用`DispatcherServlet`的`doGet`或`doPost`方法，跳过一些通用设置方法（公用属性设置等），达到关键方法`doDispatch`。

这一步通过预置的`HandlerMapping`找到`HandlerExecutionChain`。  

概念解释：  

`Handler`：`Controller`，具体的是`com.he.app.controller.JsonController`和`com.he.app.controller.HomeController`。  

`HandlerInterceptor`：`Handler`的前后拦截器。  

`HandlerExecutionChain`：`Handler`和它的`HandlerInterceptor`的合集。  

`HandlerMapping`：`HandlerExecutionChain`的工厂方法。  

那么，`List<HandlerMapping>`又是如何来的？如何实现的？看2。  

接着获取`HandlerAdapter`：

	HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());
	protected HandlerAdapter getHandlerAdapter(Object handler) throws ServletException {
		for (HandlerAdapter ha : this.handlerAdapters) {
			if (ha.supports(handler)) {
				return ha;
			}
		}
	}


那么`handlerAdapters`是如何来的，阅读3。

下面就是`HandlerAdapter`执行逻辑并得到`ModelAndView`对象。中间穿插`HandlerExecutionChain`包含的拦截器。


	if (!mappedHandler.applyPreHandle(processedRequest, response)) {
		return;
	}

	// Actually invoke the handler.
	mv = ha.handle(processedRequest, response, mappedHandler.getHandler());

	if (asyncManager.isConcurrentHandlingStarted()) {
		return;
	}

	applyDefaultViewName(processedRequest, mv);
	mappedHandler.applyPostHandle(processedRequest, response, mv);


下面就是页面渲染`processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);`：	

 
	private void processDispatchResult(HttpServletRequest request, HttpServletResponse response,
		HandlerExecutionChain mappedHandler, ModelAndView mv, Exception exception) throws Exception {

		boolean errorView = false;
	    // Did the handler return a view to render?
			if (mv != null && !mv.wasCleared()) {
				render(mv, request, response);
				if (errorView) {
					WebUtils.clearErrorRequestAttributes(request);
				}
			}
	     }

	protected void render(ModelAndView mv, HttpServletRequest request, HttpServletResponse response) throws Exception {
		// Determine locale for request and apply it to the response.
		Locale locale = this.localeResolver.resolveLocale(request);
		response.setLocale(locale);
		View view;
		if (mv.isReference()) {
			// We need to resolve the view name.
			view = resolveViewName(mv.getViewName(), mv.getModelInternal(), locale, request);
			if (view == null) {
				throw new ServletException("Could not resolve view with name '" + mv.getViewName() +
						"' in servlet with name '" + getServletName() + "'");
			}
		}
		else {
			// No need to lookup: the ModelAndView object contains the actual View object.
			view = mv.getView();
			if (view == null) {
				throw new ServletException("ModelAndView [" + mv + "] neither contains a view name nor a " +
						"View object in servlet with name '" + getServletName() + "'");
			}
		}
        // Delegate to the View object for rendering.
		if (logger.isDebugEnabled()) {
			logger.debug("Rendering view [" + view + "] in DispatcherServlet with name '" + getServletName() + "'");
		}
		try {
			view.render(mv.getModelInternal(), request, response);
		}
		catch (Exception ex) {
			if (logger.isDebugEnabled()) {
				logger.debug("Error rendering view [" + view + "] in DispatcherServlet with name '" +
						getServletName() + "'", ex);
			}
			throw ex;
		}
	}
	protected View resolveViewName(String viewName, Map<String, Object> model, Locale locale,
			HttpServletRequest request) throws Exception {	
            for (ViewResolver viewResolver : this.viewResolvers) {
			View view = viewResolver.resolveViewName(viewName, locale);
			if (view != null) {
				return view;
			}
		}
		return null;
	}

基本逻辑是遍历`viewResolvers`，找到可以解决当前view的view，然后render请求。`viewResolvers`工作原理见7.

​	

2 . `List<HandlerMapping>`的由来
   
   ​	
   
   - 接口探测，在`org.springframework.web.servlet.DispatcherServlet#initHandlerMappings`进行初始化：

	private void initHandlerMappings(ApplicationContext context) {
		this.handlerMappings = null;
		if (this.detectAllHandlerMappings) {
			// Find all HandlerMappings in the ApplicationContext, including ancestor contexts.
			Map<String, HandlerMapping> matchingBeans =
					BeanFactoryUtils.beansOfTypeIncludingAncestors(context, HandlerMapping.class, true, false);
			if (!matchingBeans.isEmpty()) {
				this.handlerMappings = new ArrayList<HandlerMapping>(matchingBeans.values());
				// We keep HandlerMappings in sorted order.
				AnnotationAwareOrderComparator.sort(this.handlerMappings);
			}
		}


调试发现在这里获取到全部的`handlerMappings`，即通过找到所有在`BeanFactory`注册的`org.springframework.web.servlet.HandlerMapping`的子类加载到`handlerMappings`。这种设计思想广泛用于Spring设计中，即动态注入思想的扩展。获益：这里同时也是一个定制点，我们可以实现自己的`HandlerMapping`，只要是一个继承`HandlerMapping`接口的`Bean`即可。那么这些实现类又是什么时候注入`BeanFactory`的呢？

- 先看前面的`web.xml`的配置`contextConfigLocation`：`/WEB-INF/spring/appServlet/servlet-context.xml`，此一步配置整个框架的配置，而该文件的调用过程如图所示：  
  
  ![getContextConfigLocation](/assets/images//getContextConfigLocation.png)  
  
  最终调用正常的`fresh`加载xml。
  
- 再看`appServlet/servlet-context.xml`：  
    
 
  	<mvc:annotation-driven>
  	</mvc:annotation-driven>
   
  
  声明mvc命名空间加载，具体如何加载不涉及，基本逻辑是根据不同的命名空间(xmlns)找到`org.springframework.beans.factory.xml.NamespaceHandlerSupport`不同的子类处理，然后注册不同的`element`。例如：`AopNamespaceHandler`来处理`<aop:config>`系列。而这里同理使用`MvcNamespaceHandler`：

	
		public void init() {
			registerBeanDefinitionParser("annotation-driven", new AnnotationDrivenBeanDefinitionParser());
			...
 

再`org.springframework.web.servlet.config.AnnotationDrivenBeanDefinitionParser#parse`中进行基础配置，`HandlerMethod`相关的：

 
	public BeanDefinition parse(Element element, ParserContext parserContext) {
		RootBeanDefinition handlerMappingDef = new RootBeanDefinition(RequestMappingHandlerMapping.class);
		handlerMappingDef.setSource(source);
		handlerMappingDef.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
		handlerMappingDef.getPropertyValues().add("order", 0);
		handlerMappingDef.getPropertyValues().add("contentNegotiationManager", contentNegotiationManager);
		...
		parserContext.registerComponent(new BeanComponentDefinition(handlerMappingDef, methodMappingName));
	
		...
		MvcNamespaceUtils.registerDefaultComponents(parserContext, source);
		...
	}	
		
	private static void org.springframework.web.servlet.config.MvcNamespaceUtils#registerBeanNameUrlHandlerMapping(ParserContext parserContext, Object source) {
		if (!parserContext.getRegistry().containsBeanDefinition(BEAN_NAME_URL_HANDLER_MAPPING_BEAN_NAME)){
			RootBeanDefinition beanNameMappingDef = new RootBeanDefinition(BeanNameUrlHandlerMapping.class);
			beanNameMappingDef.setSource(source);
			beanNameMappingDef.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
			beanNameMappingDef.getPropertyValues().add("order", 2);	// consistent with WebMvcConfigurationSupport
			RuntimeBeanReference corsConfigurationsRef = MvcNamespaceUtils.registerCorsConfigurations(null, parserContext, source);
			beanNameMappingDef.getPropertyValues().add("corsConfigurations", corsConfigurationsRef);
			parserContext.getRegistry().registerBeanDefinition(BEAN_NAME_URL_HANDLER_MAPPING_BEAN_NAME, beanNameMappingDef);
			parserContext.registerComponent(new BeanComponentDefinition(beanNameMappingDef, BEAN_NAME_URL_HANDLER_MAPPING_BEAN_NAME));
	}


这里注册2个`org.springframework.web.servlet.HandlerMapping`的`Bean`：  

`org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping`：注册基于注解(`RequestMapping`和`Controller`)的Controller  

`org.springframework.web.servlet.handler.BeanNameUrlHandlerMapping`：注册基于`Bean`名（以/开头）的的的Controller  

下面以常用的`RequestMappingHandlerMapping`说明  

- `RequestMappingHandlerMapping`初始化逻辑  
  
  作为一个`Bean`的`RequestMappingHandlerMapping`的初始化在`org.springframework.web.servlet.handler.AbstractHandlerMethodMapping#afterPropertiesSet`完成。



		public void afterPropertiesSet() {
			initHandlerMethods();
		}
		
		protected void initHandlerMethods() {
			if (logger.isDebugEnabled()) {
				logger.debug("Looking for request mappings in application context: " + getApplicationContext());
			}
			String[] beanNames = (this.detectHandlerMethodsInAncestorContexts ?
					BeanFactoryUtils.beanNamesForTypeIncludingAncestors(getApplicationContext(), Object.class) :
					getApplicationContext().getBeanNamesForType(Object.class));

			for (String name : beanNames) {
				if (!name.startsWith(SCOPED_TARGET_NAME_PREFIX) && isHandler(getApplicationContext().getType(name))) {
					detectHandlerMethods(name);
				}
			}
			handlerMethodsInitialized(getHandlerMethods());
		}

		protected boolean isHandler(Class<?> beanType) {
			return ((AnnotationUtils.findAnnotation(beanType, Controller.class) != null) ||
					(AnnotationUtils.findAnnotation(beanType, RequestMapping.class) != null));
		}
	

基本逻辑是：找到所有的`Bean对象`，如果`beanName`不以`SCOPED_TARGET_NAME_PREFIX`(scopedTarget)开始且`BeanType`有注解`Controller`或`RequestMapping`，则注册该`Bean`的`Handler Method`（`handlerMethodsInitialized`）：
	
	
		protected void detectHandlerMethods(final Object handler) {
			Class<?> handlerType =
					(handler instanceof String ? getApplicationContext().getType((String) handler) : handler.getClass());
	
			// Avoid repeated calls to getMappingForMethod which would rebuild RequestMappingInfo instances
			final Map<Method, T> mappings = new IdentityHashMap<Method, T>();
			final Class<?> userType = ClassUtils.getUserClass(handlerType);
	
			Set<Method> methods = HandlerMethodSelector.selectMethods(userType, new MethodFilter() {
				@Override
				public boolean matches(Method method) {
					T mapping = getMappingForMethod(method, userType);
					if (mapping != null) {
						mappings.put(method, mapping);
						return true;
					}
					else {
						return false;
					}
				}
			});
	
			for (Method method : methods) {
				registerHandlerMethod(handler, method, mappings.get(method));
			}
		}
	

		protected RequestMappingInfo getMappingForMethod(Method method, Class<?> handlerType) {
			RequestMappingInfo info = createRequestMappingInfo(method);
			if (info != null) {
				RequestMappingInfo typeInfo = createRequestMappingInfo(handlerType);
				if (typeInfo != null) {
					info = typeInfo.combine(info);
				}
			}
			return info;
		}
	
		private RequestMappingInfo createRequestMappingInfo(AnnotatedElement element) {
			RequestMapping requestMapping = AnnotatedElementUtils.findMergedAnnotation(element, RequestMapping.class);
			RequestCondition<?> condition = (element instanceof Class<?> ?
					getCustomTypeCondition((Class<?>) element) : getCustomMethodCondition((Method) element));
			return (requestMapping != null ? createRequestMappingInfo(requestMapping, condition) : null);
		}
	

遍历该`Bean`的所有方法，如果有注解`RequestMapping`，则为其封装为`RequestMappingInfo`对象，然后注册该方法到`mappingRegistry`。

- 获取`HandlerExecutionChain`：

	
		public final HandlerExecutionChain org.springframework.web.servlet.handler.AbstractHandlerMapping#getHandler(HttpServletRequest request) throws Exception {
			Object handler = getHandlerInternal(request);
			if (handler == null) {
				handler = getDefaultHandler();
			}
			if (handler == null) {
				return null;
			}
			// Bean name or resolved handler?
			if (handler instanceof String) {
				String handlerName = (String) handler;
				handler = getApplicationContext().getBean(handlerName);
			}
	
			HandlerExecutionChain executionChain = getHandlerExecutionChain(handler, request);
			if (CorsUtils.isCorsRequest(request)) {
				CorsConfiguration globalConfig = this.corsConfigSource.getCorsConfiguration(request);
				CorsConfiguration handlerConfig = getCorsConfiguration(handler, request);
				CorsConfiguration config = (globalConfig != null ? globalConfig.combine(handlerConfig) : handlerConfig);
				executionChain = getCorsHandlerExecutionChain(request, executionChain, config);
			}
			return executionChain;
		}
	
		protected HandlerMethod org.springframework.web.servlet.handler.AbstractHandlerMethodMapping#getHandlerInternal(HttpServletRequest request) throws Exception {
				String lookupPath = getUrlPathHelper().getLookupPathForRequest(request);
				HandlerMethod handlerMethod = lookupHandlerMethod(lookupPath, request);
				return (handlerMethod != null ? handlerMethod.createWithResolvedBean() : null)
			}

首先通过前面注册的`RequestMappingHandlerMapping`，调用它的`getHandler`，然后调用`getHandlerInternal`，从`mappingRegistry`中获取`HandlerMethod`（包括`Bean`及目标`Method`）.然后组合`HandlerMethod`和目标`HandlerInterceptor`为`HandlerExecutionChain`：

 
	protected HandlerExecutionChain getHandlerExecutionChain(Object handler, HttpServletRequest request) {
		HandlerExecutionChain chain = (handler instanceof HandlerExecutionChain ?
				(HandlerExecutionChain) handler : new HandlerExecutionChain(handler));

		String lookupPath = this.urlPathHelper.getLookupPathForRequest(request);
		for (HandlerInterceptor interceptor : this.adaptedInterceptors) {
			if (interceptor instanceof MappedInterceptor) {
				MappedInterceptor mappedInterceptor = (MappedInterceptor) interceptor;
				if (mappedInterceptor.matches(lookupPath, this.pathMatcher)) {
					chain.addInterceptor(mappedInterceptor.getInterceptor());
				}
			}
			else {
				chain.addInterceptor(interceptor);
			}
		}
		return chain;
	}


3 . `handlerAdapters`的由来  
   - 同`handlerMappings`的初始化逻辑：


    private void initHandlerAdapters(ApplicationContext context) {
		this.handlerAdapters = null;

		if (this.detectAllHandlerAdapters) {
			// Find all HandlerAdapters in the ApplicationContext, including ancestor contexts.
			Map<String, HandlerAdapter> matchingBeans =
					BeanFactoryUtils.beansOfTypeIncludingAncestors(context, HandlerAdapter.class, true, false);
			if (!matchingBeans.isEmpty()) {
				this.handlerAdapters = new ArrayList<HandlerAdapter>(matchingBeans.values());
				// We keep HandlerAdapters in sorted order.
				AnnotationAwareOrderComparator.sort(this.handlerAdapters);
			}
	}	



​	加载`HandlerAdapter`的所有子类的`Bean`到`handlerMappings`。再看这些在哪里注入的：

- 与`HandleMapper`的加载过程类似，均在`org.springframework.web.servlet.config.AnnotationDrivenBeanDefinitionParser#parse`中：

​	

  	RootBeanDefinition handlerAdapterDef = new RootBeanDefinition(RequestMappingHandlerAdapter.class);
	handlerAdapterDef.setSource(source);
	handlerAdapterDef.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
	handlerAdapterDef.getPropertyValues().add("contentNegotiationManager", contentNegotiationManager);
	handlerAdapterDef.getPropertyValues().add("webBindingInitializer", bindingDef);
	handlerAdapterDef.getPropertyValues().add("messageConverters", messageConverters);
	addRequestBodyAdvice(handlerAdapterDef);
	addResponseBodyAdvice(handlerAdapterDef);

	parserContext.registerComponent(new BeanComponentDefinition(handlerAdapterDef, handlerAdapterName));


下面以`RequestMappingHandlerAdapter`为例，说明`HandlerAdapter`处理逻辑：


	public final boolean supports(Object handler) {
		return (handler instanceof HandlerMethod && supportsInternal((HandlerMethod) handler));
	}

支持类型为`HandlerMethod`的`Handler`，这正是我们前面的`RequestMappingHandlerMapping`生成的`Handler`类型。其实这是一对组合使用的逻辑。继续看handle方法：`org.springframework.web.servlet.mvc.method.AbstractHandlerMethodAdapter#handleInternal`委托于`org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter#handleInternal`，再委托于`invokeHandlerMethod`方法：


	protected ModelAndView invokeHandlerMethod(HttpServletRequest request,
		HttpServletResponse response, HandlerMethod handlerMethod) throws Exception {
        ServletWebRequest webRequest = new ServletWebRequest(request, response);
        ServletInvocableHandlerMethod invocableMethod = createInvocableHandlerMethod(handlerMethod);
		invocableMethod.setHandlerMethodArgumentResolvers(this.argumentResolvers);
		invocableMethod.setHandlerMethodReturnValueHandlers(this.returnValueHandlers);
		invocableMethod.setDataBinderFactory(binderFactory);
		invocableMethod.setParameterNameDiscoverer(this.parameterNameDiscoverer)
	    ModelAndViewContainer mavContainer = new ModelAndViewContainer();
		mavContainer.addAllAttributes(RequestContextUtils.getInputFlashMap(request));
		modelFactory.initModel(webRequest, mavContainer, invocableMethod);
		mavContainer.setIgnoreDefaultModelOnRedirect(this.ignoreDefaultModelOnRedirect);
	    invocableMethod.invokeAndHandle(webRequest, mavContainer);
	    return getModelAndView(mavContainer, modelFactory, webRequest);
	}



封装`handlerMethod`为`ServletInvocableHandlerMethod`。复制参数（argumentResolvers和returnValueHandlers）。初始化`ModelAndViewContainer`（即包含Model和View的集合），最终调用`invokeAndHandle`方法。

	public void invokeAndHandle(ServletWebRequest webRequest,
			ModelAndViewContainer mavContainer, Object... providedArgs) throws Exception { 
		Object returnValue = invokeForRequest(webRequest, mavContainer, providedArgs);
		setResponseStatus(webRequest);
		if (returnValue == null) {
			if (isRequestNotModified(webRequest) || hasResponseStatus() || mavContainer.isRequestHandled()) {
				mavContainer.setRequestHandled(true);
				return;
			}
		}
		else if (StringUtils.hasText(this.responseReason)) {
			mavContainer.setRequestHandled(true);
			return;
		}
		mavContainer.setRequestHandled(false);
		try {
			this.returnValueHandlers.handleReturnValue(
					returnValue, getReturnValueType(returnValue), mavContainer, webRequest);
		}
		catch (Exception ex) {
			if (logger.isTraceEnabled()) {
				logger.trace(getReturnValueHandlingErrorMessage("Error handling return value", returnValue), ex);
			}
			throw ex;
		}
	}
	public Object invokeForRequest(NativeWebRequest request, ModelAndViewContainer mavContainer,
			Object... providedArgs) throws Exception {
		Object[] args = getMethodArgumentValues(request, mavContainer, providedArgs);
		Object returnValue = doInvoke(args);
		return returnValue;
	}
	
	private Object[] getMethodArgumentValues(NativeWebRequest request, ModelAndViewContainer mavContainer,
		Object... providedArgs) throws Exception {

		MethodParameter[] parameters = getMethodParameters();
		Object[] args = new Object[parameters.length];
		for (int i = 0; i < parameters.length; i++) {
			MethodParameter parameter = parameters[i];
			parameter.initParameterNameDiscovery(this.parameterNameDiscoverer);
			GenericTypeResolver.resolveParameterType(parameter, getBean().getClass());
			args[i] = resolveProvidedArgument(parameter, providedArgs);
			if (args[i] != null) {
				continue;
			}
			if (this.argumentResolvers.supportsParameter(parameter)) {
				try {
					args[i] = this.argumentResolvers.resolveArgument(
							parameter, mavContainer, request, this.dataBinderFactory);
					continue;
				}
				catch (Exception ex) {
					if (logger.isDebugEnabled()) {
						logger.debug(getArgumentResolutionErrorMessage("Error resolving argument", i), ex);
					}
					throw ex;
				}
			}
			if (args[i] == null) {
				String msg = getArgumentResolutionErrorMessage("No suitable resolver for argument", i);
				throw new IllegalStateException(msg);
			}
		}
		return args;
	}


首先执行`controller`，其中先进行参数值生成（比如`@PathVariable`，`@RequestParam`），再反射执行`doInvoke`实施`controller`,至此我们的业务代码被执行。

关于`argumentResolvers`工作原理参见4.

获取到返回值后，调用`returnValueHandlers`处理返回值：


	public void handleReturnValue(Object returnValue, MethodParameter returnType,
			ModelAndViewContainer mavContainer, NativeWebRequest webRequest) throws Exception {
            HandlerMethodReturnValueHandler handler = selectHandler(returnValue, returnType);
		Assert.notNull(handler, "Unknown return value type [" + returnType.getParameterType().getName() + "]");
		handler.handleReturnValue(returnValue, returnType, mavContainer, webRequest);
	}

	private HandlerMethodReturnValueHandler selectHandler(Object value, MethodParameter returnType) {
		boolean isAsyncValue = isAsyncReturnValue(value, returnType);
		for (HandlerMethodReturnValueHandler handler : this.returnValueHandlers) {
			if (isAsyncValue && !(handler instanceof AsyncHandlerMethodReturnValueHandler)) {
				continue;
			}
			if (handler.supportsReturnType(returnType)) {
				return handler;
			}
		}
		return null;
	}


4 . `argumentResolvers`工作原理
   - 加载：在`org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter#afterPropertiesSet`：


	public void afterPropertiesSet() {
			// Do this first, it may add ResponseBody advice beans
		initControllerAdviceCache();
	
		if (this.argumentResolvers == null) {
			List<HandlerMethodArgumentResolver> resolvers = getDefaultArgumentResolvers();
			this.argumentResolvers = new HandlerMethodArgumentResolverComposite().addResolvers(resolvers);
		}
		
		if (this.returnValueHandlers == null) {
			List<HandlerMethodReturnValueHandler> handlers = getDefaultReturnValueHandlers();
			this.returnValueHandlers = new HandlerMethodReturnValueHandlerComposite().addHandlers(handlers);
		}
	}
	private List<HandlerMethodArgumentResolver> getDefaultArgumentResolvers() {
		List<HandlerMethodArgumentResolver> resolvers = new ArrayList<HandlerMethodArgumentResolver>();

		// Annotation-based argument resolution
		resolvers.add(new RequestParamMethodArgumentResolver(getBeanFactory(), false));
		resolvers.add(new RequestParamMapMethodArgumentResolver());
		resolvers.add(new PathVariableMethodArgumentResolver());
		resolvers.add(new PathVariableMapMethodArgumentResolver());
		resolvers.add(new MatrixVariableMethodArgumentResolver());
		resolvers.add(new MatrixVariableMapMethodArgumentResolver());
		...


​	下面选择`org.springframework.web.method.annotation.RequestParamMethodArgumentResolver`说明解决过程：

- RequestParamMethodArgumentResolver故名思议是解决`RequestParam`注解的参数。


- `supportsParameter`方法判断参数是否包含注解`RequestParam`：

​	
	public boolean supportsParameter(MethodParameter parameter) {
			Class<?> paramType = parameter.getParameterType();
			if (parameter.hasParameterAnnotation(RequestParam.class)) {
				if (Map.class.isAssignableFrom(paramType)) {
					String paramName = parameter.getParameterAnnotation(RequestParam.class).name();
					return StringUtils.hasText(paramName);
				}
				else {
					return true;
				}
			}
			...
		}		

`resolveName`方法获取参数实际值：

		protected Object resolveName(String name, MethodParameter parameter, NativeWebRequest webRequest) throws Exception {
			HttpServletRequest servletRequest = webRequest.getNativeRequest(HttpServletRequest.class);
			MultipartHttpServletRequest multipartRequest =
					WebUtils.getNativeRequest(servletRequest, MultipartHttpServletRequest.class);
			Object arg;
			...
			else {
				if (arg == null) {
					String[] paramValues = webRequest.getParameterValues(name);
					if (paramValues != null) {
						arg = (paramValues.length == 1 ? paramValues[0] : paramValues);
					}
				}
			}
	
			return arg;
		}

 基本就是从`HttpServletRequest`中获取值。
5 . `returnValueHandlers`工作原理
   - 加载：   在`org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter#afterPropertiesSet`：


		private List<HandlerMethodReturnValueHandler> getDefaultReturnValueHandlers() {
			List<HandlerMethodReturnValueHandler> handlers = new ArrayList<HandlerMethodReturnValueHandler>();
	
			// Single-purpose return value types
			handlers.add(new ModelAndViewMethodReturnValueHandler());
			handlers.add(new ModelMethodProcessor());
			handlers.add(new ViewMethodReturnValueHandler());
			handlers.add(new ResponseBodyEmitterReturnValueHandler(getMessageConverters()));
			handlers.add(new StreamingResponseBodyReturnValueHandler());
			handlers.add(new HttpEntityMethodProcessor(getMessageConverters(),
					this.contentNegotiationManager, this.requestResponseBodyAdvice));
			handlers.add(new HttpHeadersReturnValueHandler());
			handlers.add(new CallableMethodReturnValueHandler());
			handlers.add(new DeferredResultMethodReturnValueHandler());
			handlers.add(new AsyncTaskMethodReturnValueHandler(this.beanFactory));
			handlers.add(new ListenableFutureReturnValueHandler());
			if (completionStagePresent) {
				handlers.add(new CompletionStageReturnValueHandler());
			}
	
			// Annotation-based return value types
			handlers.add(new ModelAttributeMethodProcessor(false));
			handlers.add(new RequestResponseBodyMethodProcessor(getMessageConverters(),
					this.contentNegotiationManager, this.requestResponseBodyAdvice));
	
			// Multi-purpose return value types
			handlers.add(new ViewNameMethodReturnValueHandler());
			handlers.add(new MapMethodProcessor());


预置大量的`ReturnValueHandler`，下面以`ViewNameMethodReturnValueHandler`和`RequestResponseBodyMethodProcessor`为例。

- `ViewNameMethodReturnValueHandler`：
  
  支持返回类型为`Void`或者`CharSequence`,然后设置`ModelAndViewContainer`的·viewname，见`org.springframework.web.servlet.mvc.method.annotation.ViewNameMethodReturnValueHandler`：

		public boolean supportsReturnType(MethodParameter returnType) {
			Class<?> paramType = returnType.getParameterType();
			return (void.class == paramType || CharSequence.class.isAssignableFrom(paramType));
		}
	
		@Override
		public void handleReturnValue(Object returnValue, MethodParameter returnType,
				ModelAndViewContainer mavContainer, NativeWebRequest webRequest) throws Exception {
	
			if (returnValue instanceof CharSequence) {
				String viewName = returnValue.toString();
				mavContainer.setViewName(viewName);
				if (isRedirectViewName(viewName)) {
					mavContainer.setRedirectModelScenario(true);
				}
			}
			else if (returnValue != null){
				// should not happen
				throw new UnsupportedOperationException("Unexpected return type: " +
						returnType.getParameterType().getName() + " in method: " + returnType.getMethod());
			}
		}

		protected boolean isRedirectViewName(String viewName) {
			return (PatternMatchUtils.simpleMatch(this.redirectPatterns, viewName) || viewName.startsWith("redirect:"));
		}


中间穿插了如果`returnValue`以"redirect:"开头，则视为页面跳转`setRedirectModelScenario`。完了，后面的页面渲染是给`org.springframework.web.servlet.ViewResolver`解决的。

- `org.springframework.web.servlet.mvc.method.annotation.RequestResponseBodyMethodProcessor`  
  
  此返回值处理器是响应没有页面需要渲染，直接返回给客户端，比如json，xml,字符串等格式。 支持返回包含注解`ResponseBody`，然后委托各种不同类型`HttpMessageConverter`处理：


		public boolean supportsReturnType(MethodParameter returnType) {
			return (AnnotationUtils.findAnnotation(returnType.getContainingClass(), ResponseBody.class) != null ||
					returnType.getMethodAnnotation(ResponseBody.class) != null);
		}

		public void handleReturnValue(Object returnValue, MethodParameter returnType,
				ModelAndViewContainer mavContainer, NativeWebRequest webRequest)
				throws IOException, HttpMediaTypeNotAcceptableException, HttpMessageNotWritableException {

			mavContainer.setRequestHandled(true);

			// Try even with null return value. ResponseBodyAdvice could get involved.
			writeWithMessageConverters(returnValue, returnType, webRequest);
		}

		protected <T> void writeWithMessageConverters(T returnValue, MethodParameter returnType, NativeWebRequest webRequest)
			throws IOException, HttpMediaTypeNotAcceptableException, HttpMessageNotWritableException {

			ServletServerHttpRequest inputMessage = createInputMessage(webRequest);
			ServletServerHttpResponse outputMessage = createOutputMessage(webRequest);
			writeWithMessageConverters(returnValue, returnType, inputMessage, outputMessage);
		}
		
		protected <T> void writeWithMessageConverters(T returnValue, MethodParameter returnType,
					ServletServerHttpRequest inputMessage, ServletServerHttpResponse outputMessage)
					throws IOException, HttpMediaTypeNotAcceptableException, HttpMessageNotWritableException {
		
				...
				if (selectedMediaType != null) {
					selectedMediaType = selectedMediaType.removeQualityValue();
					for (HttpMessageConverter<?> messageConverter : this.messageConverters) {
						if (messageConverter instanceof GenericHttpMessageConverter) {
							if (((GenericHttpMessageConverter<T>) messageConverter).canWrite(returnValueType,
									returnValueClass, selectedMediaType)) {
								returnValue = (T) getAdvice().beforeBodyWrite(returnValue, returnType, selectedMediaType,
										(Class<? extends HttpMessageConverter<?>>) messageConverter.getClass(),
										inputMessage, outputMessage);
								if (returnValue != null) {
									((GenericHttpMessageConverter<T>) messageConverter).write(returnValue,
											returnValueType, selectedMediaType, outputMessage);
									if (logger.isDebugEnabled()) {
										logger.debug("Written [" + returnValue + "] as \"" +
												selectedMediaType + "\" using [" + messageConverter + "]");
									}
								}
								return;
							}
						}
						...
					}
				}
		
				if (returnValue != null) {
					throw new HttpMediaTypeNotAcceptableException(this.allSupportedMediaTypes);
				}
			}


注意这里已经把结果返回给客户端，后面不再处理。固需要执行方法：`mavContainer.setRequestHandled(true);`。标记请求已经被处理。

``` 
`messageConverters`工作原理见6。
```



6 . `messageConverters`工作原理  
   
   ``` 
   - 加载
   在前面说到的`org.springframework.web.servlet.config.AnnotationDrivenBeanDefinitionParser#parse`中`RequestMappingHandlerAdapter`的初始化有`handlerAdapterDef.getPropertyValues().add("messageConverters", messageConverters);`的注入。
 
		private ManagedList<?> getMessageConverters(Element element, Object source, ParserContext parserContext) {
			Element convertersElement = DomUtils.getChildElementByTagName(element, "message-converters");
			ManagedList<? super Object> messageConverters = new ManagedList<Object>();
			if (convertersElement != null) {
				messageConverters.setSource(source);
				for (Element beanElement : DomUtils.getChildElementsByTagName(convertersElement, "bean", "ref")) {
					Object object = parserContext.getDelegate().parsePropertySubElement(beanElement, null);
					messageConverters.add(object);
				}
			}
		if (convertersElement == null || Boolean.valueOf(convertersElement.getAttribute("register-defaults"))) {
			messageConverters.setSource(source);
			messageConverters.add(createConverterDefinition(ByteArrayHttpMessageConverter.class, source));
			RootBeanDefinition stringConverterDef = createConverterDefinition(StringHttpMessageConverter.class, source);
			stringConverterDef.getPropertyValues().add("writeAcceptCharset", false);
			messageConverters.add(createConverterDefinition(ResourceHttpMessageConverter.class, source));
			messageConverters.add(createConverterDefinition(SourceHttpMessageConverter.class, source));
			messageConverters.add(createConverterDefinition(AllEncompassingFormHttpMessageConverter.class, source));
			if (romePresent) {
				messageConverters.add(createConverterDefinition(AtomFeedHttpMessageConverter.class, source));
				messageConverters.add(createConverterDefinition(RssChannelHttpMessageConverter.class, source));
			}
			if (jackson2XmlPresent) {
				RootBeanDefinition jacksonConverterDef = createConverterDefinition(MappingJackson2XmlHttpMessageConverter.class, source);
				GenericBeanDefinition jacksonFactoryDef = createObjectMapperFactoryDefinition(source);
				jacksonFactoryDef.getPropertyValues().add("createXmlMapper", true);
				jacksonConverterDef.getConstructorArgumentValues().addIndexedArgumentValue(0, jacksonFactoryDef);
				messageConverters.add(jacksonConverterDef);
			}
			else if (jaxb2Present) {
				messageConverters.add(createConverterDefinition(Jaxb2RootElementHttpMessageConverter.class, source));
			}
			if (jackson2Present) {
				RootBeanDefinition jacksonConverterDef = createConverterDefinition(MappingJackson2HttpMessageConverter.class, source);
				GenericBeanDefinition jacksonFactoryDef = createObjectMapperFactoryDefinition(source);
				jacksonConverterDef.getConstructorArgumentValues().addIndexedArgumentValue(0, jacksonFactoryDef);
				messageConverters.add(jacksonConverterDef);
			}
			else if (gsonPresent) {
				messageConverters.add(createConverterDefinition(GsonHttpMessageConverter.class, source));
			}
		}
		return messageConverters;
		}

		private static final boolean jackson2Present =
			ClassUtils.isPresent("com.fasterxml.jackson.databind.ObjectMapper", AnnotationDrivenBeanDefinitionParser.class.getClassLoader()) &&
					ClassUtils.isPresent("com.fasterxml.jackson.core.JsonGenerator", AnnotationDrivenBeanDefinitionParser.class.getClassLoader());


这里如果没有在xml明确定义`message-converters`元素，那么会提供默认值，本项目是用的`jackson`，Spring是判断classpath是否存在类`com.fasterxml.jackson.databind.ObjectMapper`来判断是否启用`jackson`消息转换，我们有配置，故提供`MappingJackson2HttpMessageConverter`作为`Json`返回值的处理器。
而同时在`RequestMappingHandlerAdapter`构造函数中提供4个预置的消息处理器。

	public RequestMappingHandlerAdapter() {
		StringHttpMessageConverter stringHttpMessageConverter = new StringHttpMessageConverter();
		stringHttpMessageConverter.setWriteAcceptCharset(false);  // see SPR-7316
		this.messageConverters = new ArrayList<HttpMessageConverter<?>>(4);
		this.messageConverters.add(new ByteArrayHttpMessageConverter());
		this.messageConverters.add(stringHttpMessageConverter);
		this.messageConverters.add(new SourceHttpMessageConverter<Source>());
		this.messageConverters.add(new AllEncompassingFormHttpMessageConverter());
	}

预置的处理器都很简单，比如：  
`org.springframework.http.converter.StringHttpMessageConverter`：支持返回值为String,直接写该字符串到流中  
`org.springframework.http.converter.ByteArrayHttpMessageConverter`：支持返回值byte[]，写到response中
其他`MappingJackson2HttpMessageConverter`基本类似。不再说明。
```

7 . `viewResolvers`工作原理
   
   
   - 加载
   逻辑等同于`HandleMapper`和`HandleAdapter`,加载所有`org.springframework.web.servlet.ViewResolver`的子类Bean到`viewResolvers`，如果不存在，则加载默认配置。
  

	private void initViewResolvers(ApplicationContext context) {
		this.viewResolvers = null;
		if (this.detectAllViewResolvers) {
			// Find all ViewResolvers in the ApplicationContext, including ancestor contexts.
			Map<String, ViewResolver> matchingBeans =
					BeanFactoryUtils.beansOfTypeIncludingAncestors(context, ViewResolver.class, true, false);
			if (!matchingBeans.isEmpty()) {
				this.viewResolvers = new ArrayList<ViewResolver>(matchingBeans.values());
				// We keep ViewResolvers in sorted order.
				AnnotationAwareOrderComparator.sort(this.viewResolvers);
			}
		}
		...
		}

本项目我们是明确配置的`viewResolver`：

		<beans:bean
			class="org.springframework.web.servlet.view.InternalResourceViewResolver">
			<beans:property name="prefix" value="/WEB-INF/views/" />
			<beans:property name="suffix" value=".jsp" />
		</beans:bean>

`ViewResolver`有大量的子类，由于处理不同的引擎，比如：
`org.springframework.web.servlet.view.InternalResourceViewResolver`：jsp  
`org.springframework.web.servlet.view.velocity.VelocityViewResolver`: Velocity
基本逻辑都差不多，如果是jsp，使用jstl的标准逻辑`forword`等方法，如果其他的模板引擎，则自己解析，响应到response中即可。 
比如`org.springframework.web.servlet.view.velocity.VelocityViewResolver`:

		public View org.springframework.web.servlet.view.AbstractCachingViewResolver#resolveViewName(String viewName, Locale locale) throws Exception {
			if (!isCache()) {
				return createView(viewName, locale);
			}
			...
		}

		protected View createView(String viewName, Locale locale) throws Exception {
			return loadView(viewName, locale);
		}
		
		protected View org.springframework.web.servlet.view.UrlBasedViewResolver#loadView(String viewName, Locale locale) throws Exception {
			AbstractUrlBasedView view = buildView(viewName);
			View result = applyLifecycleMethods(viewName, view);
			return (view.checkResource(locale) ? result : null);
		}
		
		protected AbstractUrlBasedView org.springframework.web.servlet.view.velocity.VelocityViewResolver#buildView(String viewName) throws Exception {
			VelocityView view = (VelocityView) super.buildView(viewName);
			view.setDateToolAttribute(this.dateToolAttribute);
			view.setNumberToolAttribute(this.numberToolAttribute);
			if (this.toolboxConfigLocation != null) {
				((VelocityToolboxView) view).setToolboxConfigLocation(this.toolboxConfigLocation);
			}
			return view;
		}

就是最终调用`buildView`，返回`org.springframework.web.servlet.view.velocity.VelocityView`。最终在`VelocityView`中：

		protected void doRender(Context context, HttpServletResponse response) throws Exception {
			if (logger.isDebugEnabled()) {
				logger.debug("Rendering Velocity template [" + getUrl() + "] in VelocityView '" + getBeanName() + "'");
			}
			mergeTemplate(getTemplate(), context, response);
		}

		protected void mergeTemplate(
		Template template, Context context, HttpServletResponse response) throws Exception {

			try {
				template.merge(context, response.getWriter());
			}
			catch (MethodInvocationException ex) {
				Throwable cause = ex.getWrappedThrowable();
				throw new NestedServletException(
						"Method invocation failed during rendering of Velocity view with name '" +
						getBeanName() + "': " + ex.getMessage() + "; reference [" + ex.getReferenceName() +
						"], method '" + ex.getMethodName() + "'",
						cause==null ? ex : cause);
			}
		}
		
最终委托于`Velocity`的`merge`方法中。直接把渲染后的模板写到客户端。
