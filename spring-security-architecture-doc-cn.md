## **Spring Security 架构**
本指南是Spring Security的入门，致力于深入了解框架设计和基本构建块。 虽然仅设计应用程序安全性的基础知识，但是这样做可以清除开发人员使用Spring Security时可能遇到的一些困惑并清楚混淆。 要做到这一点，我们首先来看看在Web应用程序中使用过滤器以及较普通地使用方法的方式的安全性。 如果您需要了解高级别安全应用程序的工作方式，以及如何定制安全的应用程序，或只学习如何考量应用程序安全性，请使用本指南。

本指南不是解决更多的基本问题（还有其他来源）的手册或配方，但对于初学者或专家来说可能还是有用的。 Spring Boot在本指南中也被引用了很多，因为它为安全应用程序提供了一些默认的支持，并且了解它如何与整个体系结构相适应是非常有用的。 所有这些原则同样适用于不使用Spring Boot的应用程序。

## **身份验证和访问控制**
应用程序安全性归结为两个或多个独立的问题：身份验证（authentication 你是谁？）和授权（authorization你可以做什么？）。 有时候，人们常会说“访问控制”而不是说“授权”，这会让人感到困惑，这是由于“授权”在其他地方有“超载于本身”的含义。 Spring Security有一个旨，即将认证从授权的体系结构分开，并在认证和授权有各自的策略和扩展点。

### **认证**
认证的主要策略接口是AuthenticationManager，它只有一个方法：
```java
public interface AuthenticationManager {
	Authentication authenticate(Authentication authentication)
    throws AuthenticationException;
}
```
AuthenticationManager可以在authenticate（）方法中做以下三件事之一：

如果它可以验证输入的请求是一个有效的委托人，则返回一个认证（通常是authenticated = true）。

如果它认为输入表示一个无效的主体，则抛出一个AuthenticationException。

如果无法确定，则返回null。

AuthenticationException是一个运行时异常。 通常由应用程序以通用方式处理，具体取决于应用程序的风格或目的。 换句话说，用户的代码通常不应该去处理这种异常。 例如，Web UI会弹出一个页面，表示认证失败，或者后端HTTP服务发送401响应，有没有WWW-Authenticate标头，具体取决于上下文。

AuthenticationManager最常用的实现是ProviderManager，它委托给一个AuthenticationProvider实例链。 AuthenticationProvider有点像AuthenticationManager，但它有一个额外的方法来允许调用者查询它是否支持传入的认证类型：
```java
 public interface AuthenticationProvider {

	Authentication authenticate(Authentication authentication)
			throws AuthenticationException;

	boolean supports(Class<?> authentication);

}
```
supports（）方法中的Class <？>参数实际上是Class <？ 扩展认证>（传递到authenticate（）方法的传入是否被支持才会涉及到它）。 通过委托给一个AuthenticationProviders链，一个ProviderManager可以支持应用程序中多个不同的认证机制。 如果一个ProviderManager不能识别一个特定的Authentication类型，它将被跳过。

一个ProviderManager有一个可选的父类，即如果所有的Provider返回null，可以参考这个父类。 如果父节点不可用，则null验证会导致AuthenticationException。

有时应用程序具有受保护资源的逻辑组（例如所有与路径模式/ api / **相匹配的Web资源），并且每个组可以具有其自己的专用AuthenticationManager。 通常，每个都是一个ProviderManager，他们共享一个parent。 parent是一种“全局”资源，充当所有提供者的后备。
<img src="http://img.blog.csdn.net/20171211103357420" width=400px />

### **定制身份验证管理器**
Spring Security提供了一些配置助手来快速获得应用程序中设置的通用身份验证管理器功能。 最常用的帮助程序是AuthenticationManagerBuilder，它非常适用于设置内存，JDBC或LDAP的用户详细信息，或添加自定义的UserDetailsService。 以下是配置全局（父母）AuthenticationManager的应用程序示例：
```java
@Configuration
public class ApplicationSecurity extends WebSecurityConfigurerAdapter {

	... // web stuff here

	@Autowired
	public initialize(AuthenticationManagerBuilder builder, DataSource dataSource) {
		builder.jdbcAuthentication()
	    .dataSource(dataSource)
	    .withUser("dave")
	    .password("secret")
	    .roles("USER");
	}
}
```
这个例子仅涉及到一个Web应用程序，但是AuthenticationManagerBuilder的使用还有更加广泛的适用场景（更多细节请参见下面关于Web应用程序安全性的实现）。 请注意，AuthenticationManagerBuilder是@Autowired到@Bean中的一个方法 - 这使得构建了全局（父母）AuthenticationManager。 相反，如果我们这样做了：
```java
@Configuration
public class ApplicationSecurity extends WebSecurityConfigurerAdapter {

	@Autowired
	DataSource dataSource;

	... // web stuff here

	@Override
	public configure(AuthenticationManagerBuilder builder) {
		builder.jdbcAuthentication()
		.dataSource(dataSource)
		.withUser("dave")
		.password("secret")
		.roles("USER");
	}
}
```
（在配置器(configurer)中使用一个方法的@Override），那么AuthenticationManagerBuilder仅可用于构建一个“本地”AuthenticationManager，它是全局的一个子AuthenticationManager。 在Spring Boot应用程序中，您可以将全局的通过@Autowired注入到另一个bean，但除非您自己明确暴露，否则不能使用本地变量。

Spring Boot提供了一个默认的全局AuthenticationManager（只有一个用户），你可以提供你自己的AuthenticationManager类型的bean。 默认的AuthenticationManager是足够安全的，你不必担心太多，除非你主动去自定义一个全局AuthenticationManager。 如果你做任何构建AuthenticationManager的配置，你可以经常针对于此执行你正在保护的资源，而不用担心影响全局的。

### **授权或访问控制**
一旦认证成功，我们可以继续进行授权，这里的核心策略是AccessDecisionManager。 这个框架提供了三个实现，所有这三个实现都是委托给一个AccessDecisionVoter链，有点像ProviderManager委托给AuthenticationProviders。
```java
boolean supports(ConfigAttribute attribute);

boolean supports(Class<?> clazz);

int vote(Authentication authentication, S object,
        Collection<ConfigAttribute> attributes);
```
该Object在AccessDecisionManager和AccessDecisionVoter的签名中是完全通用的 - 它代表用户可能想要访问的任何内容（Web资源或Java类中的方法是最常见的两种情况）。 ConfigAttributes也是相当通用的，用一些元数据来装饰安全对象的，这些元数据决定了访问它所需的权限级别。 ConfigAttribute是一个接口，但它只有一个非常通用的方法，并返回一个String，所以这些字符串以某种方式编码资源所有者的意图，表达关于谁可以访问的规则。 一个典型的ConfigAttribute是一个用户角色的名字（比如ROLE_ADMIN或者ROLE_AUDIT），它们通常有特殊的格式（比如 ROLE_ 前缀）或者表示需要评估的表达式。

大多数人只是使用默认的AccessDecisionManager，即AffirmativeBased（如果没有选民拒绝，然后授予访问权限）。 任何定制都倾向于发生在选民身上，或者增加新的定制，或者改变现有的工作方式。

使用Spring表达式语言（SpEL）ConfigAttributes是很常见的，例如isFullyAuthenticated（）&& hasRole（'FOO'）。 这由AccessDecisionVoter支持，可以处理表达式并为它们创建一个上下文。 要扩展可以处理的表达式的范围，需要自定义实现SecurityExpressionRoot，有时还需要实现SecurityExpressionHandler。

### **网络完全**
Web层中的Spring Security（用于UI和HTTP后端）基于Servlet过滤器，所以首先查看过滤器的作用是很有帮助的。 下图显示了单个HTTP请求的处理程序的典型分层结构。

<img src="http://img.blog.csdn.net/20171211111951296" width=200px />

客户端向应用程序发送一个请求，容器根据请求URI的路径决定哪些过滤器和哪个servlet适用于它。单个请求最多可以被一个servlet处理，但是过滤器形成一个链，所以它们是有序的，事实上，如果一个过滤器想要处理请求，过滤器可以否决链的其余部分。过滤器还可以修改在下游过滤器和servlet中使用的请求和响应。过滤器链的顺序是非常重要的，Spring Boot通过两种机制来管理它：一种是Filter类型的@Beans可以有@Order或实现Ordered，另一种是它们可以是FilterRegistrationBean本身具有订单API作为自身的一部分。一些现成的过滤器定义了自己的常量，以帮助表示它们喜欢相互之间的顺序（例如，来自Spring会话的SessionRepositoryFilter具有Integer.MIN_VALUE + 50的DEFAULT_ORDER，这告诉我们它在链中可能比较靠前）。

Spring Security作为一个单独的过滤器安装在链中，其配置类型为FilterChainProxy，原因很快就会出现。 在Spring Boot应用程序中，安全筛选器是ApplicationContext中的一个@Bean，并且默认安装它，以便将其应用于每个请求。 它被安装在由SecurityProperties.DEFAULT_FILTER_ORDER定义的位置，而该位置又由FilterRegistrationBean.REQUEST_WRAPPER_FILTER_MAX_ORDER（Spring Boot应用程序在包装请求时修改其行为的期望过滤器的最大顺序）锚定。 除此之外还有更多的内容：从容器的角度来看，Spring Security是一个单一的过滤器，但里面还有额外的过滤器，每个过滤器都扮演着特殊的角色。 这是一张图片：

<img src="http://img.blog.csdn.net/20171211114217804" width=400px />

事实上，在安全过滤器中甚至还有一层间接寻址：它通常作为DelegatingFilterProxy安装在容器中，而不必是Spring @Bean。 代理委托给一个总是@Bean的FilterChainProxy，通常使用固定名称springSecurityFilterChain。 FilterChainProxy包含所有安全逻辑，内部安排为过滤器的一个或多个链。 所有的过滤器都有相同的API（他们都实现了Servlet规范中的Filter接口），他们都有机会否决链的其余部分。

在同一个顶级FilterChainProxy中，可以有多个由Spring Security管理的过滤器链，并且容器都是未知的。 Spring Security筛选器包含一个筛选器链列表，并向与之匹配的第一个链派发一个请求。 下图显示了匹配请求路径（/ foo /\** 匹配/\**之前）的分派情况。 这是非常普遍的，但不是匹配请求的唯一方法。 这个调度过程最重要的特点是只有一个链处理请求。

<img src="http://img.blog.csdn.net/20171211115301003" width=400px />

没有自定义安全配置的vanilla Spring Boot应用程序有几个（称为n）过滤器链，通常n = 6。 第一个（n-1）链只是为了忽略静态资源模式，如/ css / **和/ images / **，错误视图/error（路径可以由用户来控制， SecurityProperties配置bean）。 最后一个链匹配所有路径/ **，并且更加活跃，包含认证，授权，异常处理，会话处理，头文件写等逻辑。默认情况下，链中总共有11个过滤器，但通常情况下 用户不必关心使用哪个过滤器以及何时使用过滤器。
>**注意**
>Spring Security内部的所有过滤器对于容器是未知的，这一点很重要，特别是在Spring Boot应用程序中，默认情况下，所有类型为Filter的@Beans都会自动注册到容器中。 所以，如果你想添加一个自定义的过滤器到安全链，你不需要把它作为一个@Bean或者包装在一个FilterRegistrationBean中，因为那样明确地禁止了容器的注册

### **创建和自定义筛选链**
Spring Boot应用程序（具有/ **请求匹配器的应用）中的默认回退过滤器链具有SecurityProperties.BASIC_AUTH_ORDER预定义的顺序。 您可以通过设置security.basic.enabled = false将其完全关闭，或者可以将其用作后备，只需定义其他较低顺序的规则即可。 要做到这一点，只需添加一个类型为WebSecurityConfigurerAdapter（或WebSecurityConfigurer）的@Bean，然后用@Order修饰。 例：
```java
@Configuration
@Order(SecurityProperties.BASIC_AUTH_ORDER - 10)
public class ApplicationConfigurerAdapter extends WebSecurityConfigurerAdapter {
	@Override
	protected void configure(HttpSecurity http) throws Exception {
		http.antMatcher("/foo/**")
	    ...;
  }
}
```

这个bean将使Spring Security添加一个新的过滤器链，并在回退之前对其进行排序。

对于一组资源，许多应用程序具有完全不同的访问规则。 例如，托管UI和支持API的应用程序可能支持基于cookie的身份验证，重定向到UI部件的登录页面，以及基于令牌的身份验证，对未经身份验证的API部件请求进行401响应。 每一组资源都有自己的WebSecurityConfigurerAdapter，它具有唯一的顺序和自己的请求匹配器。 如果匹配规则重叠，则最早排序的过滤器链将获胜。

### **请求匹配调度和授权**
安全过滤器链（或等同于WebSecurityConfigurerAdapter）具有请求匹配器，用于决定是否将其应用于HTTP请求。 一旦决定采用特定的过滤器链，则不会应用其他过滤器。 但是在一个过滤链中，通过在HttpSecurity配置器中设置额外的匹配器，可以对授权进行更细粒度的控制。 例：
```java
@Configuration
@Order(SecurityProperties.BASIC_AUTH_ORDER - 10)
public class ApplicationConfigurerAdapter extends WebSecurityConfigurerAdapter {
	@Override
	protected void configure(HttpSecurity http) throws Exception {
	    http.antMatcher("/foo/**")
		.authorizeRequests()
        .antMatchers("/foo/bar").hasRole("BAR")
        .antMatchers("/foo/spam").hasRole("SPAM")
        .anyRequest().isAuthenticated();
	}
}
```

配置Spring Security最容易犯的一个错误是忘记这些匹配器适用于不同的进程，哪个是整个过滤器链的请求匹配器，哪些又是只适用于特定应用的访问规则。

### **将应用安全规则与执行器规则相结合**
如果你使用Spring Boot Actuator作为管理端点，你可能希望它们是安全的，默认情况下它们是。 事实上，只要将执行器添加到安全的应用程序中，您就会得到一个仅适用于执行器端点的附加过滤器链。 它是由一个请求匹配器定义的，它只匹配执行器端点，它的顺序是ManagementServerProperties.BASIC_AUTH_ORDER，比默认的SecurityProperties后备过滤器少5个，所以在回退之前会被查询。

如果您希望您的应用程序安全规则适用于执行器端点，则可以添加一个比执行器更早订购的过滤器链，以及包含所有执行器端点的请求匹配器。 如果您更喜欢执行器端点的默认安全设置，那么最简单的方法是在执行器之后添加自己的过滤器，但早于回退（例如ManagementServerProperties.BASIC_AUTH_ORDER + 1）。 例：
```java
@Configuration
@Order(ManagementServerProperties.BASIC_AUTH_ORDER + 1)
public class ApplicationConfigurerAdapter extends WebSecurityConfigurerAdapter {
	@Override
	protected void configure(HttpSecurity http) throws Exception {
		http.antMatcher("/foo/**")
		...;
	}
}
```

>**注意**
>Web层中的Spring Security目前与Servlet API绑定在一起，因此只有在servlet容器中运行应用程序（嵌入式或其他方式）时才是真正适用的。 但是，它并不是绑定到Spring MVC或Spring Web堆栈的其余部分，所以它可以用在任何servlet应用程序中，例如使用JAX-RS的应用程序。

## **方法安全**
Spring Security基本上是线程绑定的，因为它需要使当前的身份验证委托人可用于各种下游消费者。 基本构建块是SecurityContext，其中可能包含一个身份验证（并且当用户登录时它将是一个明确验证的身份验证）。 你总是可以通过SecurityContextHolder中的静态便利方法访问和操作SecurityContext，而后者只需操作一个TheadLocal
```java
SecurityContext context = SecurityContextHolder.getContext();
Authentication authentication = context.getAuthentication();
assert(authentication.isAuthenticated);
```
用户应用程序代码执行此操作并不常见，但如果您需要编写自定义身份验证筛选器（尽管Spring Security中有基类可用于避免需要的地方 使用SecurityContextHolder）。

如果您需要访问Web端点中当前已通过身份验证的用户，则可以在@RequestMapping中使用方法参数。 例如。
```java
@RequestMapping("/foo")
public String foo(@AuthenticationPrincipal User user) {
	... // do stuff with user
}
```

这个注解将当前Authentication从SecurityContext中抽出，并调用其上的getPrincipal（）方法来产生方法参数。 认证中的委托人类型取决于用于验证认证的认证管理器，所以这对于获得对用户数据的类型安全引用是一个有用的小技巧。

如果使用Spring Security，则HttpServletRequest中的Principal将是Authentication类型，因此您也可以直接使用它：
```java
@RequestMapping("/foo")
public String foo(Principal principal) {
	Authentication authentication = (Authentication) principal;
	User = (User) authentication.getPrincipal();
	... // do stuff with user
}
```
如果你需要编写在没有使用Spring Security的情况下工作的代码，那么这有时候会很有用（你需要在加载Authentication类时更加防御）。

### **异步处理安全方法**
由于SecurityContext是线程绑定的，因此如果要执行任何调用安全方法的后台处理，例如 与@Async，你需要确保上下文传播。 这可以归结为将SecurityContext包装在后台执行的任务（Runnable，Callable等）中。 Spring Security提供了一些帮助器，使之更容易，比如Runnable和Callable的包装器。 要将SecurityContext传播到@Async方法，您需要提供一个AsyncConfigurer并确保Executor的类型正确：
```java
@Configuration
public class ApplicationConfiguration extends AsyncConfigurerSupport {

	@Override
	public Executor getAsyncExecutor() {
		return new DelegatingSecurityContextExecutorService(Executors.newFixedThreadPool(5));
  }

}
```
