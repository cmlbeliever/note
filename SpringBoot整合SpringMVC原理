通过SpringBoot整合各个框架是越来越方便了，整合SpringMVC只需要添加对应的starer依赖即可。

```
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-web</artifactId>
</dependency>
```
而且还配备了Tomcat的starter

```
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-web</artifactId>
</dependency>
```
这样，只需要根据自身需求，设置配置文件。启动web服务器只需要运行java application就可以了，不再需要部署到tomcat服务了。

之前一直很好奇，使用SpringMVC时需要在web.xml上配置DispatcherServlet。而整合了SpringBoot后为什么就不需要配置了，下面就进行完整的分析。

**看着累？可以直接看步骤7，核心分析。**

1、寻找入口，找到WebServlet自动配置类：EmbeddedServletContainerAutoConfiguration

```
org.springframework.boot.autoconfigure.web.EmbeddedServletContainerAutoConfiguration

@AutoConfigureOrder(Ordered.HIGHEST_PRECEDENCE)
@Configuration
@ConditionalOnWebApplication
@Import(BeanPostProcessorsRegistrar.class)
public class EmbeddedServletContainerAutoConfiguration{
	...省略代码
}
```

> SpringBoot 自动配置功能类都以AutoConfiguration结尾

2、注入需要的Bean
从类上的注解可以看出，导入了BeanPostProcessorsRegistrar，来添加EmbeddedServletContainerCustomizerBeanPostProcessor。首先会查看工程是否有自定的EmbeddedServletContainerCustomizerBeanPostProcessor，如果没有，则注入默认的EmbeddedServletContainerCustomizerBeanPostProcessor。代码如下：

```
@Override
public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata,BeanDefinitionRegistry registry) {
			if (this.beanFactory == null) {
				return;
			}
			if (ObjectUtils.isEmpty(this.beanFactory.getBeanNamesForType(
	EmbeddedServletContainerCustomizerBeanPostProcessor.class, true,
					false))) {
				registry.registerBeanDefinition(					"embeddedServletContainerCustomizerBeanPostProcessor",
						new RootBeanDefinition(
		EmbeddedServletContainerCustomizerBeanPostProcessor.class));
			}
			if (ObjectUtils.isEmpty(this.beanFactory.getBeanNamesForType(
					ErrorPageRegistrarBeanPostProcessor.class, true, false))) {
				registry.registerBeanDefinition("errorPageRegistrarBeanPostProcessor",
						new RootBeanDefinition(
								ErrorPageRegistrarBeanPostProcessor.class));

			}
		}
```

> 实现ImportBeanDefinitionRegistrar接口，实现注入需要的Bean到Spring容器中，Mybatis（MapperScannerRegistrar）也是通过此接口来完成Mapper类的定义。

3、步骤2注入了bean：EmbeddedServletContainerCustomizerBeanPostProcessor，该类实现了在ConfigurableEmbeddedServletContainer对象初始化前，进行行必要的参数配置。

 1. 获取所有EmbeddedServletContainerCustomizer对象
 2. 调用EmbeddedServletContainerCustomizer.customize方法
 3. EmbeddedServletContainerCustomizer实现类根据自身需求设置WebServlet容器参数（如：端口号、连接数等等）
核心代码如下：
```
	@Override
	public Object postProcessBeforeInitialization(Object bean, String beanName)throws BeansException {
		if (bean instanceof ConfigurableEmbeddedServletContainer) {		postProcessBeforeInitialization((ConfigurableEmbeddedServletContainer) bean);
		}
		return bean;
	}
private void postProcessBeforeInitialization(ConfigurableEmbeddedServletContainer bean) {
		for (EmbeddedServletContainerCustomizer customizer : getCustomizers()) {
			customizer.customize(bean);
		}
	}
```

> ConfigurableEmbeddedServletContainer：是Web容器的接口，默认注入的有
> ![这里写图片描述](http://img.blog.csdn.net/20170416164649580?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvY21sX2Jsb2c=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
> BeanPostProcessor ： 是Spring容器的回调接口，在所有Bean初始化之前和之后分别回调此接口的postProcessBeforeInitialization，postProcessAfterInitialization方法。这样就可以根据需求在Bean初始化前后配置设置需要的功能。

通过步骤1-3完成了Web容器启动前的参数配置功能。

4、EmbeddedWebApplicationContext入场
Spring容器配置加载完成后，会回调EmbeddedWebApplicationContext.refresh方法。EmbeddedWebApplicationContext在执行refresh方法中，调用了onRefresh方法进行ServletContainer配置。代码如下：

```
@Override
	public final void refresh() throws BeansException, IllegalStateException {
			 ...省略
			super.refresh();
			 ...省略
	}

	@Override
	protected void onRefresh() {
        ...省略
	 	//创建ServletContainer
		createEmbeddedServletContainer();
		 ...省略
	}
```

> EmbeddedWebApplicationContext实现了接口ConfigurableApplicationContext,Spring容器配置加载完成后会回调所有的ConfigurableApplicationContext对象的refresh方法。

 1. 在onRefresh方法中，获取EmbeddedServletContainerFactory对象，因为工程上使用Tomcat，所以这里就是TomcatEmbeddedServletContainerFactory
 2. 执行EmbeddedServletContainerFactory.getEmbeddedServletContainer方法

5、TomcatEmbeddedServletContainerFactory.getEmbeddedServletContainer这里是Tomcat容器核心功能完的地方。主要完成了对Tomcat配置（不是这篇重点，省略代码），在configureContext方法添加Tomcat容器启动回调接口（重点）。

```
protected void configureContext(Context context,
			ServletContextInitializer[] initializers) {
		TomcatStarter starter = new TomcatStarter(initializers);
		if (context instanceof TomcatEmbeddedContext) {
			// Should be true
			((TomcatEmbeddedContext) context).setStarter(starter);
		}
		...省略
	}

```

> ServletContainerInitializer是Tomcat容器启动的一个回调接口。
> 在Tomcat启动前，SpringBoot通过TomcatStarter完成Servlet，Filter等Web组件的组注入

6、TomcatStarter，在Tomcat启动后回调onStartup。

**7、EmbeddedWebApplicationContext在onStartup回调中完成SpringMvc功能注入**
---------------------------------------------------------------

7.1、在selfInitialize方法中获取到所有ServletContextInitializer对象，并调用其onStartup方法，
7.2、ServletContextInitializer实现类如下：
![这里写图片描述](http://img.blog.csdn.net/20170416174603474?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvY21sX2Jsb2c=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
7.3、通过上图就很清楚的说明了Servlet，Filter等Web组件实现类
7.4、在ServletRegistrationBean向ServletContainer添加Servlet

```
@Override
	public void onStartup(ServletContext servletContext) throws ServletException {
		Assert.notNull(this.servlet, "Servlet must not be null");
		String name = getServletName();
		if (!isEnabled()) {
			logger.info("Servlet " + name + " was not registered (disabled)");
			return;
		}
		logger.info("Mapping servlet: '" + name + "' to " + this.urlMappings);
		Dynamic added = servletContext.addServlet(name, this.servlet);
		if (added == null) {
			logger.info("Servlet " + name + " was not registered "
					+ "(possibly already registered?)");
			return;
		}
		configure(added);
	}

```

7.5、这里也就解释了SpringBoot官方文档70.1节上为什么是通过RegistrationBean添加Servlet与Filter的原因了。
![这里写图片描述](http://img.blog.csdn.net/20170416175229482?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvY21sX2Jsb2c=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

7.6、DispatcherServletAutoConfiguration.DispatcherServletRegistrationConfiguration此处添加SpringMVC核心功能类DispatcherServlet

```
	@Bean(name = DEFAULT_DISPATCHER_SERVLET_REGISTRATION_BEAN_NAME)
		@ConditionalOnBean(value = DispatcherServlet.class, name = DEFAULT_DISPATCHER_SERVLET_BEAN_NAME)
		public ServletRegistrationBean dispatcherServletRegistration(
				DispatcherServlet dispatcherServlet) {
			ServletRegistrationBean registration = new ServletRegistrationBean(
					dispatcherServlet, this.serverProperties.getServletMapping());
			registration.setName(DEFAULT_DISPATCHER_SERVLET_BEAN_NAME);
			registration.setLoadOnStartup(
					this.webMvcProperties.getServlet().getLoadOnStartup());
			if (this.multipartConfig != null) {
				registration.setMultipartConfig(this.multipartConfig);
			}
			return registration;
		}
```


以上只是将重要点抽出来说明，贴上全部源码也是无意义的。要理解其中过程还需要自行查看源码。

通过以上步骤分析了SpringBoot集成SpringMVC和Tomcat功能简要步骤。其实只要找到了入口，即可Debug一步一步的走下去，来查看内部实现。


----------
**总结**

> 通过以上分析和Mybatis功能分析，发现满满的都是套路。在SpringBoot上实现自定义Starter功能应该都是如下套路：
> 1、在自定义的XXAutoConfiguration上Import一个ImportBeanDefinitionRegistrar来注入指定Bean
> 2、添加自定义的BeanPostProcessor在Bean初始化之前或之后完成配置功能或初始化某些依赖功能

以上代码分析来自我的工程SpringBootLearning
**SpringBootLean 是对springboot学习与研究项目，是根据实际项目的形式对进行配置与处理，欢迎star与fork。**
[oschina 地址]
http://git.oschina.net/cmlbeliever/SpringBootLearning
[github 地址]
https://github.com/cmlbeliever/SpringBootLearning
