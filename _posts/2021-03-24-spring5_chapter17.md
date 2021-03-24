---
layout: post
title: "Spring5 : 프로필과 프로퍼티 파일"
subtitle: "프로필, 프로퍼티"
author: "GuGyu"
header-style: text
tags:
  - Spring
  - Spring5
  - Java
  - Study
  - Server
  - Profile
  - Property
---

## 1. 프로필  

개발 단계에서는 실제 서비스 목적으로 운영중인 DB를 이용할 수는 없다. 테스트 서버를 따로 사용하거나 개인 PC에 DB를 설치하는 방법 등을 선택해야한다. 또한 실제 서비스 환경에서는 웹 서버와 DB 서버가 서로 다른 장비에 설치된 경우가 많다. 개발 환경에서 사용한 DB 계정과 실 서비스 환경에서 사용할 DB 계정이 다른 경우도 흔하다. 즉 개발을 완료한 어플리케이션을 실제 서버에 배포하려면 실 서비스 환경에 맞는 JDBC 연결 정보를 사용해야 한다.  

배포 전에 설정 정보를 직접 변경하는 것은 실수하기가 쉽다. 따라서 처음부터 개발 목적의 설정과 실 서비스 목적읠 설정을 구분해서 작성해두는 방법을 많이 사용하는데, 이를 위한 스프링 기능이 **프로필(profile)** 이다.  

프로필은 논리적인 이름으로서 설정 집합에 프로필을 지정할 수 있다. 스프링 컨테이너는 설정 집합 중에서 지정한 이름을 사용하는 프로필을 선택하고 해당 프로필에 속한 설정을 이용해서 컨테이너를 초기화할 수 있다. 

<img src="https://user-images.githubusercontent.com/57310034/112262025-4b46dc80-8cb0-11eb-9b07-c27736f17102.png"/>  

예를 들어 로컬 개발 환경을 위한 `DataSource` 설정을 "dev" 프로필로 지정하고 실 서비스 환경을 위한 `DataSource` 설정을 "real" 프로필로 지정한 뒤, "dev" 프로필을 사용해서 스프링 컨테이너를 초기화할 수 있다. 그러면 스프링은 "dev" 프로필에 정의된 빈을 사용하게 된다.  

### 1-1. 프로필 사용  

#### 1-1-1. @Configuration

`@Configuration` 어노테이션을 이용한 설정에서 프로필을 지정하려면 `@Profile` 어노테이션을 이용한다.  

```java
@Configuration
@Profile("dev")
public class DevelopConfig {

    @Bean(destroyMethod = "close")
    public DataSource dataSource() {
        DataSource datasource = new DataSource();
        dataSource.setDriverClassName("com.mysql.jdbc.Driver");
        dataSource.setUrl("jdbc:mysql://localhost/spring?characterEncoding=ut8");
        ...
        return dataSource;
    }
}
```  

스프링 컨테이너를 초기화할 때 "dev" 프로필을 활성화하면 위 클래스(`DevelopConfig`)를 설정으로 사용한다.  

"dev"가 아닌 "real" 프로필을 활성화했을 때 사용할 설정 클래스는 다음과 같이 `@Profile` 어노테이션의 값으로 "real"을 지정한다.  

```java
@Configuration
@Profile("dev")
public class ProductConfig {

    @Bean(destroyMethod = "close")
    public DataSource dataSource() {
        DataSource datasource = new DataSource();
        dataSource.setDriverClassName("com.mysql.jdbc.Driver");
        //주소 변경
        dataSource.setUrl("jdbc:mysql://realdb/spring?characterEncoding=ut8");
        ...
        return dataSource;
    }
}
```  

`DevleopConfig` 클래스와 `ProductConfig` 클래스는 둘 다 이름이 "dataSource"인 `DataSource` 타입의 빈을 설정하고 있다. 두 "dataSource" 빈 중에서 어떤 빈을 사용할지는 활성화한 프로필에 따라 달라진다.  

특정 프로필을 선택하려면 컨테이너를 초기화하기 전에 `setActiveProfiles()` 메서드를 사용해서 프로필을 선택해야 한다.  

```java
AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext();
context.getEnvironment().setActiveProfiles("dev");
context.register(MemberConfig.class, DevelopConfig.class, ProductConfig.class);
context.refresh();
```

`getEnvironment()` 메서드는 스프링 실행 환경을 설정하는데 사용되는 `Environment`를 리턴한다. 이 `Environment`의 `setActiveProfiles()` 메서드를 사용해서 사용할 프로필을 선택할 수 있다.  

프로필을 사용할 때 주의할 점은 설정 정보를 전달하기 전에 프로필을 설정해야한다는 점이다. 위 코드 처럼 `setActiveProfile()` -> `register()` -> `refresh()` 순서를 지켜주지 않으면 설정을 읽어오는 과정에서 빈을 찾지 못해 예외가 발생할 수 있다.  

두 개 이상의 프로필을 활성화하고 싶다면 다음과 같이 각 프로필 이름을 메서드에 파라미터로 전달한다.  

`context.getEnvironment().setActiveProfile("dev", "mysql");`

#### 1-1-2. 시스템 프로퍼티  

프로필을 선택하는 또 다른 방법은 spring.profiles.active 시스템 프로퍼티에 사용할 프로필 값을 지정하는 것이다. 두 개 이상인 경우 사용할 프로필을 콤마로 구분해서 설정하면 된다. 시스템 프로퍼티는 명령행에서 -D 옵션을 이용하거나 `System.setProperty()`를 이용해서 지정할 수 있다.  

`java -D spring.profiles.active=dev main.Main`  

자바의 시스템 프로퍼티뿐만 아니라 OS의 "spring.profiles.active" 환경 변수에 값을 설정해도 된다. 우선 순위는 다음과 같다.  

- `setActiveProfile()`
- 자바 시스템 프로퍼티
- OS 환경 변수  

#### 1-1-3. 다수 프로필 설정  

스프링 설정은 두 개 이상의 프로필 이름을 가질 수 있다.  

```java
@Configuration
@Profile("real,test")
public class DataSourceConfig {
    ...
}
```

혹은 특정 프로필 이름을 제외한 설정도 가능하다.  

```java
@Configuration
@Profile("!real")
public class DevelopConfig {
    ...
}
```  

#### 1-1-4. 어플리케이션에서 프로필 설정하기  

웹 어플리케이션의 경우에도 spring.profiles.active 시스템 프로퍼티나 환경 변수를 사용해서 사용할 프로필을 선택할 수 있다. 그리고 web.xml에서 다음과 같이 sping.profiles.active 초기화 파라미터를 이용해서 프로필을 선택할 수 있다.  

```xml
<servelt>
    <servlet-name>dispatcher</servlet-name>
    <servlet-class>
        org.springframework.web.servlet.DispatcherServlet
    </servlet-class>
    <init-param>
        <param-name>spring.profiles.active</param-name>
        <param-value>dev</param-value>
    </init-param>
    ...
</servlet>
```  

<br>

## 2. 프로퍼티  

스프링은 외부의 프로퍼티 파일을 이용해서 스프링 빈을 설정하는 방법을 제공하고 있다. 예를 들어 다음과 같은 db.properties 파일이 있다고 하자.  

```properties
db.driver=com.mysql.jdbc.Driver
db.url=jdbc:mysql://localhost/spring?characterEncoding=utf8
db.user=spring5
db.password=spring5
```  

이 파일의 프로퍼티 값을 자바 설정에서 사용할 수 있으며, 이를 통해 설정 일부를 외부 프로퍼티 파일을 사용해서 변경할 수 있다.  

### 2-1. @Configuration 설정에서 사용  

자바 설정에서 프로퍼티 파일을 사용하려면 다음 두 가지를 설정한다.  

- `PropertySourcePlaceholderConfigurer` 빈 설정
- `@Value` 어노테이션으로 프로퍼티 값 사용  

```java
@Configuration
public class PropertyConfig {
    @Bean
    public static PropertySourcesPlaceholderConfigurer properties() {
        PropertySourcePlaceholderConfigurer configurer = 
            new PropertySourcesPlaceholderConfigurer();
        configurer.setLocation(
            new ClassPathResource("db.properties"),
            new ClassPathResource("info.properties")
        )
        return configurer;
    }
}
```  

`PropertySourcePlaceholderConfigurer#setLocation()` 메서드는 프로퍼티 파일 목록을 인자로 전달받는다. 이때 스프링의 `Resource` 타입을 이용해서 파일 경로를 전달한다.  

위 코드에서 주의해서 볼 점은 빈을 설정하는 메서드가 정적(static) 메서드라는 것이다. 이는 특수한 목적의 빈이기 때문에 정적 메서드로 지정하지 않으면 원하는 방식으로 동작하지 않는다.  

`PropertySourcePlaceholderConfigurer` 타입 빈은 `setLocations()` 메서드로 전달받은 프로퍼티 파일 목록 정보를 읽어와 필요할 때 사용한다. 이를 위한 것이 `@Value` 어노테이션이다.  

```java
@Configuration
public class Config {
    @Value("${db.driver}")
    private String driver;
    @Value("${db.url}")
    private String jdbcUrl;
    @Value("${db.user}")
    private String user;
    @Value("${db.password}")
    private String password;

    @Bean(destroyMethod = "close")
    public DataSource dataSource() {
        DataSource dataSource = new DataSource();
        dataSource.setDriverClassName(driver);
        dataSource.setUrl(jdbcUrl);
        ...
        return dataSource;
    }
}
```  

`@Value` 어노테이션은 **${구분자}** 형식의 플레이스홀더를 값으로 갖고 있는데, `PropertySourcesPlaceholderConfigurer`가 플레이스홀더의 값을 일치하는 프로퍼티 값으로 치환해준다.  

### 2-2 빈 클래스에서 사용하기  

빈으로 사용할 클래스에도 `@Value` 어노테이션을 붙일 수 있다.  

```java
public class Info {

    @Value("${info.version}")
    private String version;

    public void printInfo() {
        System.out.println("version = " + version);
    }

    public void setVersion(String version) {
        this.version = version;
    }
}
```  

`@Value` 어노테이션을 필드에 붙이면 플레이스홀더에 해당하는 프로퍼티를 필드에 할당한다. 혹은 다음과 같이 `@Value` 어노테이션을 set 메서드에 적용할 수도 있다.  

```java
public class Info {

    private String version;

    public void printInfo() {
        System.out.println("version = " + version);
    }

    @Value("${info.version}")
    public void setVersion(String version) {
        this.version = version;
    }
}
```  



<br>

--- 
해당 포스팅은 [초보 웹 개발자를 위한 스프링5 프로그래밍 입문 - 최범균 저](https://www.aladin.co.kr/shop/wproduct.aspx?ItemId=157472828)를 정리하며 작성하였습니다.  

<br>
