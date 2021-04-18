# Feign Client

spring cloud의 중요 요소중의 하나인 `feign client` 에 대해서 알아보자

## 목차
- [Feign Client](#feign-client)
  - [목차](#목차)
  - [EnableFeignClient](#enablefeignclient)
  - [FeignClient](#feignclient)
    - [별다른 구성 없이 파싱하는 경우](#별다른-구성-없이-파싱하는-경우)
    - [직접 파싱하는 경우](#직접-파싱하는-경우)
    - [Retry](#retry)
  - [Testing](#testing)
    - [ContractTest](#contracttest)
    - [WireMockTest](#wiremocktest)
  - [참고 GitHub](#참고-github)

## EnableFeignClient

가장 중요한 것은 역시 설정 파일인데요  
feign 에서 제공하는 `@EnableFeignClients` 를 활용하면 정말 좋습니다  
기본적으로

1. class 을 가져오는 방식과
2. basepackage 를 기반으로 scan 하는 방식

이 있는데요  
basePackage를 기반으로 가져온다면.. 더 많이 편리하게 사용할 수 있겠죠?  
feignClient가 추가될 때마다 환경 설정들을 추가적으로 할 필요가 없으니까요
<br>

또한, 보통 Feign 레벨의 client를 한번 더 감싸서 어떻게 처리할 것인지 결정하게 되는데요  
이 경우에도 같이 `@ComponentScan` 을 같이 해준다면,  
통합 테스트를 실시할 때 감싸진 Bean 들도 같이 가져올 수 있게 됩니다  

```kotlin
@Configuration
@EnableFeignClients(basePackageClasses = [BaseFeignClientPackage::class])
@ComponentScan(basePackageClasses = [BaseFeignClientPackage::class])
class FeignClientConfiguration
```

## FeignClient

인터페이스 기반이므로 사용법이란 참 간단합니다  

```kotlin
@FeignClient(value = "placeholder", url = "\${external.api.placeHolder}")
interface PlaceHolderFeignClient {

    @GetMapping("/posts")
    fun posts(): Response

    @GetMapping("/posts/{id}")
    fun post(@PathVariable("id") id: Long): Response
}
```

`FeignClient` 를 활용할 때는 2가지 방식이 있는데요.

1. Feign 에서 제공하는 `AsyncResponseHandler` 를 활용해서 response를 파싱하는 방법
2. 직접 구현해서 파싱하는 방법

### 별다른 구성 없이 파싱하는 경우

기본적으로 feign은 HttpStatus `200` 번대는 정상 케이스임을 인지하고  
ResponseBody에 대한 파싱이 이루어지게 됩니다  
그 외의 상황이라면 `FeignException`으로 감싸서 예외로 던져지게 되죠  

```java
else if (response.status() >= 200 && response.status() < 300) {
  if (isVoidType(returnType)) {
    resultFuture.complete(null);
  } else {
    final Object result = decode(response, returnType);
    shouldClose = closeAfterDecode;
    resultFuture.complete(result);
  }
```

다만 404의 경우에는 조금 예외인데요  
`404` 상황에 에러를 던질 것인지, 정상으로 인지하고 Body를 파싱할 것인지  
옵션을 줄 수 있습니다  

```java
/**
 * @return whether 404s should be decoded instead of throwing FeignExceptions
 */
boolean decode404() default false;
```

### 직접 파싱하는 경우


Feign에서 ResponseBody에 대한 정보는 `ByteArrayBody` 에 담겨져 있는데요  
해당 클래스를 생성하는 시점에 response에 대한 byteArray 정보를 같이 저장하게 됩니다.!  

```java
@Override
public String toString() {
    return decodeOrDefault(data, UTF_8, "Binary data");
}
```

해당 기능을 적절하게 이용하면 아래와 같이 직접파싱할 수도 있죠  
다만, 이 경우에는 `Feign.Response` 객체를 적절히 활용해야 됩니다  

```kotlin
val responseBody = objectMapper.readValue<RestPlaceHolderPost>(response.body().toString())
```

`MSA` 상황 속에서는 가장 힘든게 어디서 문제가 터졌는지입니다  
기본적으로 Request, Response 에 대한 로깅이 필수적인 요소로 작용하는데요  
신기하게도 위 Response 객체에는 **Request** 에 대한 정보도 담겨 있습니다  

```java
/**
 * the request that generated this response
 */
public Request request() {
    return request;
}
```

그러니 적절히 활용하면, 직접 로깅도 가능하겠죠..? ㅎㅎ  

### Retry

뭐 여러 방법들이 있겠지만..!

1. Feign에서 제공하는 `Retry` 객체를 활용하는 방식
2. Spring에서 제공하는 `@Retryable` 을 활용하는 방식

```kotlin
@Bean
fun retryer() = Retryer.Default(1000, 2000, 3)
```

다만 feign에서 제공하는 Retryer 는 Global 설정이므로  
저는 선호하지는 않습니다  

특정 상황에서만 재시도하고 싶은 경우와 재시도에 대한 정책은 달라질 수 있으니까요  

```kotlin
@Retryable(backoff = Backoff(delay = 500L), maxAttempts = 3)
@GetMapping("/posts/{id}")
fun post(@PathVariable("id") id: Long): Response
```

그래서 가급적 `@Retryable` 을 활용하는 편입니다! ㅎㅎ  


## Testing

테스트는 2가지 방식을 소개해드릴께요

1. 직접 API에 대한 규약을 테스트 하는 방식
2. mock server로 다양한 httpStatus를 테스트하는 방식

### ContractTest

API 간의 규약테스트가 바로 ContractTest 입니다

즉, 개발환경으로 떠 있는 API를 직접 찌르는 것이죠  

```kotlin
@ImportAutoConfiguration(value = [FeignAutoConfiguration::class])
@AutoConfigureJson
annotation class AutoConfigureTestFeign {}

@AutoConfigureTestFeign
@SpringBootTest(classes = [FeignClientConfiguration::class])
abstract class FeignContractTest {
    @Autowired
    protected lateinit var objectMapper: ObjectMapper
}
```

설정은 위와 같이 가져갑니다  
실제 host를 찔러서 응답을 가져오는 방식의 테스트죠  
개발 환경에서 제공하는 데이터들을 확인하고 싶을 때, 진행하게 됩니다  

```
GET https://jsonplaceholder.typicode.com/posts/1 HTTP/1.1

Binary data, response : {
  "userId": 1,
  "id": 1,
  "title": "sunt aut facere repellat provident occaecati excepturi optio reprehenderit",
  "body": "quia et suscipit\nsuscipit recusandae consequuntur expedita et cum\nreprehenderit molestiae ut ut quas totam\nnostrum rerum est autem sunt rem eveniet architecto"
}
```


### WireMockTest

WireMock이라는 Mock Server로 통해서 테스트를 진행하는 방식인데요  
위 테스트의 장점은 실제 서버로부터 받지 못하는 상황들을 테스트할 수 있습니다  
대표적으로 `503` 혹은 `404` 의 실패상황을 실제로 연출할 수 있게 되죠.!  

```groovy
testImplementation("org.springframework.cloud:spring-cloud-contract-wiremock")
```

spring cloud의 wireMock 의존성을 사용할 것이구요..

```kotlin
@ActiveProfiles("wiremock")
@AutoConfigureWireMock(port = 0)
@AutoConfigureTestFeign
@SpringBootTest(
    classes = [FeignClientConfiguration::class]
)
abstract class FeignWireMockTest {
    @Autowired
    protected lateinit var objectMapper: ObjectMapper
}
```

`@AutoConfigureWireMock` 이라는 어노테이션을 통해서 가짜 서버가 띄워지게 되는 방식이죠.!  

그러면 아래처럼 테스트를 작성해서..!

```kotlin
@Test
fun `503이면 IllegalStateException을 던진다`() {
    // given
    stubFor(
        get(anyUrl())
            .willReturn(serviceUnavailable())
    )
    // when
    val exception = catchThrowable { client.posts() }
    // then
    verify(
        getRequestedFor(urlEqualTo("/posts"))
    )
    assertThat(exception).isInstanceOf(IllegalStateException::class.java)
}
```

503 이라는 응답을 mocking해서 받게되고,  
해당 상황에 대한 커스터마이징한 **Exception** 이 제대로 발생하는지 검증하는 테스트입니다  

```
127.0.0.1 - GET /posts

Accept: [*/*]
User-Agent: [Java/15.0.1]
Connection: [keep-alive]
Host: [localhost:10123]



Matched response definition:
{
  "status" : 503
}

Response:
HTTP/1.1 503
Matched-Stub-Id: [415ccf84-5202-458e-8bd3-7aaaf2f47b34]
```

실제로 요청도 위와 같이 가짜 Mocking Server로 날라간 것을 확인할 수 있게 되죠.!  


## 참고 GitHub

관련 소스자료는 [여기서](https://github.com/huisam/kotlin-web/tree/main/src/main/kotlin/com/huisam/kotlinweb/fegin) 확인할 수 있습니다  
feign 모듈을 위주로 보면 좋을 것 같네요.!  