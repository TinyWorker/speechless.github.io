## Spring Security OAuth2的自动配置

### 授权服务器

要创建授权服务并授权访问令牌，你需要使用@EnableAuthorizationServer注解，以及提供以下的属性

	security.oauth2.client.client-id
	security.oauth2.client.client-secret

客户端会被注册到内存仓库，此时可以使用客户端证书来创建一个访问令牌，示例如下：

	curl client:secret@localhost:8080/oauth/token -d grant_type=password -d username=user -d password=pwd

/token节点的基本凭证是client-id和client-secret，用户凭证是常规的SpringSecurity用户明细（SpringBoot中默认是user和一个随机密码）

若要关闭自动配置，仅需在AuthorizationServerConfigurer类型上添加@Bean注解即可。

如果使用自定义的授权服务配置来配置合法客户端，使用ClientDetailsServiceConfigurer实例，以下的示例中，配置的密码是取决于Spring Security 5的密码存储，意思是当你使用Spring Boot Security默认密码存储，必须用id作为密码前缀。

	@Component
	public class CustomAuthorizationServerConfigurer extends
	    AuthorizationServerConfigurerAdapter {
	
	    AuthenticationManager authenticationManager;
	
	    public CustomAuthorizationServerConfigurer(
	        AuthenticationConfiguration authenticationConfiguration
	    ) throws Exception {
	        this.authenticationManager =
	            authenticationConfiguration.getAuthenticationManager();
	    }
	
	    @Override
	    public void configure(
	        ClientDetailsServiceConfigurer clients
	    ) throws Exception {
	        clients.inMemory()
	            .withClient("client")
	                .authorizedGrantTypes("password")
	                .secret("{noop}secret")
	                .scopes("all");
	    }
	
	    @Override
	    public void configure(
	        AuthorizationServerEndpointsConfigurer endpoints
	    ) throws Exception {
	        endpoints.authenticationManager(authenticationManager);
	    }
	}


### 资源服务

要使用访问令牌就要有资源服务（可以与授权服务是同一个服务器）。创建资源服务进需添加@EnableResourceServer注解并提供配置项允许服务可以解码访问令牌。

如果应用已经是一个授权服务，且知道如何解码令牌，那就不需要做额外的事情。如果应用是一个独立服务，那就要提供配置，有如下选项：

- security.oauth2.resource.user-info-uri，使用/me的资源（比如PWS,https://uaa.run.pivotal.io/userinfo）
 
- security.oauth2.resource.token-info-uri，使用令牌解码端点（比如PWS，https://uaa.run.pivotal.io/check_token）

如果同时指定了两个方式，可以设置优先级，默认是prefer-token-info。

若令牌是JWT，则可以配置security.oauth2.resoure.jwt.key-value来本地解码。其中的key是验证key，是一个对称加密密码或PEM编码的RSA公钥。

如果key是公开的但你没有key，可以配置security.oauth2.resouce.jwt.key-uri，提供URI来下载key。以PWS为例：

	$ curl https://uaa.run.pivotal.io/token_key
	{"alg":"SHA256withRSA","value":"-----BEGIN PUBLIC KEY-----\nMIIBI...\n-----END PUBLIC KEY-----\n"}

另外，若授权服务有端点可以返回JWKs集合，可以配置security.oauth2.resource.jwk.key-set-uri，以PWS为例

	$ curl https://uaa.run.pivotal.io/token_keys
	{"keys":[{"kid":"key-1","alg":"RS256","value":"-----BEGIN PUBLIC KEY-----\nMIIBI...\n-----END PUBLIC KEY-----\n"]}

要注意的是，同时配置JWT和JWK会导致错误，仅有一个配置项需要配置。另外，若使用了JWT或JWK，授权服务需要在你的应用启动前运行，如果没有找到key它会记录警告，并提示修正的方法。

OAuth2资源是收到过滤链保护的，顺序为security.oauth2.resource.filter-order，默认在默认保护执行器端点的过滤器之后。


#### 用户信息中的令牌类型

谷歌及其他三方身份提供商，对头部发送到用户信息端点的令牌类型命名更加严格。 默认是Bearer，适合大多数提供商和匹配规范，但若要更改，可以设置security.oauth2.resource.token-type。

#### 自定义用户信息模板

如果定义了user-info-uri，资源服务内部使用了OAuth2RestTemplate来获取用户信息。在类型UserInfoRestTemplateFactory上添加@Bean注解。默认情况应该适用于大部分的提供商，但偶尔需要添加拦截器，或者更改请求授权器（令牌如何附加到传出请求中）。


### 客户端

将web应用转变为一个OAuth2客户端只需要添加@EnableOAuth2Client注解，Spring Boot将创建一个OAuth2ClientContext和OAuth2ProtectedResourceDetails，这是创建OAuth2RestOperations的必须对象。 Spring Boot不会自动创建这些对象，但可以简单的自己创建：

	@Bean
	public OAuth2RestTemplate oauth2RestTemplate(OAuth2ClientContext oauth2ClientContext,
	        OAuth2ProtectedResourceDetails details) {
	    return new OAuth2RestTemplate(details, oauth2ClientContext);
	}

下面的配置使用security.oauth2.client.*作为凭证，但也需要知道授权和令牌的URI，示例如下：

	security:
	  oauth2:
	    client:
	      clientId: bd1c0a783ccdd1c9b9e4
	      clientSecret: 1a9030fbca47a5b2c28e92f19050bb77824b5ad1
	      accessTokenUri: https://github.com/login/oauth/access_token
	      userAuthorizationUri: https://github.com/login/oauth/authorize
	      clientAuthenticationScheme: form

这个配置的应用会重定向到GitHub进行授权，若已经登录了Github可以不会注意到有验证的发生。这些指定凭证仅在你的应用运行在端口8080上。

### 单点登录

OAuth2客户端可从提供商出获取用户明细，并将其转换成验证令牌。资源服务通过user-info-uri属性来支持这一行为，这是SSO协议基于OAuth2的基础。Spring Boot通过注解@EnableOAuth2Sso参与。

比如Gihub客户端可以保护自己的资源和使用Github /user/端点进行验证。

	security:
	  oauth2:
	# ...
	  resource:
	    userInfoUri: https://api.github.com/user
	    preferTokenInfo: false

因为默认情况下所有路径是安全的，所以没有主页来显示给未验证的用户并邀请他们登录（访问/login，或配置指定的声明 security.oauth2.sso.login-path）

要自定义访问规则，比如可以添加主页，可以在WebSecurityConfigurerAdapter上添加@EnableOAuth2Sso，这样能使其被必要组件装饰并增强，能让/login路径工作。

	@Configuration
	public class WebSecurityConfiguration extends WebSecurityConfigurerAdapter {
	
	    @Override
	    protected void configure(HttpSecurity http) throws Exception {
	        http
	            .authorizeRequests()
	                .mvcMatchers("/").permitAll()
	                .anyRequest().authenticated();
	    }
	}

由于所有端点默认是安全的，同样也包括错误处理端点。若在sso过程中发生错误，要求应用重定向到错误页面，这可能导致一个无限重定向。

首先，仔细考虑使端点不安全，因为您可能会发现行为只是不同问题的证据。 但是，通过将应用程序配置为允许“/ error”可以解决此问题：

	@Configuration
	public class WebSecurityConfiguration extends WebSecurityConfigurerAdapter {
	
	    @Override
	    protected void configure(HttpSecurity http) throws Exception {
	        http
	            .authorizeRequests()
	                .antMatchers("/error").permitAll()
	                .anyRequest().authenticated();
	    }
	}


### 附录：常见应用属性

	# SECURITY OAUTH2 CLIENT (OAuth2ClientProperties)
	security.oauth2.client.client-id= # OAuth2 client id.
	security.oauth2.client.client-secret= # OAuth2 client secret. A random secret is generated by default
	
	# SECURITY OAUTH2 RESOURCES (ResourceServerProperties)
	security.oauth2.resource.id= # Identifier of the resource.
	security.oauth2.resource.jwt.key-uri= # The URI of the JWT token. Can be set if the value is not available and the key is public.
	security.oauth2.resource.jwt.key-value= # The verification key of the JWT token. Can either be a symmetric secret or PEM-encoded RSA public key.
	security.oauth2.resource.jwk.key-set-uri= # The URI for getting the set of keys that can be used to validate the token.
	security.oauth2.resource.prefer-token-info=true # Use the token info, can be set to false to use the user info.
	security.oauth2.resource.service-id=resource #
	security.oauth2.resource.token-info-uri= # URI of the token decoding endpoint.
	security.oauth2.resource.token-type= # The token type to send when using the userInfoUri.
	security.oauth2.resource.user-info-uri= # URI of the user endpoint.
	
	# SECURITY OAUTH2 SSO (OAuth2SsoProperties)
	security.oauth2.sso.login-path=/login # Path to the login page, i.e. the one that triggers the redirect to the OAuth2 Authorization Server