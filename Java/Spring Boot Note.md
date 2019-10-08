SpringBoot配置项笔记

spring.profiles.active=*，表示当前激活配置文件，默认的格式是application-[*].properties/yml

###外部配置（Externalized Configuration）

Spring Boot允许扩展配置，所以可以在不同的环境上运行同一套代码。配置支持properties/yml格式，也可以用环境变量和命令行参数来扩展配置。属性值可以直接通过@Value注入到bean中，通过Environment抽象访问或@ConfigurationProperties来绑定到结构对象。

Spring Boot使用特定的PropertySource顺序，该顺序旨在允许合理地覆盖值。 按以下顺序考虑属性：

1. 当Devtools激活时，根目录的Devtools global settings properties。
1. @TestPropertySource的测试注解。
1. @SpringBootTest#properties的测试注解属性
1. 命令行参数。
1. 来自SPRING_APPLICATION_JSON的属性（嵌入到环境变量或系统属性的内联json）
1. ServletConfig的初始化参数
1. ServletContext的初始化参数
1. 来源于java:comp/env的JNDI属性。
1. Java系统属性（System.getProperties())
1. 操作系统的环境变量。
1. RandomValuePropertySource，仅在random.*具有属性。
1. 在jar包外部的指定文件的应用属性。（application-{profile}.properties/yml）
1. 在jar包内部的指定文件的应用属性。（application-{profile}.properties/yml）
1. 在jar包外部的应用属性（application.properties/yml)
1. 在jar包内部的指定文件的应用属性。（application.properties/yml)
1. 在@Configuration类下的@PropertySource注解。
1. 默认属性（通过SpringApplication.setDefaultProperties指定）


###配置随机值（Configuring random values）

RandomValuePropertySource用于注入随机值，可以产生Integer,Long,uuid或string。

在int中可以设置随机区间，语法为: OPEN value [,max] CLOSE，其中OPEN和CLOSE为任意字符，value,max为int，指定区间（排除最大值）

###获取命令行属性（Accessing command line properties）

默认情况下，SpringApplication会将任意的命令行选项参数转化为属性并添加到Spring Environment中。如上所述，命令行属性始终优先于其他属性源。


### 应用属性文件（Application property files）

SpringApplication会从几个路径对application.properties/yml文件加载属性并将其添加到Spring Environment。

1. 当前目录下的/config子目录
1. 当前目录。
1. 类路径下的/config包。
1. 类路径的根目录。

该列表依照优先级排序（高优先级会覆盖低优先级的属性）。

环境属性通过spring.config.name来指定配置文件名称，通过spring.config.location来指定配置文件位置。二者必须定义为环境属性（系统环境变量，系统属性或命令行参数）

示例：

	$ java -jar myproject.jar --spring.config.name=myproject
	$ java -jar myproject.jar --spring.config.location=classpath:/default.properties,classpath:/override.properties

若spring.config.location中包含有路径，应该以/结尾（会与spring.config.name的值组合，也包括指定的profile名称）。spring.config.location中指定的文件按原样使用，不支持特定于配置文件的变体，并且将被任何特定于配置文件的属性覆盖。

配置路径基于反向顺序搜索，默认的路径是classpath:/, classpath:/config/,file:./,file:./config/。自定义配置时，会添加到默认路径下，会在默认路径前进行搜索：

1. file:./custom-config/
1. classpath:custom-config/
1. file:./config/
1. file:./
1. classpath:/config/
1. classpath:/


### 特定文件的属性（Profile-specific properties）

