# Declarative Http Client - Feign
* interface 선언을 통해 자동으로 Http Client 생성
* RestTemplate은 concreate 클래스라 테스트하기 어렵다
* 관심사의 분리
    * ```서비스의 관심``` - 다른 리소스, 외부 서비스 호출과 리턴값
    * 관심 X - 어떤 URL, 어떻게 파싱할까
* **spring cloud에서 open-feign 기반으로 wrapping한 것이 spring-cloud-feign**

## Spring Cloud Feign
* 인터페이스 선언만으로 Http Client 구현체를 만든다.
* ExternalFeignClient를 사용하고 싶은 곳에서 DI를 받아 사용하기만 하면 된다.

---

## Feign Client 적용
Home에서 External 호출 시 RestTemplate 대신 Feign Client 적용
```
compile('org.springframework.cloud:spring-cloud-starter-openfeign')
```
```java
@EnableFeignClients
@EnableEurekaClient
@EnableCircuitBreaker
@SpringBootApplication
public class HomeApplication {
    public static void main(String[] args) {
        SpringApplication.run(HomeApplication.class);
    }
}
```
* **각 메소드는 HystrixCommand로 감싸져 호출되고, 개별의 Circuit Breaker로 분리되며, 지정된 ThreadPool에서 실행된다.**
* **호출할 서버 주소를 eureka로부터 읽고, 서버 호출은 ribbon을 통해 이루어진다.**

### Feign + Ribbon + Eureka
```java
/**
 * url property를 명시하면 feign client은 hystrix, ribbon, eureka를 사용하지 않는다. (순수 Feign Client로 동작)
 * url property를 명시하지 않으면 마법같이 feign + ribbon + eureka로 동작한다.
 * - 즉, eureka에서 external 서버 목록을 조회한 다음 ribbon을 통해 LB를 하면서 HTTP 호출을 수행한다.
 */
//@FeignClient(name = "external", url = "http://localhost:8081")
@FeignClient(name = "external")
public interface ExternalFeignClient {
    
    @GetMapping("/any/{anyId}")
    String getAnyDetail(@PathVariable("anyId") String anyId);
    
}
```

### Feign + Hystrix + Ribbon + Eureka
```yaml
feign:
  hystrix:
    enabled: true
```
[Feign + Ribbon + Eureka](#Feign-+-Ribbon-+-Eureka)는 그대로 두고 yml에 위 설정만 추가하면 Feign Client의 메소드가 HystrixCommand로 감싸져 호출된다.

### Feign + Hystrix Fallback
```java
@Component
public class ExternalFeignClientFallbackFactory implements FallbackFactory<ExternalFeignClient> {
    
    @Override
    public ExternalFeignClient create(Throwable throwable) {
    
        return new ExternalFeignClient() {
            
            @Override
            public String getAnyDetail(String anyId) {
              log.warn("[External][getAnyDetail][Fail] message={}", throwable.getMessage());
              return "개발자가 정의할 무언가";
            }
            
        };
    }
}
```
```java
@FeignClient(name = "external", fallbackFactory = ExternalFeignClientFallbackFactory.class)
public interface ExternalFeignClient {
    
    @GetMapping("/any/{anyId}")
    String getAnyDetail(@PathVariable("anyId") String anyId);
    
}
```

### Custom HystrixCommand Config
```yaml
hystrix:
  command:
    default:
      execution:
        isolation:
          thread:
            timeoutInMilliseconds: 3000 # default 1,000ms
      circuitBreaker:
        requestVolumeThreshold: 10  # 서킷 브레이커의 상태 계산을 할 최소 요청 수 default: 20
        errorThresholdPercentage: 50  # 서킷을 열기 위한 에러 비율 default: 50
    ExternalFeignClient#getAnyDetail(String):
      execution:
        isolation:
          thread:
            timeoutInMilliseconds: 3000
```

---

### 요약
```Feign = 인터페이스 선언 + config 설정```으로 다음이 가능하다.
* Http Client
* Eureka에 등록된 서버 주소 조회
* Ribbon을 통한 Client Side Load Balancing
* Hystrix를 통한 메소드별 Circuit Breaker

---

### 예제 코드
* https://github.com/Spring-MicroServiceArchitecture/Home-api
* https://github.com/Spring-MicroServiceArchitecture/Product-api
