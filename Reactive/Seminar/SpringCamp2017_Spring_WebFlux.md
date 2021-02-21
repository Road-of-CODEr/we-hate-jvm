g스프링캠프 2017[Day1 A3]: Spring WebFlux 발표 영상을 사실상 받아쓰기 함.

[Spring 5.0 WebFlux 소개와 개발 방법](https://www.youtube.com/watch?v=2E_1yb8iLKk)

### Spring-WebFlux

- 구 Spring-Web-Reactive
- 스프링5의 메인 테마는 원래 JDK9이었는데 이제는 WebFlux로 바뀜
- 스프링 리액티브 스택의 웹 파트 담당

#### 왜 쓸까?

- 비동기-논블록킹 리액티브 개발에 사용
- 효율적으로 동작하는 고성능 웹 어플리케이션 개발
- 서비스간 호출이 많은 마이크로서비스 아키텍처에 적합

2가지 개발 방식

- 기존의 어노테이션을 활용한 @MVC방식
    - @Controller, @restController ... 와 유사(똑같진 않음)
- 새로운 함수형 모델
    - RouterFunction
    - HandlerFunction

새로운 요청-응답 모델

- 서블릿 스택과 API에서 탈피
    - 서블릿 API는 리액티브 함수형 스타일에 적합하지 않음
- ServerRequest와 ServerResponse라는 새로운 모델

지원 웹 서버/컨테이너

- Servlet 3.1+ (Tomcat, Jetty, ...)
    - 서블릿 3.1+의 비동기-논블록킹 요청 처리 기능 사용
- Netty
- Undertow

### 함수형 스타일 WebFlux

#### 스프링이 웹 요청을 처리하는 방식

1. 요청 매핑
    - 웹 요청을 어느 핸들러로 보낼지 결정
    - URL, 헤더
    - @RequestMapping (기존의 MVC)
2. 요청 바인딩
    - 핸들러에 전달된 웹 요청 준비
    - 웹 URL, 헤더, 쿠키, 바디
3. 핸들러 실행
    - 전달 받은 요청 정보를 이용해 로직을 수행하고 결과를 리턴
4. 핸들러 결과 처리(응답 생성)
    - 눈에 보이지 않지만 핸들러의 리턴 값으로 웹 응답 생성

```java

@RestContoller
public class MyController {
    @GetMapping("/hello/{name}")
        // 요청 매핑
    String hello(@PathVariable String name) { // 요청 바인딩
        return "Hello" + name; // 핸들러 결과 처리 (응답 생성)
    }
}
```

#### RouterFunction

- 함수형 스타일 요청 매핑

```java

@FunctionalInterface
public interface RouterFunction<T extends ServerResponse> {
    Mono<HandlerFunction<T>> route(ServerRequest request);
}
```

ServerRequest request: 웹 플럭스 버전의 웹 요청

Mono<HandlerFunction<T>>: 웹 플럭스 버전의 웹 응답인 ServerResponse나 그 서브타입의 Mono 퍼블리셔를 리턴하는 HandlerFunction의 Mono 타입...

<br>

#### HandlerFunction

- 함수형 스타일의 웹 핸들러(컨트롤러 메소드)
- 웹 요청을 받아 웹 응답을 돌려주는 함수

#### 함수형 WebFlux가 웹 요청을 처리하는 방식

- 요청 매핑 - RouterFunction
- 요청 바인딩 - HandlerFunction
- 핸들러 실행 - HandlerFunction
- 핸들러 결과 처리(응답 생성) - HandlerFunction

#### WebFlux 함수형 Hello/{name}

- HandlerFunction을 먼저 만들고
- RouterFunction에서 path기준 매핑을 함

```java
HandlerFunction helloHandler = req -> {
        String name = req.pathVariable("name"); // ServerRequest.pathVariable()로 {name} 추출
        Mono<String> result = Mono.just("Hello"+name); // 로직 적용 후 결과 값을 Mono에 담는다 
        Mono<serverResponse> res = ServerResponse.ok().body(result, String.class);
        // 웹 응답을 ServerResponse로 만든다
        // Http 응답에는 응답코드, 헤더, 바디가 필요
        // ServerResponse의 빌더를 활용
        // Mono에 담긴 ServerResponse 타입으로 리턴
        return res;
    }
```
위는 HandlerFunction.handle()의 람다식
```Mono<T> handle(ServerRequest request);```

<br>

```java
HandlerFunction helloHander = req ->
    ok().body(fromObject("Hello" + req.pathVariable("name")));
```
함수형 스타일의 코드는 간결하게 작성할 수도 있다
함수형 방식에서는 명시적으로 path variable을 가져옴
content-type이나 기타 헤더는 스프링에서 알아서 넣어줌.

```java
RouterFunction router = req ->
    RequestPredicates.path("/hello/{name}").test(req)? // 웹 요청 정보 중에서 URL경로 패턴 검사
        Mono.just(helloHandler) // 조건에 맞으면 핸들러 함수를 Mono에 담아서 반환  
        : Mono.empty(); // 조건에 맞지 않으면 Mono반환 함수니까 뭐라도 반환해야 함
```

<br>

#### HandlerFunction과 RouteFunction 조합
- RouterFunctions.route(predicate, handler)

```java
RouterFunction router = 
    RouterFunctions.route(RequestPredicates.path("/hello/{name}"), // RouterFunction의 매핑 조건을 체크하는 로직만 발췌
        req -> ServerResponse.ok().body(fromObject("Hello" + // HandlerFunction은 그대로
        req.pathVariable("name"))));
```

<br>

#### RouterFunction의 등록
- RouterFunction 타입의 @Bean으로 만든다

```java
@Bean
RouterFunction helloPathVarRouter(){
 return route(RequestPredicates.path("/hello/{name}"),
    req->ServerResponse.ok().body(fromObject("Hello"+ 
                         req.pathVariable("name"))));
}
```

<br>

#### 핸들러 내부 로직이 복잡하다면 분리한다
- 핸들러 코드만 람다 식을 따로 선언하거나
- 메소드를 정의하고 메소드 참조로 가져온다

```java
HandlerFunction handler = req -> {
    String res = myService.hello(req.pathVariable("name"));
            return ok().body(fromObject(res));
        }; // 다른 bean 호출을 포함한 복잡한 로직을 담은 람다식

    return route(path("/hello/{name}"), handler);
```

```java
@Component
public class HelloHandler {
    @Autowired MyService myService;
    
    Mono<ServerResponse> hello(ServerRequest req){
        String res = myService.hello(req.pathVariable("name"));
        return ok().body(fromObject(res));
    }
}

@Bean
RouterFunction helloRouter(@Autowired HelloHandler helloHandler){
    return route(path("/hello/{name}"), helloHandler::hello);
}
```

handlerFunction은 앞서 말했듯이 람다식으로 사용가능. 람다식은 메소드 타입이 일치하면 일반 메소드와 호환해서 사용가능.    
그래서 앞의 람다식과 동일한 메소드 타입을 가진 메소드를 만들어 주입할 수 있음.   
위의 방법의 단점은 bean 메소드 하나에 handlerFunction을 딱 하나만 연결 해 줄 수 있음.

<br> 

#### RouterFunction의 조합
- 핸들러 하나에 @Bean하나씩 만들어야 하나?
- RouterFunction의 and(), andRoute()등으로 하나의 @Bean에 n개의 RouterFunction을 선언할 수 있다.

#### RouterFunction의 매핑 조건의 중복 
- 타입 레벨 ~ 메소드 레벨의 @RequestMapping 처럼 공통의 조건을 정의하는 것 가능 
- RouterFunction.nest()

```java
public RouterFunction<?> routingFunction(){
    return nest(pathPrefix("/person"), // URL이 /person으로 시작하는 조건을 공통으로
        nest(accept(APPLICATION_JSON), // APPLICATION_JSON을 accept하는 공통조건 중첩
            route(GET("/{id}"),handler::getPerson) 
            .andRoute(method(HttpMethod.GET), handler::listPeople)
        ).andRoute(POST("/").and(contentType(APPLICATION_JSON)),
        handler::createPerson));
        }
```

#### WebFlux 함수형 스타일의 장점
- 모든 웹 요청 처리 작업을 명시적인 코드로 작성
    - 메소드 시그니처 관례와 타입체크가 불가능한 애노테이션에 의존하는 @MVC스타일보다 명확
    - 정확한 타입 체크 가능
- 함수 조합을 통한 편리한 구성, 추상화에 유리
- 테스트 작성의 편리함
    - 핸들러 로직은 물론이고 요청 매핑과 리턴 값 처리까지 단위테스트로 작성 가능
    
#### Webflux 함수형 스타일의 단점
- 함수형 스타일의 코드 작성이 편하지 않으면 코드 작성과 이해 모두 어려움
- 익숙한 방식으로도 가능한데 뭐하러~

### @MVC WebFlux

- 애노테이션과 메소드 현식의 관례를 이용하는 @MVC방식과 유사
- 비동기 + 논븐로킹 리액티브 스타일로 작성

#### ServerRequest, ServerResponse
- Webflux의 기본 요청, 응답 인터페이스 사용
- 함수형 WebFlux의 HandlerFunction을 메소드로 만들었을 때와 유사
- 매핑만 애노테이션 방식을 이용


예시 코드
```java
@RestController
public static class MyController{
    @RequestMapping("/hello/{name}")
    Mono<ServerResponse> hello(ServerRequest req){
        return ok().body(fromObject(req.pathVariable("name"))); // 메소드로 재정의된 HandlerFunction
    }
}
```

<br>

#### @MVC 요청 바인딩과 Mono/Flux 리턴 값
- 가장 대표적인 @MVC WebFlux 작성 방식 
- 파라미터 바인딩은 @MVC 방식 그대로
- 핸들러 로직 코드의 결과를 Mono/Flux 타입으로 리턴

```java
@GetMapping("/hello/{name}")
Mono<String> hello(@PathVariable String name){
    return Mono.just("Hello" + name);
}

@RequestMapping("/hello")
Mono<String> hello(User user){ // 커맨드 오브젝트, 모델 오브젝트 바인딩 / URL 파라미터 또는 form-data 
    return Mono.just("Hello" + user.getName());
}
``` 
<br>

#### @RequestBody 바인딩(Json,XML)
- T
- Mono<T>
- Flux<T>

```java
@RequestMapping("/hello")
Mono<String> hello(@RequestBody User user){  //웹 요청의 body를 MessageConverter에서 바인딩(@MVC와 동일)
    return Mono.just("Hello" + user.getName());
}

@RequestMapping("/hello")
Mono<String> hello(@RequestBody Mono<User> user){  //반환된 오브젝트를 Mono에 담아서 전달
    return user.map(u->"Hello" + u.getName()); //Mono의 연산자를 사용해서 로직을 수행하고 Mono로 리턴
}

@RequestMapping("/hello")
Flux<String> hello(@RequestBody Flux<User> user){  //Flux같은 경우는 오브젝트 하나로 받기 보다는 데이터 스트림을 클라이언트로 받을 때 사용
    return user.map(u->"Hello" + u.getName()); //User의 스트림 형태로 로직 실행
}
```

<br>

#### @ResponseBody 리턴 값 타입
- T
- Mono<T>
- Flux<T>
- Flux<ServerSideEvent>
- void
- Mono<Void>


### WebFlux와 리액티브 기술 (WebClient + Reactive Data)

#### WebFlux만으로 성능이 좋아질까?
- 비동기-논블록킹 구조의 장점은 블록킹IO를 제거하는 데서 나온다
- HTTP서버가 클라이언트로 부터 응답을 받는 것은 예전부터 논블럭킹 방식이었음. (거의 톰캣 6, 서블릿2시절부터도)

#### 개선할 블록킹 IO
- 데이터 액세스 리포지토리
- HTTP API 호출
- 기타 네트워크 사용 서비스

#### JPA - JDBC 기방 RDB연결
- 현재는 답이 없다.
- 블로킹 메소드로 점철된 JDBC API
- 일부 DB는 논블록킹 드라이버가 존재하지만...
- @Async 비동기 호출과 CFuture를 리액티브로 연결하고 쓰레드풀 관리를 통해서 웹 연결 자원을 효율화하는 정도

#### Spring Data JPA 비동기 쿼리 결과 방식
- 리포지토리 소드의 리턴 값을 @Async 메소드 처럼 작성 
```java
@Async
CompletableFuture<User> findOneByFirstname(String firstname);
// 리포지토리 메소드를 비동기로 실행하고 결과를 CompletableFuture로 돌려받는다.
// @Async이므로 비동기 실행과 동시에 메소드 리턴

@GetMapping
Mono<User> findUser(String name){
    return Mono.fromCompletionStage(myRepository.findOneByFirstName(name));
}
// DB를 액세스 하는 구간 동안은 블로킹이 일어나지만 서블릿 쓰레드를 점유하지 않은채로 동작 가능 
```

<br>

#### 본격 리액티브 데이터 액세스 기술
- 스프링 데이터의 리액티브 리포지토리 이용 
    - MongoDB
    - Cassandra
    - Redis
- ReactiveCrudRepository 확장

#### 논블록킹 API 호출은 WebClient
- AsyncRestTemplat의 리액티브 버전
- 요청을 Mono/Flux로 전달할 수 있고
- 응답을 Mono/Flux 형태로 가져옴

##### 함수형 스타일 코드가 읽기 어렵다면!
- 각 단계의 타입이 보이지 않기 때문이다.
- 타입이 보이도록 코드를 재구성하고 익숙해지도록 연습이 필요함

#### 비동기 논블록킹 리액티브 웹 어플리케이션의 효과를 얻으려면
- Webflux
    - + 리액티브 리포지토리
    - + 리액티브 원격 API 호출
    - + 리액티브 지원 외부 서비스 (메시지 큐 등등)
    - + @Async 블록킹 IO
- 코드에서 블록킹 작업이 발생하지 않도록 Flux스트림 또는 Mono에 데이터를 넣어서 전달

#### 리액티브 함수형은 꼭 성능 때문에 써야한다?
- 함수형 스타일 코드를 이용해 간결하고 읽기 좋고 조합하기 편한 코드 작성
- 데이터 흐름에 다양한 오퍼레이터 적용
- 연산을 조합해서 만든 동시성 정보가 노출되지 않는 추상화된 코드 작성
    - 동기, 비동기, 블록킹, 논블록킹 등을 유연하게 적용
- 데이터의 흐름의 속도를 제어할 수 있는 메커니즘 제공

#### 논블록킹 IO에만 효과가 있나?
- 시스템 외부에서 발생하는 이벤트에도 유용
- 클라이언트로부터의 이벤트에도 활용 가능

#### ReactiveStreams
- WebFlux가 사용하는 Reactor외에 RxJava2를 비롯한 다양한 리액티브 기술에 적용된 표준 인터페이스
- 다양한 기술, 서비스 간의 상호 호환성에 유리
- 자바9에 Flow API로 포함