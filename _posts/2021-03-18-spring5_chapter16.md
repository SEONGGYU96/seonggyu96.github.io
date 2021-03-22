---
layout: post
title: "Spring5 : JSON 응답과 요청 처리"
subtitle: "API 구축"
author: "GuGyu"
header-style: text
tags:
  - Spring
  - Spring5
  - Java
  - Study
  - Server
  - JSON
  - ResponseEntity
---

웹 페이지에서 Ajax를 이용해서 서버 API를 호출하는 사이트가 많다. 이들 API는 HTML 대신 JSON이나 XML을 응답으로 사용한다. 이때 GET, POST 뿐만 아니라 ~~PUT, DELETE와 같은 다른 방식도 함께 사용한다.~~ (PUT, DELETE 메서드는 보안에 취약하여 현업에서는 사용하지 않는다고 한다) 스프링 MVC를 사용하면 이를 위한 웹 컨트롤러를 쉽게 만들 수 있다.  


## JSON  

JSON(JavaScript Object Notation)은 간단한 형식을 갖는 문자열로 데이터 교환에 주로 사용한다.  
```json
{
    "name": "GuGyu",
    "github": "https://github.com/SEONGGYU96",
    "projects": [
        {
            "title": "santa-manitto",
            "year": 2021
        },
        {
            "title": "STORM",
            "year": 2020
        }
    ]
}
```  

### 규칙  

- 중괄호를 사용해서 객체를 표현한다.
- 객체는 (이름, 값) 쌍을 갖는다.
- 이름과 값은 콜론(:)으로 구분한다.

값이 가질 수 있는 형태는 다음과 같다.

- 문자열, 숫자, 불리언, null
- 배열
- 다른 객체  

문자열은 큰따옴표나 작은따옴표 사이에 위치한 값이다. 문자열은 `\"`(큰따옴표), `\n`(개행), `\r`(캐리지 리턴), `\t`(탭)과 같이 역슬래시를 이용해서 특수 문자를 표시할 수 있다.  

숫자는 10진수 표기법이나 지수 표기법(ex 1.07e2)을 따른다. 불리언 타입 값은 ture와 false가 있다.  

배열은 대괄호로 표현한다. 대괄호 안에 콤마로 구분한 값 목록을 갖는다.  

<br>

## Jackson  

Jackson은 자바 객체와 JSON 형식 문자열 간 변환을 처리하는 라이브러리이다.  

```gradle
//Jackson core와 Jackson Annotation
implementation "com.fasterxml.jackson.core:jackson-databind:$jacksonVersion"

//java8 이상의 date/time 지원을 위한 Jackson 모듈  
implementation "com.fasterxml.jackson.datatype:jackson-datatype-jsr310:$jacksonVersion"
```  

Jackson은 프로퍼티의 이름과 값을 JSONs 객체의 (이름, 값) 쌍으로 사용한다. 프로퍼티 타입이 배열이나 List인 경우 JSON 배열로 변환된다.  

### 스프링 MVC에서의 동작 방식  

스프링 3.0 이후로 컨트롤러 리턴 방식이 `@RequestBody` 형식이라면 스프링은 `MessageConverter`를 통해 리턴하는 객체를 후킹한다. 그런데 스프링 3.1 이후로 Jackson에 대한 의존 설정을 했다면 Jackson이 제공하는 `MappingJacksonHttpMaessageConverter`를 스프링 MVC가 사용할 `MessageConverter`로 **자동**등록한다.  

스프링 MVC가 제공하는 Jackson과의 호환성 덕분에 다양한 JSON 변환 라이브러리 (GSON, SimpleJSON, ...) 보다 Jackson을 많이 사용한다.  

<br>

## @RestController

스프링 MVC에서 JSON 형식으로 데이터를 응답하는 것은 매우 간단하다. `@Controller` 어노테이션 대신 `@RestController` 어노테이션을 사용하면 된다.  

```java
@RestoController
public class RestMemeberController {
    ...

    @GetMapping("/api/members")
    public List<Member> memebers() {
        return memberDao.selectAll();
    }

    @GetMapping("/api/members/{id}")
    public Memeber member(@PathVariable Long id, HttpServletResponse response) throws IOException {
        Memeber memeber = memeberDao.selectById(id);
        if (member == null) {
            response.sendError(HttpServletResponse.SC_NOT_FOUND);
            return null;
        }
        return member;
    }
    ...
}
```  

`@RestController` 어노테이션을 붙이면 스프링 MVC는 요청 매핑 어노테이션을 붙인 메서드가 리턴한 객체를 알맞은 형식으로 변환해서 응답 데이터로 전송한다. 이때 Jackson에 대한 의존을 추가했다면 JSON 형식의 문자열로 변환해서 응답한다. 

### @JsonIgnore  

보통 암호와 같이 민감한 데이터는 응답 결과에 포함시키지 않는다. 따라서 Jaskcon이 제공하는 `@JsonIgnore` 어노테이션을 사용하여 객체에서 민감한 데이터나 불필요한 데이터를 빼고 응답 결과를 만들 수 있다.  

```java
public class Member {
    private Long id;
    private String email;
    @JsonIgnore private password;
    private String name;
    private LocalDateTime registerDateTime;
}
```  
### @JsonFormat  

Json은 기본적으로는 `LocalDateTime` 타입을 배열로 변환시킨다. 예를 들어 2021-03-17 13:30:00은 [2021, 3, 17, 13, 30, 0]이 된다. 보통 날짜나 시간은 배열보다는 특정한 형식을 갖는 문자열로 표현하는 것을 선호한다. `@JsonFormat` 어노테이션을 사용하면 이를 해결할 수 있다. 만약 ISO-8601 형식으로 변환하고 싶다면 shape 속성 값으로 `Shape.STRING`을 갖는 `@JsonFormat` 어노테이션을 변환 대상에 적용하면 된다.  

```java
public class Memeber {
    private Long id;
    private String email;
    private String name;
    @JsonFormat(shape = Shape.STRING)
    private LocalDateTime registerDateTime;
}
```

위와 같은 객체를 JSON으로 변환하면 다음과 같이 나타낼 수 있다.  

```json
{
    "id": 1,
    "email": "gugyu@github.io",
    "name": "GuGyu",
    "registerDateTime": "2021-03-17T13:30:00"
}
```  

물론 ISO-8601 형식이 아닌 형식으로도 변환할 수 있다. 이때는 pattern 속성을 사용하면 된다.  

```java
public class Memeber {
    private Long id;
    private String email;
    private String name;
    @JsonFormat(pattern = "yyyyMMddHHmmss")
    private LocalDateTime registerDateTime;
}
```  

pattern 속성은 `java.time.format.DateTimeFormatter` 클래스나 `java.text.SimpleDateFormat` 클래스의 API 문서에 정의된 패턴을 따른다.  

### 형식 변환 기본 설정

날짜 형식을 변환할 모든 대상에 `@JsonFormat` 어노테이션을 붙여야한다면 상당히 귀찮을 것이다. 일정한 변환 규칙을 모든 날짜 타입에 적용하려면 스프링 MVC 설정을 변경해야한다.  

```java
@Configuration
@EnableWebMvc
public class MvcConfig implements WebMvcConfigurer {
    ...
    @Override
    public void extendMessageConverters(List<HttpMessageConverter<?>> converters) {
        ObjectMapper objectMapper = Jackson2ObjectMapperBuilder
            .json()
            .featuresToDisable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS)
            .build();
        converters.add(0, new MappingJackson2HttpMessageConverter(objectMapper));
    }
}
```

`extendMessageConverters()` 메서드는 `WebMvcConfigurer` 인터페이스에 정의된 메서드로서 `HttpMessageConverter`를 추가로 설정할 때 사용한다. `@EnableWebMvc` 어노테이션을 사용하면 여러 형식으로 변환할 수 있는 `HttpMessageConverter`를 미리 등록한다. `extendMessageConverters()`는 등록된 `HttpMessageConverter` 목록을 파라미터로 받는다.  

미리 등록된 `HttpMessageConverter`에는 Jackson을 이용하는 것도 포함되어 있기 때문에 새로 생성한 `HttpMessageConverter`는 목록의 제일 앞에 위치시켜야한다. 그래서 `converters.add(0, ...)` 처럼 인덱스를 지정해 추가해주었다.  

`Jackson2ObjectMapperBuilder`는 `ObjectMapper`를 보다 쉽게 생성할 수 있도록 **스프링이** 제공하는 클래스이다. 위 코드에서는 `featuresToDisable()`에서 `WRITE_DATES_AS_TIMESTAMP`를 비활성화하는데 Jackson이 날짜 형식을 출력할 때 유닉스 타임 스탬프로 출력하는 기능을 비활성화 하는 것이다. 이 기능을 비활성화하면 `ObjectMapper`는 날짜 타입의 값을 ISO-8601 형식으로 출력한다.

`java.util.Date` 타입의 값을 원하는 형식으로 출력하고 싶다면 아래와 같이 설정할 수 있다.  

```java
@Configuration
@EnableWebMvc
public class MvcConfig implements WebMvcConfigurer {
    ...
    @Override
    public void extendMessageConverters(List<HttpMessageConverter<?>> converters) {
        ObjectMapper objectMapper = Jackson2ObjectMapperBuilder
            .json()
            .simpleDateFormat("yyyyMMddHHmmmm")
            .build();
        converters.add(0, new MappingJackson2HttpMessageConverter(objectMapper));
    }
}
```

그러나 위 방법은 `LocalDateTime` 변환에는 적용되지 않는다. `LocalDateTime` 타입에 대해 원하는 패턴을 설정하고 싶다면 `serializerByType()` 메서드를 이용해서 `JsonSerializer`를 직접 설정하면 된다.  

```java
@Configuration
@EnableWebMvc
public class MvcConfig implements WebMvcConfigurer {
    ...
    @Override
    public void extendMessageConverters(List<HttpMessageConverter<?>> converters) {
        DateTimeFormatter formatter = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss");
        ObjectMapper objectMapper = Jackson2ObjectMapperBuilder
            .json()
            .serializerByType(LocalDateTime.class, new LocalDateTimeSerializer(formatter))
            .build();
        converters.add(0, new MappingJackson2HttpMessageConverter(objectMapper));
    }
}
```  

<br>

## @RequestBody  

지금까지는 응답을 JSON으로 변환하는 것에 대해 살펴봤다. 이제 반대로 JSON 형식의 요청 데이터를 자바 객체로 변환하는 기능에 대해 살펴보자. POST 방식이나 PUT 방식을 사용하면 name=이름&age=17과 같은 쿼리 문자열 형식이 아니라 다음과 같은 JSON 형식의 데이터를 요청 데이터로 전송할 수 있다.  

```json
{
    "name": "이름",
    "age": 17
}
```

JSON 형식으로 전송된 요청 데이터를 커맨드 객체로 전달받는 방법은 매우 간단하다. 커맨드 객체에 @RequestBody 어노테이션을 붙이기만 하면 된다.  

```java
@RestController
public class RestMemberController {
    ...
    @PostMapping("/api/members")
    public void newMember(
        @RequestBody @Valid RegisterRequest registerRequest,
        HttpServletResponse response
    ) throws IOException {
        try {
            Long newMemberId = registerService.regist(registerRequest);
            response.setHeader("Location", "/api/members/" + newMemberId);
            response.setStatus(HttpServletResponse.SC_CREATED);
        } catch (DuplicateMemeberException exception) {
            response.sendError(HttpServletResponse.SC_CONFLICT);
        }
    }
    ...
}
```  

`@RequestBody` 어노테이션을 커맨드 객체에 붙이면 JSON 형식의 문자열을 해당 자바 객체로 변환한다. 

스프링 MVC가 JSON 형식으로 전송된 데이터를 올바르게 처리하려면 요청 컨텐츠 타입이 `application/json` 이어야 한다. 보통 POST 방식의 폼 데이터는 쿼리 문자열인 "p1=v1&p2=v2"로 전송되는데 이때 컨텐츠 타입은 `application/x-www-form-urlencoded`이다. 쿼리 문자열 대신 JSON 형식을 사용하려면 `application/json` 타입으로 데이터를 전송할 수 있는 별도 프로그램(Post man 등)이 필요하다.  

### JSON 데이터의 날짜 형식  

JSON 형식의 데이터를 날짜 형식으로 변환하는 방법을 살펴보자. 별도 설정을 하지 않으면 `yyyy-MM-ddTHH:mm:ss` 형식의 문자열을 LocalDateTime과 Date로 변환한다.  

특정 패턴을 가진 문자열을 변환하고싶다면 객체를 JSON으로 변환할 때와 동일하게 `@JsonFormat` 어노테이션의 pattern 속성을 사용해서 패턴을 지정한다.  

```java
@JsonFormat(pattern = "yyyyMMddHHmmss")
private LocalDateTime birthDateTime;

@JsonFormat(pattern = "yyyyMMdd HHmmss")
private Date birthDate;
```

특정 속성이 아니라 해당 타입을 갖는 모든 속성에 적용하고 싶다면 스프링 MVC 설정을 추가하면 된다.  

```java
@Configuration
@EnableWebMvc
public class MvcConfig implements WebMvcConfigurer {
    ...
    @Override
    public void extendMessageConverters(List<HttpMessageConverter<?>> converters) {
        DateTimeFormatter formatter = DateTimeFormatter.ofPattern("yyyyMMddHHmmss");
        ObjectMapper objectMapper = Jackson2ObjectMapperBuilder
            .json()
            .deserializerByType(LocalDateTime.class, new LocalDateTimeDeserializer(formatter))
            .simpleDateFormat("yyyyMMdd HHmmss")
            .build(); 
        converters.add(0, new MappingJackson2HttpMessageConverter(objectMapper));
    }
}
```  

객체를 JSON으로 변환할 때는 `serializerByType`을 사용했지만 반대는 `deserializerByType`을 사용한다. 그러나 Date는 방향과 무관하게 `simpleDateFormat()`을 사용한다.  

### 요청 객체 검증  

`@RequestBody`를 통해 JSON 형식으로부터 변환한 객체도 `@Valid` 어노테이션이나 별도 `Validator`를 이용해서 검증할 수 있다. `@Valid` 어노테이션을 사용한 경우 검증에 실패하면 400 상태 코드를 응답한다.  

`Validator`를 사용할 경우 다음과 같이 직접 상태 코드를 처리해야 한다.  

```java
@PostMapping("/api/members")
public void newMember(
    @RequestBody RegisterRequest registerRequest, Errors errors, 
    HttpServletResponse response) throws IOException {
        try {
            new RegisterRequestValidator().validate(registerRequest, errors);
            if (errors.hasErrors()) {
                response.sendError(HttpServletResponse.SC_BAD_REQUEST);
                return;
            }
            ...
        } catch (DuplicateMemberException exception) {
            response.sendError(HttpServletResponse.SC_CONFLICT);
        }
}
```  

<br>

## ResponseEntity  

지금까지는 상태 코드를 지정하기 위해 `HttpSevletResponse`의 `setStatus()` 메서드와 `sendError()` 메서드를 사용했다.  

```java
@GetMapping("/api/members/{id}")
public Member member(@PathVariable Long id, HttpServletResponse response) throws IOException {
    Member member = memberDao.selectById(id);
    if (member == null) {
        response.sendError(HttpServletResponse.SC_NOT_FOUND);
        return null;
    }
    return member;
}
```  

그러나 이 방법으로 404 응답을 반환하면 JSON 형식이 아니라 서버가 기본으로 제공하는 HTML을 응답 결과로 제공한다. 404나 500과 같이 처리에 실패한 경우 HTML 응답 데이터 대신에 JSON 형식의 응답 데이터를 전송하기 위해서 `ResponseEntity`를 사용할 수 있다.  

먼제 에러 상황일 때 응답으로 사용할 `ErrorResponse` 클래스를 다음과 같이 작성하자.  

```java
public class ErrorResponse {
    private String message;

    public ErrorResponse(String message) {
        this.message = message;
    }

    public String getMessage() {
        return message;
    }
}
```  

`ResponseEntity`를 이용하면 `member()` 메서드를 다음과 같이 구현할 수 있다.  

```java
@RestController
public class RestMemberController {
    private MemberDao memberDao;
    ...
    @GetMapping("/api/members/{id}")
    public ResponseEntity<Object> member(@PathVariable Long id) {
        Member member = memberDao.selectById(id);
        if (member == null) {
            return ResponseEntity.status(HttpStatus.NOT_FOUND)
                .body(new ErrorResponse("no member"));
        }
        return ResponseEntity.status(HttpStatus.OK).body(member);
    }
}
```  

스프링 MVC는 리턴 타입이 `ResponseEntity`이면 `ResponseEntity`의 body로 지정한 객체를 사용해서 변환을 처리한다. 위 코드를 예로 들자면 해당 id에 해당하는 member가 null일 때는 `ErrorResponse`를 JSON으로 변환하고 null이 아닐 때는 `Member` 객체를 변환한다.  

`ResponseEntity`의 status로 지정한 값을 응답 상태 코드로 사용한다. null일 때는 404(NOT_FOUND)를 상태 코드로 응답하고 null이 아닐 때는 200(OK)를 상태 코드로 응답한다.  

이처럼 `ResponseEntity`를 생성하는 기본 방법은 status와 body를 이용해서 상태 코드와 JSON으로 변환할 객체를 지정하는 것이다.  

- `ResponseEntity.status(상태코드).body(객체)`  

상태 코드는 `HttpStauts` 열거 타입에 정의된 값을 이용해서 정의한다.  

200(OK) 응답 코드와 몸체 데이터를 생성할 경우 다음과 같이 ok() 메서드를 이용해서 생성할 수도 있다.  

- `ResponseEntity.ok(member)`

만약 몸체 내용이 없다면 다음과 같이 body를 지정하지 않고 `build()`로 바로 생성한다.  

- `ResponseEntity.status(HttpStatus.NOT_FOUND).build()`  

몸체가 없을 때 `status()` 대신 사용할 수 있는 메서드는 다음과 같다.  

- `noContent()` : 204
- `badRequest()` : 400
- `notFound()`: 404  

Location 헤더를 함께 전송하려면 `ResponseEntity.created()` 메서드에 URI를 전달하면 된다.  

```java
@RestController
public class RestMemberController {
    private MemberDao memberDao;
    ...
    @GetMapping("/api/members")
    public ResponseEntity<Object> newMember(
        @RequestBody @Valid RegisterRequeset registerRequest) {
            try {
                Long newMemberId = registerService.regiset(registerRequest);
                URI uri = URI.create("/api/members/" + newMemberId);
                return ResponseEntity.created(uri).build();
            }
        } catch ...
}
```  

### @ExceptionHandler  

한 메서드에서 정상 응답과 에러 응답을 `ResponseBody`로 생성하면 코드가 중복될 수 있다.  

```java
@GetMapping("/api/members/{id}")
public ResponseEntity<Object> member(@PathVariable Long id) {
    if (member == null) {
        return ResponseEntity
            .status(HttpStatus.NOT_FOUND)
            .body(new ErrorResponse("no member"));
    }
    return ResponseEntity.ok(member);
}
```  

위 코드는 member가 존재하지 않을 때 기본 HTML 에러 응답 대신에 JSON 응답을 제공하기 위해 `ResponseEntity`를 사용했다. 그런데 회원이 존재하지 않을 때 404 상태 코드를 응답해야하는 기능이 많다면 에러 응답을 위해 `ResponseEntity`를 생성하는 코드가 여러 곳에 중복된다.  

이럴 때 `@ExceptionHandler` 어노테이션을 적용한 메서드에서 에러 응답을 처리하도록 구현하면 중복을 없앨 수 있다.  

```java
@GetMapping("/api/members/{id}")
public ResponseEntity<Object> member(@PathVariable Long id) {
    if (member == null) {
        throw new MemberNotFoundException();
    }
    return ResponseEntity.ok(member);
}

@ExceptionHander(MemberNotFoundException.class)
public ResponseEntity<ErrorResponse> handleNoDate() {
    return ResponseEntity
            .status(HttpStatus.NOT_FOUND)
            .body(new ErrorResponse("no member"));
}
```  

회원 정보가 존재하지 않으면 `MemberNotFoundException()`를 발생한다. 이 예외가 발생하면 `@ExceptionHandler` 어노테이션을 사용한 `handleNoDate()` 메서드가 에러를 처리한다. 이는 404 상태 코드와 `ErrorResponse` 객체를 몸체로 갖는 `ResponseEntity`를 리턴한다. 따라서 `MemberNotFoundException`이 발생하면 404의 JSON 응답을 전송한다.  

`@RestControllerAdvice` 어노테이션을 이용해서 에러 처리코드를 별도 클래스로 분리할 수도 있다. `@RestControllerAdvice` 어노테이션은 `@ControllerAdvice` 어노테이션과 동일하다. 차이라면 `@RestContoller` 어노테이션과 동일하게 응답을 JSON이나 XML과 같은 형식으로 변환한다는 것이다.  

```java
@RestControllerAdvice("controller")
public class ApiExceptionAdvice {
    @ExceptionHandler(MemberNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNoDate() {
        return ResponseEntity
                .status(HttpStatus.NOT_FOUND)
                .body(new ErrorResponse("no member"));
    }
}
```  

`@RestControllerAdvice` 어노테이션을 사용하면 에러 처리 코드가 한 곳에 모여 효과적으로 에러 응답을 관리할 수 있다.  

### @Valid 에러 결과 응답  

`@Valid` 어노테이션을 붙인 커맨드 객체가 값 검증에 실패하면 400 상태 코드를 응답하는데, 문제는 `HttpServeltResponse`를 이용해서 상태 코드를 응답했을 때와 마찬가지로 HTML 응답을 전송한다는 점이다. JSON 형식 응답을 제공하고 싶다면 다음과 같이 `Errors` 타입 파라미터를 추가해서 직접 에러 응답을 생성하면 된다.  

```java
@PostMapping("/api/members")
public ResponseEntity<Object> newMember(
    @RequestBody @Valid RegisterRequest registerRequest,
    Errors errors
) {
    if (erros.hasErrors()) {
        String errorCodes = errors.getAllErrors() //List<ObjectError>
            .stream()
            .map(error -> error.getCodes()[0]) //error는 ObjectError
            .collect(Collectors.joining("."));
        return ResponseEntity
            .status(HttpStatus.BAD_REQUEST)
            .body(new ErrorResponse("errorCodes = " + errorCodes));
    }
    ...
}
```  

위 코드는 검증 에러가 존재하면 `getAllErrors()` 메서드로 모든 에러 정보를 구하고, 각 에러 코드 값을 연결한 문자열을 생성해서 `errorCodes` 변수에 할당한다. 그리고 이 변수를 `ErrorResponse`에 담아 JSON 형태로 전송하는 것이다.  

`@RequestBody` 어노테이션을 붙인 경우 `@Valid` 어노테이션을 붙인 객체의 검증에 실패했을 때 `Errors` 타입 파라미터가 존재하지 않으면 `MethodArgumentNotValidException`이 발생한다. 따라서 다음과 같이 `@ExceptionHandler` 어노테이션을 이용해서 검증 실패시 에러 응답을 생성해도 된다.  

```java
@RestControllerAdvice("controller")
public class ApiExceptionAdvice {
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleBindException() {
        String errorCodes = errors.getAllErrors()
            .stream()
            .map(error -> error.getCodes()[0])
            .collect(Collectors.joining("."));
        return ResponseEntity
            .status(HttpStatus.BAD_REQUEST)
            .body(new ErrorResponse("errorCodes = " + errorCodes));
    }
}
```

<br>

--- 
해당 포스팅은 [초보 웹 개발자를 위한 스프링5 프로그래밍 입문 - 최범균 저](https://www.aladin.co.kr/shop/wproduct.aspx?ItemId=157472828)를 정리하며 작성하였습니다.  

<br>

참고 :  
[[Spring] Jackson 라이브러리 이해하기.](https://mommoo.tistory.com/83)
