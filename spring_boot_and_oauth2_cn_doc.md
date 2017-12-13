## **Spring Boot and OAuth2**
本指南将向您展示如何使用OAuth2和Spring Boot构建一个使用“社交登录”功能做各种事情的应用程序示例。 它从一个简单的单一提供者单点登录开始，并运行一个带有身份验证提供程序（Facebook或Github）的OAuth2自我托管的授权服务器。 这些示例都是在后端使用Spring Boot和Spring OAuth的单页应用程序，在前端使用纯jQuery，如果需要转换到不同的JavaScript或服务器端渲染框架，其变动代价是很小的。

有几个样本相互构建，增加了新的功能：

**simple**：一个非常基本的静态应用程序，只有一个主页，并通过Spring Boot的EnableOAuth2Sso无条件登录（如果您访问主页，您将自动重定向到Facebook）。

**click**：添加用户必须点击登录的显式链接。

**logout**：为注册用户添加一个注销链接。

**manual**：通过取消选中并手动配置所有配置项，演示@ EnableOAuth2Sso的工作方式。

**github**：添加了第二个登录提供者（Github），所以用户可以在主页上选择使用哪一个（原本仅有一个Facebook）。

**auth-server**：将应用程序变成一个完全成熟的OAuth2授权服务器，能够发出自己的令牌，但仍然使用外部OAuth2提供程序进行身份验证（检验证与授权分离）。

**custom-error**：为没有通过身份验证的用户添加错误消息，以及基于Github API的自定义身份验证。

>可以在源代码中跟踪功能梯形图的应用程序迁移到下一个应用程序所需的更改（源代码位于Github中）。 版本库中的前6个更改正在转换为单个应用程序，以便您可以轻松查看差异。 在早期提交的应用程序和在指南中看到的最终状态之间，您可能看到的任何进一步差异都是表面上的，并不能说明什么。

它们中的每一个项目都可以被导入到一个IDE中，并且都有一个可以在IDE中运行的主类SocialApplication。 他们都在http：// localhost：8080上提供了一个主页（如果您想登录并查看内容，所有这些都需要您至少有一个Facebook帐户）。 你也可以使用mvn spring-boot：run或通过构建jar文件并使用mvn package和java -jar target / *。jar（根据Spring Boot文档和其他可用文档）运行命令行中的所有应用程序。。 如果你在顶层使用包装器，则不需要安装Maven。

```cmd
$ cd simple
$ ../mvnw package
$ java -jar target/*.jar
```

>这些应用程序都在localhost:8080上运行，因为他们使用在Facebook和Github上注册的OAuth2客户端来访问该地址。 要在不同的主机或端口上运行它们，您需要注册自己的应用程序，并将凭据放在配置文件中。 如果您使用默认值，则不会在本地主机之外泄漏您的Facebook或Github凭据，但要小心您在互联网上公开的内容，并且不要将您自己的应用程序注册信息置于开源代码管理中。

## **Single Sign On With Facebook**
在本节中，我们创建一个使用Facebook进行身份验证的最小应用程序。 如果我们利用Spring Boot中的自动配置功能，这将是相当容易的。

### **Creating a New Project**
首先，我们需要创建一个Spring Boot应用程序，可以通过多种方式来完成。 最简单的是去http://start.spring.io并生成一个空的项目（选择“Web”依赖项作为起点）。 等同于在命令行上执行此操作：
```cmd
$ mkdir ui && cd ui
$ curl https://start.spring.io/starter.tgz -d style=web -d name=simple | tar -xzvf -
```

然后，您可以将该项目导入到您最喜欢的IDE中（默认情况下，这是一个普通的Maven Java项目），或者只是在命令行上用“mvn”构建工程。

### **Add a Home Page**

在您的新项目中，在“src / main / resources / static”文件夹中创建一个index.html。 你需要添加一些样式表和Java脚本链接，所以结果如下所示：

**index.html**
```html
<!doctype html>
<html lang="en">
<head>
    <meta charset="utf-8"/>
    <meta http-equiv="X-UA-Compatible" content="IE=edge"/>
    <title>Demo</title>
    <meta name="description" content=""/>
    <meta name="viewport" content="width=device-width"/>
    <base href="/"/>
    <link rel="stylesheet" type="text/css" href="/webjars/bootstrap/css/bootstrap.min.css"/>
    <script type="text/javascript" src="/webjars/jquery/jquery.min.js"></script>
    <script type="text/javascript" src="/webjars/bootstrap/js/bootstrap.min.js"></script>
</head>
<body>
	<h1>Demo</h1>
	<div class="container"></div>
</body>
</html>
```

这些对于演示OAuth2登录功能来说都不是必须的，但是我们希望最终有一个好看的用户界面，所以我们不妨从主页中的一些基本的东西开始。

如果您启动应用程序并加载主页，可能是因为样式表尚未加载。 所以我们也需要将样式添加进去，我们可以通过添加一些依赖来实现：

**pom.xml**
```xml
<dependency>
	<groupId>org.webjars</groupId>
	<artifactId>jquery</artifactId>
	<version>2.1.1</version>
</dependency>
<dependency>
	<groupId>org.webjars</groupId>
	<artifactId>bootstrap</artifactId>
	<version>3.2.0</version>
</dependency>
<dependency>
	<groupId>org.webjars</groupId>
	<artifactId>webjars-locator</artifactId>
</dependency>
```

我们添加了Twitter bootstrap和jQuery（这就是我们现在所需要的）。 另一个依赖是webjars“定位器”，它由webjars站点作为一个库提供，Spring可以使用这个定位器在webjars中定位静态资源，而不需要知道确切的版本（因此versionless / webjars / **链接 在index.html中）。 只要不关闭MVC自动配置，webjar定位器在Spring Boot应用程序中被默认激活。

随着这些变化，我们应该有一个漂亮的主页作为我们的应用程序首页。

### **Securing the Application**
为了使应用程序安全，我们只需要添加Spring Security作为依赖。 如果我们这样做，默认情况下是使用HTTP Basic来保护它。不过，既然我们想做一个“社交”登录（委托给Facebook），我们也添加了Spring Security OAuth2依赖项：

**pom.xml**
```xml
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-security</artifactId>
</dependency>
<dependency>
	<groupId>org.springframework.security.oauth</groupId>
	<artifactId>spring-security-oauth2</artifactId>
</dependency>
```

为了建立到Facebook的链接，我们需要在我们的主类上使用一个@EnableOAuth2Sso注解：

**SocialApplication.java**
```java
@SpringBootApplication
@EnableOAuth2Sso
public class SocialApplication {

  ...

}
```
和一些配置（将application.properties转换为YAML以获得更好的可读性）：

**application.yml**
```yml
security:
  oauth2:
    client:
      clientId: 233668646673605
      clientSecret: 33b17e044ee6a4fa383f46ec6e28ea1d
      accessTokenUri: https://graph.facebook.com/oauth/access_token
      userAuthorizationUri: https://www.facebook.com/dialog/oauth
      tokenName: oauth_token
      authenticationScheme: query
      clientAuthenticationScheme: form
    resource:
      userInfoUri: https://graph.facebook.com/me
...
```

该配置是指在其开发者站点上向Facebook注册的客户端应用程序，其中您必须为该应用程序提供注册的重定向（主页）。 该示例是注册到“localhost:8080”，所以它是个只能运行在本地的应用程序。

有了这些配置，你可以再次运行应用程序，并访问http://localhost:8080的主页。 而不是被重定向的Facebook登录主页。 当你访问本地主页，重定向并选择您要求的任何授权，您将被重定向回到本地应用程序的主页。 如果您已经通过Facebook认证了身份，即使您使用新的浏览器打开它，不使用Cookie和缓存数据，也不需要在本地重新进行身份验证。（这就是单点登录的含义。）

>如果您正在着手于示例应用程序的这一部分，请务必清除您的Cookie和HTTP Basic凭据的浏览器缓存。 在Chrome浏览器中，打开一个新的隐身窗口是此操作的最佳方式。

授予对此示例的访问权是安全的，因为只有在本地运行的应用程序可以使用令牌，并且要求的权限范围有限。 当你登录到这种类型的应用程序时，请注意你正在批准的是什么：他们可能会要求等多但是并不是你自己需要的许可授权（例如，他们可能会要求允许更改你的个人数据，而这些数据不太可能在你自己愿意的）。

### **What Just Happened?**
您刚刚以OAuth2的名义编写的应用程序是一个客户端应用程序，它使用授权code从Facebook（授权服务器）获取访问令牌(access_token)。 然后使用access_token向Facebook获取一些个人信息（仅限于您允许的内容），包括您的登录ID和您的姓名。 在这个阶段，facebook充当了一个资源服务器，对你发送的access_token进行解码，检查它给了应用程序访问用户详细信息的权限。 如果该过程成功，则应用程序将用户详细信息插入到Spring Security上下文中，以便进行身份验证。

如果您查看浏览器工具（Chrome上的F12），并锁定所有活跃的网络传输，您将看到Facebook的重定向，最后会有一个新的Set-Cookie header返回主页。 这个cookie（默认情况下是JSESSIONID）是Spring（或任何基于servlet的）应用程序的认证详情的标记。

所以我们有一个用户使用外部供应商（Facebook）进行认证相应内容安全的应用程序。 我们不希望将其应用于网上银行网站，仅是用于基本的身份识别目的，并将网站不同用户的内容隔离开来，这是一个很好的起点，这就解释了为什么这种认证现在非常流行。 在下一节中，我们将为应用程序添加一些基本功能，并且使用户对于最初重定向到Facebook时所发生的事情更加理解得清晰。

## **Add a Welcome Page**
在本节中，我们通过添加一个显式链接来修改我们刚刚构建的通过Facebook雅正的简单应用程序。 新链接不会立即被重定向，而是可以在主页上看到，用户可以选择登录或不登录。 只有当用户点击链接时，才会显示安全内容。

### **Conditional Content in Home Page**

要呈现一些内容，以用户是否通过验证为条件，我们可以使用服务器端呈现（例如，使用Freemarker或Tymeleaf），或者我们可以使用一些JavaScript渲染浏览器。 要做到这一点，我们选择使用AngularJS，但是如果你更喜欢使用不同的框架，更改客户端的代码不会很难。

要开始使用动态内容，我们需要在部分一些标签，以显示它：

**index.html**
```html
<div class="container unauthenticated">
    With Facebook: <a href="/login">click here</a>
</div>
<div class="container authenticated" style="display:none">
    Logged in as: <span id="user"></span>
</div>
```
这个HTML使我们用一些客户端代码来操纵已认证的、未经认证的和用户的元素的呈现情况。 下面是这些功能的简单实现（将它们放在<body>的末尾）：

**index.html**
```html
<script type="text/javascript">
    $.get("/user", function(data) {
        $("#user").html(data.userAuthentication.details.name);
        $(".unauthenticated").hide()
        $(".authenticated").show()
    });
</script>
```

### **Server Side Changes**
我们需要在服务器端进行一些更改。 “home”控制器需要一个描述当前认证用户的“/ user”端点。 这很容易做到，例如 在我们的main class中这样写：

**SocialApplication**
```java
@SpringBootApplication
@EnableOAuth2Sso
@RestController
public class SocialApplication {

  @RequestMapping("/user")
  public Principal user(Principal principal) {
    return principal;
  }

  public static void main(String[] args) {
    SpringApplication.run(SocialApplication.class, args);
  }

}
```

请注意使用@RestController和@RequestMapping以及我们注入处理程序方法的java.security.Principal。

>在这样的"/user"端点中返回一个完整的用户信息并不是一个好主意（它可能包含你不愿透露给浏览器客户端的信息）。 我们只是为了能快速完成这件事。 稍后在指南中，我们将转换端点以隐藏我们不需要暴露给浏览器的信息。

这个应用程序现在可以像以前一样正常工作，但不会让用户有机会点击我们刚刚提供的链接。 为了使链接可见，我们还需要通过添加一个WebSecurityConfigurer来关闭主页上的安全性配置：

**SocialApplication**
```java
@SpringBootApplication
@EnableOAuth2Sso
@RestController
public class SocialApplication extends WebSecurityConfigurerAdapter {

  ...

  @Override
  protected void configure(HttpSecurity http) throws Exception {
    http
      .antMatcher("/**")
      .authorizeRequests()
        .antMatchers("/", "/login**", "/webjars/**")
        .permitAll()
      .anyRequest()
        .authenticated();
  }

}
```

Spring Boot对使用@EnableOAuth2Sso注释的WebSecurityConfigurer实现类附加了一个特殊含义：使用它来配置携带OAuth2身份验证处理器的安全筛选器链。 所以我们所需要做的就是显式authorizeRequests()给主页和它包含的静态资源（还包括认证登录端点访问处理）的AuthorizeRequests（）。 所有其他请求（例如到/user端点）都需要认证。

更改到这里，应用程序仍是完整的，如果你运行它并访问主页，你应该看到一个风格很不错的HTML链接“登录到Facebook”。 链接不会直接传送到Facebook，而是传送到处理身份验证的本地路径（并将重定向发送到Facebook）。 一旦你通过身份验证，你会被重定向回当地的应用程序，并显示你的名字（假设你在Facebook上授予了你的数据权限）

## **Add a Logout Button**
在本节中，我们通过添加一个允许用户注销应用程序的按钮来修改我们构建的点击应用程序。 这看起来像一个简单的功能，但是需要花费点心思，所以值得花一些时间讨论如何去做。 大部分的改变是因为我们正在将应用程序从只读资源转换为读写资源（注销需要状态更改），所以在任何实际应用程序中都需要进行更改的不仅只是静态的内容。

### **Client Side Changes**
在客户端，我们只需要提供一个注销按钮和一些JavaScript来调用服务端，请求注销认证。 首先，在UI的“已认证”部分，我们添加按钮：

**index.html**
```html
<div class="container authenticated">
  Logged in as: <span id="user"></span>
  <div>
    <button onClick="logout()" class="btn btn-primary">Logout</button>
  </div>
</div>
```
然后我们提供它在JavaScript中引用的logout（）函数：

**index.html**
```html
var logout = function() {
    $.post("/logout", function() {
        $("#user").html('');
        $(".unauthenticated").show();
        $(".authenticated").hide();
    })
    return true;
}
```

logout（）函数向/logout端点发送一个POST请求 ，然后清除动态信息。 现在我们可以切换到服务器端来实现该端点。

### **Adding a Logout Endpoint**
Spring Security已经构建了一个支持/logout的端点，它将为我们做正确的事情（清除会话并使Cookie无效）。 要配置端点，我们只需在WebSecurityConfigurer中扩展现有的configure（）方法：

**SocialApplication.java**
```java
@Override
protected void configure(HttpSecurity http) throws Exception {
  http.antMatcher("/**")
    ... // existing code here
    .and().logout().logoutSuccessUrl("/").permitAll();
}
```

/logout端点需要我们发送POST请求给它，为了保护用户免受跨站点请求伪造（CSRF，发音为“sea surf”），它要求在请求中包含一个令牌。该令牌的值与当前session相关联，这是提高安全性的方式，所以我们需要一种方法将这些数据存入JavaScript应用程序。

许多JavaScript框架都支持CSRF（例如，在Angular中，他们称之为XSRF），但是它通常以与Spring Security的开箱即用方式稍有不同的方式实现。例如，在Angular中，前端希望服务器发送一个叫做“XSRF-TOKEN”的cookie，如果它看到的话，它会把这个值作为一个名为“X-XSRF-TOKEN”的header发回去。我们可以用我们简单的jQuery实现相同的行为，然后服务器端的变化将与其他前端实现一起工作，没有或很少有变化。为了向Spring Security讲授这个，我们需要添加一个创建cookie的过滤器，同时我们还需要告诉现有的CRSF过滤器header名称。在WebSecurityConfigurer中：

**SocialApplication.java**
```java
@Override
protected void configure(HttpSecurity http) throws Exception {
  http.antMatcher("/**")
    ... // existing code here
    .and().csrf().csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse());
}
```

**Adding the CSRF Token in the Client**
由于本示例中没有使用更高级别的框架，因此我们需要明确添加CSRF令牌,作为cookie来提供。 为了使代码更简单一些，我们添加了一个额外的库：

**pom.xml**
```xml
<dependency>
    <groupId>org.webjars</groupId>
    <artifactId>js-cookie</artifactId>
    <version>2.1.0</version>
</dependency>
```

添加进html中：
**index.html**
```html
<script type="text/javascript" src="/webjars/js-cookie/js.cookie.js"></script>

```

然后我们可以在xhr中使用Cookies方便的方法：

**index.html**
```html
$.ajaxSetup({
beforeSend : function(xhr, settings) {
  if (settings.type == 'POST' || settings.type == 'PUT'
      || settings.type == 'DELETE') {
    if (!(/^http:.*/.test(settings.url) || /^https:.*/
        .test(settings.url))) {
      // Only send the token to relative URLs i.e. locally.
      xhr.setRequestHeader("X-XSRF-TOKEN",
          Cookies.get('XSRF-TOKEN'));
    }
  }
}
});
```
### **Ready To Roll!**
随着这些变化，我们准备好运行应用程序，并尝试新的注销按钮。 启动应用程序并在新的浏览器窗口中加载主页。 点击“登录”链接将您带到Facebook（如果您已经登录，您可能不会注意到重定向）。 点击“注销”按钮取消当前会话，并将应用程序返回到未认证状态。 如果您好奇，您应该能够在浏览器与本地服务器交互的请求中看到新的cookie和header。

请记住，现在注销端点正在与浏览器客户端协同工作，那么所有其他的HTTP请求（POST，PUT，DELETE等）也同样适用。 所以这应该是一个有实用功能功能的好平台。

## **Manual Configuration of OAuth2 Client**
在本节中，我们通过选择@EnableOAuth2Sso注释中的“magic”来修改我们已经构建的注销应用程序，手动配置其中的任何内容以使其显式化。

### **Clients and Authentication**

@ EnableOAuth2Sso后面有两个功能：OAuth2客户端和身份验证。 客户端是可重用的，因此您也可以使用它来与您的授权服务器（本例中为Facebook）提供的OAuth2资源（本例中为Graph API）进行交互。 身份验证部分将您的应用程序与Spring Security的其余部分对接，所以一旦与Facebook交互结束，您的应用程序就会像其他任何安全的Spring应用程序一样运行。客户端部分由Spring Security OAuth2提供，并由不同的注释@EnableOAuth2Client开启。 所以这个转换的第一步就是删除@EnableOAuth2Sso并用下面的注解代替它：

**SocialApplication.java**
```java
@SpringBootApplication
@EnableOAuth2Client
@RestController
public class SocialApplication extends WebSecurityConfigurerAdapter {
  ...
}
```

一旦完成，我们便创造了一些有用的东西。 首先，我们可以注入一个OAuth2ClientContext并使用它来构建一个认证过滤器，并将其添加到我们的安全配置中：

**SocialApplication.java**
```java
@SpringBootApplication
@EnableOAuth2Client
@RestController
public class SocialApplication extends WebSecurityConfigurerAdapter {

  @Autowired
  OAuth2ClientContext oauth2ClientContext;

  @Override
  protected void configure(HttpSecurity http) throws Exception {
    http.antMatcher("/**")
      ...
      .and().addFilterBefore(ssoFilter(), BasicAuthenticationFilter.class);
  }

  ...

}
```
这个过滤器是在我们使用OAuth2ClientContext的新方法中创建的：
**SocialApplication.java**
```java
private Filter ssoFilter() {
  OAuth2ClientAuthenticationProcessingFilter facebookFilter = new OAuth2ClientAuthenticationProcessingFilter("/login/facebook");
  OAuth2RestTemplate facebookTemplate = new OAuth2RestTemplate(facebook(), oauth2ClientContext);
  facebookFilter.setRestTemplate(facebookTemplate);
  UserInfoTokenServices tokenServices = new UserInfoTokenServices(facebookResource().getUserInfoUri(), facebook().getClientId());
  tokenServices.setRestTemplate(facebookTemplate);
  facebookFilter.setTokenServices(tokenServices);
  return facebookFilter;
}
```

过滤器还需要了解有关Facebook的客户端注册：
**SocialApplication.java**
```java
  @Bean
  @ConfigurationProperties("facebook.client")
  public AuthorizationCodeResourceDetails facebook() {
    return new AuthorizationCodeResourceDetails();
  }
```
  为了完成认证，需要知道用户信息端点在Facebook的哪个位置：
  
  **SocialApplication.java**
```java
   @Bean
  @ConfigurationProperties("facebook.resource")
  public ResourceServerProperties facebookResource() {
    return new ResourceServerProperties();
  }
```
  请注意，对于这两个“静态”数据对象（Facebook（）和facebookResource（）），我们使用一个@Bean修饰@ConfigurationProperties。 这意味着我们可以将application.yml的配置格式改成一下，其中配置的前缀是facebook而不是security.oauth2：
**application.yml**
```yml
facebook:
  client:
    clientId: 233668646673605
    clientSecret: 33b17e044ee6a4fa383f46ec6e28ea1d
    accessTokenUri: https://graph.facebook.com/oauth/access_token
    userAuthorizationUri: https://www.facebook.com/dialog/oauth
    tokenName: oauth_token
    authenticationScheme: query
    clientAuthenticationScheme: form
  resource:
    userInfoUri: https://graph.facebook.com/me
```

最后，我们在上面的Filter声明中将登录路径更改为了facebook-specific，所以我们需要在HTML中进行相同的更改：

**index.html**
```html
<h1>Login</h1>
<div class="container unauthenticated">
	<div>
	With Facebook: <a href="/login/facebook">click here</a>
	</div>
</div>
```

### **Handling the Redirects**
我们需要做的最后一个变化是明确支持从我们的应用程序重定向到Facebook。 这是在Spring OAuth2中用Servlet Filter来处理的，并且过滤器已经在应用程序上下文中可用，因为我们使用了@ EnableOAuth2Client。 所有需要的是连接过滤器，以便在Spring Boot应用程序中以正确的顺序调用过滤器。 要做到这一点，我们需要一个FilterRegistrationBean：

**SocialApplication.java**
```java
@Bean
public FilterRegistrationBean oauth2ClientFilterRegistration(
    OAuth2ClientContextFilter filter) {
  FilterRegistrationBean registration = new FilterRegistrationBean();
  registration.setFilter(filter);
  registration.setOrder(-100);
  return registration;
}
```

我们自动装载可用的过滤器，并以足够低的顺序将其注册到主Spring Security过滤器之前。 通过这种方式，我们可以使用它来处理认证请求异常导致的重定向信号。

通过这些更改，应用程序可以很好地运行，并且在运行时等同于我们在上一节中建立的注销示例。 打破这个配置并明确地告诉我们，Spring Boot没有什么神奇的东西（它只是配置样板），它也准备了我们的应用程序来扩展的自动配置功能，按照我们自己的意见和业务要求添加。

## **login with github**
在本节中，我们修改已经创建的应用程序，添加一个链接，以便用户可以选择使用Github进行身份验证
### **Adding the Github Link**
在客户端，变化很小，我们只是添加另一个链接：

**index.html**
```html
<div class="container unauthenticated">
  <div>
    With Facebook: <a href="/login/facebook">click here</a>
  </div>
  <div>
    With Github: <a href="/login/github">click here</a>
  </div>
</div>
```

原则上，一旦我们开始添加身份验证提供程序，我们可能需要更加小心对待“/ user”端点返回的数据。 事实证明，Github和Facebook在他们的用户信息中都有一个“name”字段，所以实际上我们的简单终点没有任何变化。

### **Adding the Github Authentication Filter**
服务器上的主要变化是添加一个额外的安全过滤器来处理来自我们新链接的“/ login / github”请求。 我们已经在我们的ssoFilter（）方法中为Facebook创建了一个自定义的认证过滤器，所以我们只需要用一个可以处理多个认证路径的组合来代替它：

**SocialApplication.java**
```java
private Filter ssoFilter() {

  CompositeFilter filter = new CompositeFilter();
  List<Filter> filters = new ArrayList<>();

  OAuth2ClientAuthenticationProcessingFilter facebookFilter = new OAuth2ClientAuthenticationProcessingFilter("/login/facebook");
  OAuth2RestTemplate facebookTemplate = new OAuth2RestTemplate(facebook(), oauth2ClientContext);
  facebookFilter.setRestTemplate(facebookTemplate);
  UserInfoTokenServices tokenServices = new UserInfoTokenServices(facebookResource().getUserInfoUri(), facebook().getClientId());
  tokenServices.setRestTemplate(facebookTemplate);
  facebookFilter.setTokenServices(tokenServices);
  filters.add(facebookFilter);

  OAuth2ClientAuthenticationProcessingFilter githubFilter = new OAuth2ClientAuthenticationProcessingFilter("/login/github");
  OAuth2RestTemplate githubTemplate = new OAuth2RestTemplate(github(), oauth2ClientContext);
  githubFilter.setRestTemplate(githubTemplate);
  tokenServices = new UserInfoTokenServices(githubResource().getUserInfoUri(), github().getClientId());
  tokenServices.setRestTemplate(githubTemplate);
  githubFilter.setTokenServices(tokenServices);
  filters.add(githubFilter);

  filter.setFilters(filters);
  return filter;

}
```
其中旧的ssoFilter（）的代码已被复制过来，一个用于Facebook，一个用于Github，两个过滤器合并为一个复合。

请注意，facebook（）和facebookResource（）方法已经衍生了类似的方法，github（）和githubResource（）：

**SocialApplication.java**
```java

@Bean
@ConfigurationProperties("github.client")
public AuthorizationCodeResourceDetails github() {
	return new AuthorizationCodeResourceDetails();
}

@Bean
@ConfigurationProperties("github.resource")
public ResourceServerProperties githubResource() {
	return new ResourceServerProperties();
}
```

相应的配置：

**application.yml**
```yml
github:
  client:
    clientId: bd1c0a783ccdd1c9b9e4
    clientSecret: 1a9030fbca47a5b2c28e92f19050bb77824b5ad1
    accessTokenUri: https://github.com/login/oauth/access_token
    userAuthorizationUri: https://github.com/login/oauth/authorize
    clientAuthenticationScheme: form
  resource:
    userInfoUri: https://api.github.com/user
```
此处的客户端细节在于Github上的注册，地址为localhost：8080（与Facebook相同）。该应用程序现在准备运行，并为用户提供了与Facebook或Github进行身份验证的选择。

### **How to Add a Local User Database**
即使身份验证委托给外部提供者，许多应用程序也需要在本地保存有关的用户数据。 我们不在这里展示代码，但是分两步很容易做到。

1、为您的后端选择一个数据库，并为适合您的需求的自定义用户设置存储库（例如使用Spring Data），并且可以从外部认证中完全或部分地填充该对象。

2、通过检查/ user端点中的存储库，为每个登录的唯一用户配置一个User对象。 如果已有User对象具有当前用户的身份，则可以更新，否则创建。

提示：在User对象中添加一个字段，以链接到外部提供者中的唯一标识（不是用户名，而是外部提供者中帐户的唯一标识符）。

## **Hosting an Authorization Server**
在本节中，我们通过修改我们构建的github应用程序将其变成一个完整的OAuth2授权服务器，仍然使用Facebook和Github进行身份验证，但是能够创建自己的访问令牌（access_token）。 然后，这些令牌可用于保护后端资源，或与其他应用程序（我们需要以同样的方式保护）进行SSO。

### **整理认证配置(Tidying up the Authentication Configuration)**

在开始着手于授权服务器功能之前，我们将清理两个外部提供程序的配置代码。 有一些代码是在ssoFilter（）方法中重复的，所以我们把它们放到一个共享的方法中：
**SocialApplication.java**
```java
private Filter ssoFilter() {
  CompositeFilter filter = new CompositeFilter();
  List<Filter> filters = new ArrayList<>();
  filters.add(ssoFilter(facebook(), "/login/facebook"));
  filters.add(ssoFilter(github(), "/login/github"));
  filter.setFilters(filters);
  return filter;
}
```
新的方法具有旧方法的所有代码功能：
**SocialApplication.java**
```java
private Filter ssoFilter(ClientResources client, String path) {
  OAuth2ClientAuthenticationProcessingFilter filter = new OAuth2ClientAuthenticationProcessingFilter(path);
  OAuth2RestTemplate template = new OAuth2RestTemplate(client.getClient(), oauth2ClientContext);
  filter.setRestTemplate(template);
  UserInfoTokenServices tokenServices = new UserInfoTokenServices(
      client.getResource().getUserInfoUri(), client.getClient().getClientId());
  tokenServices.setRestTemplate(template);
  filter.setTokenServices(tokenServices);
  return filter;
}
```
它使用一个新的包装器对象ClientResources来合并在应用最后一个版本中单独声明且被@Beans修饰的OAuth2ProtectedResourceDetails和ResourceServerProperties：
**SocialApplication.java**
```java
class ClientResources {

  @NestedConfigurationProperty
  private AuthorizationCodeResourceDetails client = new AuthorizationCodeResourceDetails();

  @NestedConfigurationProperty
  private ResourceServerProperties resource = new ResourceServerProperties();

  public AuthorizationCodeResourceDetails getClient() {
    return client;
  }

  public ResourceServerProperties getResource() {
    return resource;
  }
}
```

>包装器使用@NestedConfigurationProperty指示注释处理器抓取元数据的类型，因为它不代表一个单一的值，而是一个完整的嵌套类型。

有了这个包装器，我们可以像以前一样使用相同的YAML配置，但是每个提供者只需要一个方法：
**SocialApplication.java**
```java
@Bean
@ConfigurationProperties("github")
public ClientResources github() {
  return new ClientResources();
}

@Bean
@ConfigurationProperties("facebook")
public ClientResources facebook() {
  return new ClientResources();
}
```

### **Enabling the Authorization Server**
如果我们想将我们的应用程序变成一个OAuth2授权服务器，那么至少要开始创造一些基本功能（一个客户端和创建访问令牌的能力）。 授权服务器只不过是一堆端点，它们在Spring OAuth2中以Spring MVC的处理形式呈现。 我们已经有一个安全的应用程序，所以这只是一个添加@EnableAuthorizationServer注释的问题：
**SocialApplication.java**
```java
@SpringBootApplication
@RestController
@EnableOAuth2Client
@EnableAuthorizationServer
public class SocialApplication extends WebSecurityConfigurerAdapter {

   ...

}
```
Spring Boot将安装所有必要的端点并为它们设置安全配置，前提是我们提供了一些我们想要支持的OAuth2客户端的细节：

**application.yml**
```yml
security:
  oauth2:
    client:
      client-id: acme
      client-secret: acmesecret
      scope: read,write
      auto-approve-scopes: '.*'
```

这个客户端相当于我们需要进行外部认证的facebook.client *和github.client *。 对于外部供应商，我们必须注册并获得客户端ID和安全信息，以在我们的应用程序中使用。 在这种情况下，我们提供了相同的功能，所以我们需要（至少一个）客户端来工作。

我们已经将auto-approve-scopes设置为匹配所有范围的正则表达式。 我们将这个应用放在一个单独的服务中不一定有必要，为了可以让我们快速地开展工作，而无需重新发布Spring OAuth2在我们的用户需要访问令牌时弹出的白标签审批页面。 要为令牌授予添加一个明确的批准步骤，我们需要提供一个替换白标签版本的界面（at / oauth / confirm_access）。
>**原文**
>We have set the auto-approve-scopes to a regex matching all scopes. This is not necessarily where we would leave this app in a real system, but it gets us something working quickly without having toreplace the whitelabel approval page that Spring OAuth2 would otherwise pop up for our users when they wanted an access token. To add an explicit approval step to the token grant we would need to provide a UI replacing the whitelabel version (at /oauth/confirm_access).

为了完成授权服务器，我们需要为其UI提供安全配置。 事实上，在这个简单的应用程序中没有太多的用户界面，但是我们仍然需要保护/ oauth / authorize端点，并确保主页上的“登录”按钮是可见的。 这就是为什么我们有这个方法：
```java
@Override
protected void configure(HttpSecurity http) throws Exception {
  http.antMatcher("/**")                                       (1)
    .authorizeRequests()
      .antMatchers("/", "/login**", "/webjars/**").permitAll() (2)
      .anyRequest().authenticated()                            (3)
    .and().exceptionHandling()
      .authenticationEntryPoint(new LoginUrlAuthenticationEntryPoint("/")) (4)
    ...
}
```
1所有的请求都被默认保护
2明确排除主页和登录端点
3所有其他端点需要经过身份验证的用户
4未经身份验证的用户将被重定向到主页

### **How to Get an Access Token**
现在可以从我们新的授权服务器获得令牌。 获取令牌的最简单的方法是抓取一个“acme”客户端。 你可以看到这个，如果你运行应用程序并curl它：

```curl
$ curl acme:acmesecret@localhost:8080/oauth/token -d grant_type=client_credentials
{"access_token":"370592fd-b9f8-452d-816a-4fd5c6b4b8a6","token_type":"bearer","expires_in":43199,"scope":"read write"}
```

客户端证书令牌在某些情况下很有用（例如测试令牌端点的工作情况），但为了充分利用我们服务器的所有功能，我们希望能够为用户创建令牌。我们需要能够验证用户是不是我们应用的一个有效身份，因此需要能校验令牌。 如果您在应用程序启动时仔细观看日志，则会看到为默认的Spring Boot用户（根据Spring Boot用户指南）记录了一个随机密码。 您可以使用此密码获取一个代表“user”的用户令牌：

```curl
$ curl acme:acmesecret@localhost:8080/oauth/token -d grant_type=password -d username=user -d password=...
{"access_token":"aa49e025-c4fe-4892-86af-15af2e6b72a2","token_type":"bearer","refresh_token":"97a9f978-7aad-4af7-9329-78ff2ce9962d","expires_in":43199,"scope":"read write"}
```

其中“...”应该用实际的密码替换。 这被称为“密码”授权，您可以在其中交换访问令牌的用户名和密码。密码授权也主要用于测试，也可以适用于本地或移动应用程序，当您有本地用户数据库来存储和验证凭据时。 对于大多数应用程序或任何具有“社交”登录功能的应用程序（如我们的应用程序），您需要“授权码”授权，这意味着您需要浏览器（或像浏览器那样的客户端）来处理重定向和Cookie， 调用来自外部提供者的用户界面。

### **Creating a Client Application**

**ClientApplication.java**
```java
@EnableAutoConfiguration
@Configuration
@EnableOAuth2Sso
@RestController
public class ClientApplication {

  @RequestMapping("/")
  public String home(Principal user) {
    return "Hello " + user.getName();
  }

  public static void main(String[] args) {
    new SpringApplicationBuilder(ClientApplication.class)
        .properties("spring.config.name=client").run(args);
  }

}
```

ClientApplication类不能在SocialApplication类的相同包路劲（或子包）中创建。 否则，当启动SocialApplication服务器时，Spring会加载一些ClientApplication自动配置，导致启动错误。

客户端的组件是一个主页（只是输出用户的名字）和一个有明确应用名称的配置文件（通过spring.config.name = client）。 当我们运行这个应用程序时，它会查找我们提供的配置文件，如下所示：

**client.yml**
```yml
server:
  port: 9999
  context-path: /client
security:
  oauth2:
    client:
      client-id: acme
      client-secret: acmesecret
      access-token-uri: http://localhost:8080/oauth/token
      user-authorization-uri: http://localhost:8080/oauth/authorize
    resource:
      user-info-uri: http://localhost:8080/me
```

配置看起来很像我们在之前的应用程序中使用的配置，但是使用“acme”客户端而不是Facebook或Github。 该应用程序将运行在9999端口，以避免与主应用程序的冲突。 它指的是我们还没有实现的用户信息端点“/me”。

请注意，server.context-path是显式设置的，所以如果你运行应用程序来测试它，记得主页是http：// localhost：9999 / client。 点击该链接跳转到auth认证服务器，一旦你已经与你选择的社交提供者进行身份验证，你将被重定向到客户端应用程序

>如果您在本地主机上同时运行客户端和auth服务器，则上下文路径必须是明确的，否则cookie路径冲突和将导致两个应用程序无法会话达成一致。

### **Protecting the User Info Endpoint**
要使用我们的新授权服务器进行单点登录，就像我们已经使用Facebook和Github一样，它需要有一个/user端点受到其创建的访问令牌的保护。 到目前为止，我们有一个/user端点，并且用用户认证时创建的cookie进行安全保护。 除了本地授予的访问令牌之外，为了保护它，我们可以重新使用现有的端点并在新的路径上创建一个别名：
**SocialApplication.java**
```java
@RequestMapping({ "/user", "/me" })
public Map<String, String> user(Principal principal) {
  Map<String, String> map = new LinkedHashMap<>();
  map.put("name", principal.getName());
  return map;
}
```

>我们已经将Principal转换成Map来隐藏我们不想暴露给浏览器的部分，并且也解除了两个外部认证提供者之间端点的行为。 原则上，我们可以在这里添加更多的细节，例如特定于提供者的唯一标识符，或者可用的电子邮件地址。

通过声明我们的应用程序是资源服务器（以及授权服务器），现在可以通过访问令牌来保护“/ me”路径。 我们创建一个新的配置类（在主应用程序中作为n内部类，但也可以拆分成单独的独立类）：

**SocialApplication.java**
```java
@Configuration
@EnableResourceServer
protected static class ResourceServerConfiguration
    extends ResourceServerConfigurerAdapter {
  @Override
  public void configure(HttpSecurity http) throws Exception {
    http
      .antMatcher("/me")
      .authorizeRequests().anyRequest().authenticated();
  }
}
```

另外，我们需要为主应用程序安全性指定@Order：
**SocialApplication.java**
```java
@SpringBootApplication
...
@Order(SecurityProperties.ACCESS_OVERRIDE_ORDER)
public class SocialApplication extends WebSecurityConfigurerAdapter {
  ...
}
```

@EnableResourceServer注解默认创建了一个安全过滤器，默认情况下是@Order（SecurityProperties.ACCESS_OVERRIDE_ORDER-1），所以通过将主应用程序安全性改变为@Order（SecurityProperties.ACCESS_OVERRIDE_ORDER）可以确保优先使用“/ me”规则。

### **Testing the OAuth2 Client**
要测试新功能，您只需运行这两个应用程序，然后在浏览器中访问http：// localhost：9999 / client。 客户端应用程序将重定向到本地授权服务器，然后用户选择通过Facebook或Github进行身份验证。 一旦完成控制权返回到测试客户端，本地访问令牌被授予，即验证完成（您应该在浏览器中看到“Hello”消息）。 如果您已经通过Github或Facebook进行身份验证，您甚至可能不会注意到远程身份验证。

## **Adding an Error Page for Unauthenticated Users**
在本节中，我们修改了我们之前构建的注销应用程序，切换到Github身份验证，并向无法进行身份验证的用户提供一些反馈。 同时，我们借此机会扩展身份验证逻辑，仅在用户属于特定Github组织时才允许用户的规则。 “组织”是Github领域特定的概念，但是可以为其他提供者设计类似的规则。 对于谷歌，你可能只想认证来自特定域的用户。

### **Switching to Github**
注销示例使用Facebook作为OAuth2提供者。 通过更改本地配置，我们可以轻松切换到Github：

**application.yml**
```yml
security:
  oauth2:
    client:
      clientId: bd1c0a783ccdd1c9b9e4
      clientSecret: 1a9030fbca47a5b2c28e92f19050bb77824b5ad1
      accessTokenUri: https://github.com/login/oauth/access_token
      userAuthorizationUri: https://github.com/login/oauth/authorize
      clientAuthenticationScheme: form
    resource:
      userInfoUri: https://api.github.com/user
```

### **Detecting an Authentication Failure in the Client**
在客户端上，我们需要能够为无法认证的用户提供一些反馈。 为了促进这一点，我们添加一个提示信息：

**index.html**
```html
<div class="container text-danger error" style="display:none">
There was an error (bad credentials).
</div>
```

这个文本只有在显示“error”元素时才会显示，所以我们需要一些代码来支撑这一点：

**index.html**
```html
$.ajax({
  url : "/user",
  success : function(data) {
    $(".unauthenticated").hide();
    $("#user").html(data.userAuthentication.details.name);
    $(".authenticated").show();
  },
  error : function(data) {
    $("#user").html('');
    $(".unauthenticated").show();
    $(".authenticated").hide();
    if (location.href.indexOf("error=true")>=0) {
      $(".error").show();
    }
  }
});
```

The authentication function checks the browser location when it loads and if it finds a URL with "error=true" in it, the flag is set.

### **Adding an Error Page**
为了支持客户端中的标志设置，我们需要能够捕获认证错误，并使用在查询参数中设置的标志重定向到主页。 因此我们需要一个端点，像这样在一个常规的@Controller中编写代码：

**SocialApplication.java**
```java
@RequestMapping("/unauthenticated")
public String unauthenticated() {
  return "redirect:/?error=true";
}
```

在示例应用程序中，我们把它放在主应用程序类，它现在是@Controller（而不是@RestController），所以它可以处理重定向。 我们需要的最后一件事是配置未经验证的响应（HTTP 401，a.k.a.UNAUTHORIZED）到我们刚添加的“/ unauthenticated”端点的映射：
**ServletCustomizer.java**
```java
@Configuration
public class ServletCustomizer {
  @Bean
  public EmbeddedServletContainerCustomizer customizer() {
    return container -> {
      container.addErrorPages(new ErrorPage(HttpStatus.UNAUTHORIZED, "/unauthenticated"));
    };
  }
}
```

（在示例中，这是为了简洁起见，在主应用程序中添加为嵌套类。）

### **Generating a 401 in the Server**
如果用户不能或不想用Github登录，那么Spring Security就已经有了401响应，所以如果你没有通过认证（例如通过拒绝令牌授权），应用程序已经在运行。为了使事情变得有点夸张，我们将扩展身份验证规则以拒绝不在指定组织中的用户。 使用Github API很容易找到更多关于用户的信息，所以我们只需要将其插入到认证过程的正确部分即可。 幸运的是，对于这样一个简单的用例，Spring Boot提供了一个简单的扩展点：如果我们声明一个AuthoritiesExtractor类型的@Bean，它将被用来构造经过身份验证的用户的权限（通常是“角色”）。 我们可以使用这个钩子来声明用户的权限是正确的，如果不是，则抛出一个异常：

**SocialApplication.java**
```java
@Bean
public AuthoritiesExtractor authoritiesExtractor(OAuth2RestOperations template) {
  return map -> {
    String url = (String) map.get("organizations_url");
    @SuppressWarnings("unchecked")
    List<Map<String, Object>> orgs = template.getForObject(url, List.class);
    if (orgs.stream()
        .anyMatch(org -> "spring-projects".equals(org.get("login")))) {
      return AuthorityUtils.commaSeparatedStringToAuthorityList("ROLE_USER");
    }
    throw new BadCredentialsException("Not in Spring Projects origanization");
  };
}
```

请注意，我们已经将OAuth2RestOperations自动装入此方法，因此我们可以使用它代表经过身份验证的用户来访问Github API。 我们这样做，并循环查找组织，寻找与“spring-projects”（这是用于存储Spring开源项目的组织）相匹配的组织。 如果您希望能够成功进行身份验证，并且您不在Spring工程团队中，则可以在此替换自己的值。 如果没有匹配的话，我们抛出BadCredentialsException，这被Spring Security捕获，并且作出401响应。

OAuth2RestOperations也必须被创建为一个bean（就像Spring Boot 1.4一样），但是没很么用，因为它的所有成分都可以通过使用@EnableOAuth2Sso来自动修改：

```java
@Bean
public OAuth2RestTemplate oauth2RestTemplate(OAuth2ProtectedResourceDetails resource, OAuth2ClientContext context) {
	return new OAuth2RestTemplate(resource, context);
}
```

>显然上面的代码可以推广到其他认证规则，有些适用于Github，有些适用于其他OAuth2提供者。 所有你需要的是OAuth2RestOperations和提供者的API的一些知识。

## **Conclusion**
我们已经看到如何使用Spring Boot和Spring Security以很少的代码来构建多种样式的应用程序。 贯穿所有示例的主题是使用外部OAuth2提供者的“社交”登录。 最终的样本甚至可以用来“内部”提供这样的服务，因为它具有与外部提供者相同的基本特征。 所有示例应用程序都可以很容易地扩展和重新配置，以用于更具体的用例，通常只用配置文件更改。 请记住，如果您使用自己的服务器中的示例版本来注册Facebook或Github（或类似），并获得您自己的主机地址的客户端凭据。 请记住不要将这些凭据放在开源代码中！
