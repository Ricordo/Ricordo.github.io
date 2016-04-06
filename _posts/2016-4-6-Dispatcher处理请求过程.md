---
layout: post
title:  "Dispatcher处理请求过程"
date:   2016-04-06 16:47:38 +0800
categories: jekyll update
---

## Dispatcher处理请求过程

### FrameworkServlet

FrameworkServlet类复写了所有的处理请求方法，如`doGet()`,`doPost()`等等，其中`doOptions()`,`doTrace()`方法会根据`dispatchOptionsRequest`和`dispatchTraceRequest`属性值决定是分发还是交给父类处理，其他方法均交给`processRequest(request, response)`处理。

	/**
	 * Process this request, publishing an event regardless of the outcome.
	 * <p>The actual event handling is performed by the abstract
	 * {@link #doService} template method.
	 */
	protected final void processRequest(HttpServletRequest request, HttpServletResponse response)
			throws ServletException, IOException {

		long startTime = System.currentTimeMillis();
		Throwable failureCause = null;

		// Expose current LocaleResolver and request as LocaleContext.
		LocaleContext previousLocaleContext = LocaleContextHolder.getLocaleContext();
		LocaleContextHolder.setLocaleContext(buildLocaleContext(request), this.threadContextInheritable);

		// Expose current RequestAttributes to current thread.
		RequestAttributes previousRequestAttributes = RequestContextHolder.getRequestAttributes();
		ServletRequestAttributes requestAttributes = null;
		if (previousRequestAttributes == null || previousRequestAttributes.getClass().equals(ServletRequestAttributes.class)) {
			requestAttributes = new ServletRequestAttributes(request);
			RequestContextHolder.setRequestAttributes(requestAttributes, this.threadContextInheritable);
		}

		if (logger.isTraceEnabled()) {
			logger.trace("Bound request context to thread: " + request);
		}

		try {
			doService(request, response);
		}
		catch (ServletException ex) {
			failureCause = ex;
			throw ex;
		}
		catch (IOException ex) {
			failureCause = ex;
			throw ex;
		}
		catch (Throwable ex) {
			failureCause = ex;
			throw new NestedServletException("Request processing failed", ex);
		}

		finally {
			// Clear request attributes and reset thread-bound context.
			LocaleContextHolder.setLocaleContext(previousLocaleContext, this.threadContextInheritable);
			if (requestAttributes != null) {
				RequestContextHolder.setRequestAttributes(previousRequestAttributes, this.threadContextInheritable);
				requestAttributes.requestCompleted();
			}
			if (logger.isTraceEnabled()) {
				logger.trace("Cleared thread-bound request context: " + request);
			}

			if (logger.isDebugEnabled()) {
				if (failureCause != null) {
					this.logger.debug("Could not complete request", failureCause);
				}
				else {
					this.logger.debug("Successfully completed request");
				}
			}
			if (this.publishEvents) {
				// Whether or not we succeeded, publish an event.
				long processingTime = System.currentTimeMillis() - startTime;
				this.webApplicationContext.publishEvent(
						new ServletRequestHandledEvent(this,
								request.getRequestURI(), request.getRemoteAddr(),
								request.getMethod(), getServletConfig().getServletName(),
								WebUtils.getSessionId(request), getUsernameForRequest(request),
								processingTime, failureCause));
			}
		}
	}

`processRequest()`主要做了两件事：

- 对LocaleContext和RequestAttributes的设置和恢复。
- 处理结束后发布ServletRequestHandledEvent消息。

`LocaleContext`是一个封装了请求的本地化信息的类，`LocaleContextHolder`里面维护了两个`ThreadLocal<LocaleContext>`实例。
`ServletRequestAttributes`是一个封装了`HttpServletRequest`和`HttpSession`的类。与`LocaleContextHolder`类似，`RequestContextHolder`也封装了两个`ThreadLocal<RequestAttributes>`实例。

通过这两个Holder类的静态方法，在Spring MVC的任何地方，包括服务层，都能轻松地获得`Locale`信息和`Request`信息。

### DispatcherServlet

在FrameworkServlet中的`processRequest()`中，请求被传递给了`doService()`方法，

	/**
	 * Exposes the DispatcherServlet-specific request attributes and delegates to {@link #doDispatch}
	 * for the actual dispatching.
	 */
	@Override
	protected void doService(HttpServletRequest request, HttpServletResponse response) throws Exception {
		if (logger.isDebugEnabled()) {
			String requestUri = urlPathHelper.getRequestUri(request);
			logger.debug("DispatcherServlet with name '" + getServletName() + "' processing " + request.getMethod() +
					" request for [" + requestUri + "]");
		}

		// Keep a snapshot of the request attributes in case of an include,
		// to be able to restore the original attributes after the include.
		Map<String, Object> attributesSnapshot = null;
		if (WebUtils.isIncludeRequest(request)) {
			logger.debug("Taking snapshot of request attributes before include");
			attributesSnapshot = new HashMap<String, Object>();
			Enumeration<?> attrNames = request.getAttributeNames();
			while (attrNames.hasMoreElements()) {
				String attrName = (String) attrNames.nextElement();
				if (this.cleanupAfterInclude || attrName.startsWith("org.springframework.web.servlet")) {
					attributesSnapshot.put(attrName, request.getAttribute(attrName));
				}
			}
		}

		// Make framework objects available to handlers and view objects.
		request.setAttribute(WEB_APPLICATION_CONTEXT_ATTRIBUTE, getWebApplicationContext());
		request.setAttribute(LOCALE_RESOLVER_ATTRIBUTE, this.localeResolver);
		request.setAttribute(THEME_RESOLVER_ATTRIBUTE, this.themeResolver);
		request.setAttribute(THEME_SOURCE_ATTRIBUTE, getThemeSource());

		FlashMap inputFlashMap = this.flashMapManager.retrieveAndUpdate(request, response);
		if (inputFlashMap != null) {
			request.setAttribute(INPUT_FLASH_MAP_ATTRIBUTE, Collections.unmodifiableMap(inputFlashMap));
		}
		request.setAttribute(OUTPUT_FLASH_MAP_ATTRIBUTE, new FlashMap());
		request.setAttribute(FLASH_MAP_MANAGER_ATTRIBUTE, this.flashMapManager);

		try {
			doDispatch(request, response);
		}
		finally {
			// Restore the original attribute snapshot, in case of an include.
			if (attributesSnapshot != null) {
				restoreAttributesAfterInclude(request, attributesSnapshot);
			}
		}
	}

在`doService()`法中，首先判断了是否为include请求，如果是include请求，先对request的Attribute做个快照备份，做完快照备份后又在request的attribute中设置了一些Spring MVC框架的属性，如WebApplicationContext，LocaleResolver，themeResolver和ThemeSource，这些属性在之后的Handler和view中需要使用。再之后的三个属性都与FlashMap相关，主要用于Redirect转发时参数的传递。`INPUT_FLASH_MAP_ATTRIBUTE`用于保存上次请求中转发过来的属性，`OUTPUT_FLASH_MAP_ATTRIBUTE`用于保存本次请求需要转发的属性，`lashMapManager`用于管理它们。

做完这些操作之后，请求被传递给`doDispatch(request, response)`。

	processedRequest = checkMultipart(request);

`processedRequest`是实际处理时所使用的request，如果不是上传请求，即contentType不是`multipart/`开头的请求都直接使用接收到的request，否则使用封装过的上传类型的request。

	mappedHandler = getHandler(processedRequest, false);
				if (mappedHandler == null || mappedHandler.getHandler() == null) {
					noHandlerFound(processedRequest, response);
					return;
				}

`mappedHandler`是根据请求查找的处理器链，如果没有找到，则直接返回。

	HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());

根据相应的处理器链查找相应的HandlerAdapter。

	String method = request.getMethod();
				boolean isGet = "GET".equals(method);
				if (isGet || "HEAD".equals(method)) {
					long lastModified = ha.getLastModified(request, mappedHandler.getHandler());
					if (logger.isDebugEnabled()) {
						String requestUri = urlPathHelper.getRequestUri(request);
						logger.debug("Last-Modified value for [" + requestUri + "] is: " + lastModified);
					}
					if (new ServletWebRequest(request, response).checkNotModified(lastModified) && isGet) {
						return;
					}
				}

处理GET，HEAD请求的Last-Modified，如果资源过期返回新的资源，未过期则返回304状态码。

	HandlerInterceptor[] interceptors = mappedHandler.getInterceptors();
				if (interceptors != null) {
					for (int i = 0; i < interceptors.length; i++) {
						HandlerInterceptor interceptor = interceptors[i];
						if (!interceptor.preHandle(processedRequest, response, mappedHandler.getHandler())) {
							triggerAfterCompletion(mappedHandler, interceptorIndex, processedRequest, response, null);
							return;
						}
						interceptorIndex = i;
					}
				}

调用相应的Interceptor的`preHandle`方法。

	mv = ha.handle(processedRequest, response, mappedHandler.getHandler());

最关键的代码，调用HandlerAdapter的`handle`方法，Controller就是在这里执行的。

	if (mv != null && !mv.hasView()) {
					mv.setViewName(getDefaultViewName(request));
				}

设置view。

	if (interceptors != null) {
					for (int i = interceptors.length - 1; i >= 0; i--) {
						HandlerInterceptor interceptor = interceptors[i];
						interceptor.postHandle(processedRequest, response, mappedHandler.getHandler(), mv);
					}
				}

执行Interceptor的`postHandle`方法。

到这里，整个请求的过程就结束了，接下来就是处理异常，渲染页面和触发Interceptor的`afterCompletion`方法。

	if (mv != null && !mv.wasCleared()) {
				render(mv, processedRequest, response);
				if (errorView) {
					WebUtils.clearErrorRequestAttributes(request);
				}
			}

渲染页面在`render`方法中执行。

	triggerAfterCompletion(mappedHandler, interceptorIndex, processedRequest, response, null);

触发Interceptor的`afterCompletion`方法，与`PostHandle`相同，也是按反方向执行的。

