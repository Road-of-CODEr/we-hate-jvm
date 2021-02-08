일단 자바 비동기도 모르니까 그것부터 알아보자.

### 스프링 비동기 개발 기본 지식

스프링캠프 2017[Day1 A2]: Async & Spring 발표 영상을 사실상 받아쓰기 함.

- 자바 비동기 개발
    - 비동기와 논 블로킹
    - 자바5+
        - Future/Executer(s)
        - BlockingQueue
    - 자바7
        - ForkJoinTask
    - 자바8
        - CompletableFuture
    - 자바9
        - Flow


- 서블릿 비동기 개발
  (4점대 까지의 스프링의 비동기는 서블릿 스택 위에서 동작함)
    - Servlet 3.0 Async Processing
        - AsyncContext
    - Servlet 3.1
        - Non-Blocking IO
    - Servlet 4.0


- 스프링 비동기 개발
    - @Async
    - Async Request Processing
        - Callable
        - DeferedResult
        - ResponseBodyEmitter
    - AsyncRestTemplate

### 동기? 비동기?

- Singleton -> 하나의 오브젝트
- Dependency Injection -> 세개의 오브젝트 (제3의 오브젝트가 두 오브젝트의 관계를 설정)
- Synchronous/Asynchronous -> 2개 내지는 그 이상

동기/비동기를 언급할 때는 - 무엇과 무엇이? - 어떤 시간을?

동기: A와 B라는 두가지가 함께 간다고 할 때 동시에 시작하거나 끝나거나 진행되면 동기

ex)

    1. 자바에서 CyclicBarrier -> A쓰레드, B쓰레드... 존재 할 때 N개의 쓰레드가 특정 지점 까지 도착 하는 순간 동시에 진행 
    2. method 호출 -> if 메소드 리턴시간(A) =  결과를 전달 받는 시간(B) -> 동기 메소드 호출
    3. A가 끝나는 시간 / B가 시작하는 시간이 같으면 동기 
            - synchronized
            - BlockingQueue

블로킹, 논블로킹

    - 동기 비동기와는 관점이 다름
    - 내가 직접 제어할 수 없는 대상을 상대하는 방법
    - 대상이 제한적임
        - IO
        - 멀티쓰레드 동기화

```java 
ExecutorService es = Executors.newCachedThreadPool(); // 쓰레드풀을 하나 만들어서
String res = es.submit(()->"HelloWorld").get(); // callable object로 비동기 하나 넘기고 결과를 가져옴
```

이것을 보고 동기/비동기를 이야기하면 오해하기 쉬움.

나눠서 생각해봐야함. es.submit()은 비동기 (메소드 리턴시간과 Callable의 실행 결과를 받는 시간이 일치하지 않음)

그러나 블록킹 논블록킹은 고려할 대상이 아님. 뭔가를 기다릴 대상이 없으므로 단순히 비동기 메소드 호출.

get()은 결과를 동기로 가져옴.(리천 시간과 결과를 가져오는 시간이 일치) / 비동기로 실행시킨 제3의 쓰레드에서 결과를 가져올때까지 블럭 상태

### 비동기 실행 @Async

```
@Async
void service(){
    ...//(B)
}

//(A)
myService.service()
//(C)
```

(B)와 (C)는 각각 다른 쓰레드에서 실행

```
@Async
String service(){
    ...
    return result;
}

String result=myService.service()
```

의 결과는? result에 null이 return됨.

일반 스트링 타입은 Async가 지원하지 않음.

@Async 메소드의 리턴 타입

- void
- Future<T>
- ListenableFuture<T>
- CompletableFuture<T>

<br>

```
@Async
Future<String> service(){
    ...
    return new AsyncResult<>(result);
}

Future<String> f=myService.service();
String res=f.get();
```
get으로 블로킹하여 전달 받음 

<br>

```
@Async
ListenableFuture<String> service(){
    ...
    return new AsyncResult<>(result);
}

ListenableFuture<String> f=myService.service();
f.addCallback(
r->log.info("Success:{}",r), e->log.info("Error: {}", e)};
```
논블러킹으로 결과를 받음


<br>

```
@Async
CompletableFuture<String> service(){
    ...
    return new AsyncResult<>(result);
}

CompletableFuture<String> f=myService.service();
f.thenAccept(r->System.out.println(r));
```
CompletableFuture안 에 CompletedFuture라는 비동기 작업을 완료후 완료값을 강제로 쓸수있는 static method가 있음.

*** 문제점 ***

SimpleAsyncTaskExecutor
    - @Async가 사옹하는 기본 TaskExecutor
    - 쓰레드 풀 아님
    - @Async를 본격적으로 사용한다면 실전에서 사용하지 말 것!

1. Async를 아무 설정 없이 사용하면 위 빈에서 새로운 쓰레드를 할당 받아 사용.
2. 쓰레드는 매우 비싼 자원이기 떄문에 풀을 만들어 재사용
3. 위 빈은 호출 할 때마다 새로운 쓰레드를 만들고 끝나면 버림.
4. 빨리 사라지지만 굉장히 낭비.

*** 해결법 ***

명시적으로 지정하지 않고 
Executor, ExecutorService, TaskExecutor 중 하나를 빈에 등록시키면 @Async가 그것을 사용

만약 2개이상을 등록한다면? 에러가 남 -> @Async("myExecutor")와 같이 명시적으로 적어주면 됨

<br>

```
@Bean
TaskExecutor taskExecutor(){
    ThreadPoolTaskExecutor te = new ThreadPoolTaskExecutor();
    te.setCorePoolSize(10);
    te.setMaxPoolSize(100);
    te.setQueueCapacity(50);
    te.initialize();
    return te;    
}
```

50개의 @Async 메소드 호출이 동시에 일어나면 쓰레드는 몇 개가 만들어질까?

정답은 10개

쓰레드풀은 코어 풀 사이즈를 오버하면 먼저 큐를 채우고 그리고 감당이 안되면 풀을 늘림. (db풀과 설정 방식 다름)


### 비동기 서블릿 (+3.0 ver)

- 블러킹으로 서블릿 요청 완료핮 못하는 경우를 위해 등장
- 서블릿에서 AsyncContext를 만든 뒤 즉시 서블릿 메소드 종료 및 서블릿 쓰레드 반납
- 어디서든 AsyncContext를 이용해 응담을 넣으면 클라이언트에 결과를 보냄

서블릿 코드가 복잡해지지만 사실 직접 쓸일은 없으니 넘어가자!

외에도 Non-blocking IO가 추가됨

### 비동기 스프링 @MVC 

Servlet의 비동기 요청 처리 기반의 @MVC

비동기 @MVC의 리턴 타입

- Callable<T>
- WebAsyncTask<T>
- DeferredResult<T>
- ListenableFuture<T>
- CompletionStage<T>
- ResponseBodyEmitter

사실 두가지 더있으나 어려워서 뺌

#### Callable<T>

AsyncTaskExecutor에서 실행될 코드를 리턴

```
@FunctionalInterface
public interface Callable<V>{
    V call() throws execption;
}

@GetMapping("/callable")
Callable<String> callable(){
    return ()->{
        return "hello"; //컨트롤러 메소드가 종료된 뒤 별도의 TaskExecutor 내에서 실행
    };
}
//Callable의 리턴 값이 컨트롤러 메소드의 리턴 값으로 사용
```


#### WebAsyncTask<T>

Callable에 없는 시간, 쓰레드 풀을 명시적으로 지정 할 수 있음
```java
public WebAsyncTask(Long timeout, String executorName, Callable<V> callable){...}
public WebAsyncTask(Long timeout, AsyncTaskExecutor executor, Callable<V> callable){...}
```

#### DeferredResult<T>

굉장히 유용함! 우리가 웹에 요청을 하고 응답을 만들어 내는데 뷰네임, 리스폰스바디 등의 정보를 넣어놓는 핸들러.

이 오브젝트를 컨트롤러 내부에서 만든 후 mvc 컨트롤러를 종료시킴.
그리고 DeferredResult는 어디선가, 주로 다른 쓰레드에서 그 값을 세팅하게 한 후
스프링은 MVC 코드를 바로 리턴 했으니 서블릿 쓰레드를 리턴함(가용 쓰레드 증가)
어디선가 DeferredResult는에 결과 값을 쓰면 원래 스프링 MVC의 리턴값이 었던것처럼 됨.

```
@GetMapping("/deferredresult")
    DeferredResult dr = new DeferredResult();
    queue.add(dr); // 1. DeferredResult 생성 보관한 뒤 바로 리턴(쓰레드 반납), but 응답은 지연
    return dr;
}

void eventHandler(String event){ // 2. 다른 쓰레드에서 저장된 DefferedResult에 결과 값 쓰기 
    queue.forEach(dr->dr.setResult(event));
}
```
ex) 100명 정도가 들어와서 어떤 이벤트가 들어오면 응답을 줘 라고 대기 상태에 있으면 100개의 쓰레드가 잡혀 있는게 아니라 반납하게 됨. 단지 결과를 쓸 수 있는 채널이 만들어져 있는 것.

<br>

```
@GetMapping("/drlf")
DeferredResult<String> drAndLf(){
    DeferredResult dr = new DeferredResult();
    
    ListenableFutre<String> lf = myService.async();
    lf.addCallback(r->dr.setResult(r),e->dr.setErrorResult(e));
    return dr;
}
```
DeferredResult와 @Async
- @Async메소드가 리턴하는 ListenableFuture에서 DeferredResult 사용
- 비동기 @MVC + 비동기 메소드 실행

```aidl
// 위의 코드와 동일한 작동
@GetMapping("/lf")
ListenableFuture<String> listenableFutre(){
    return myService.async();
}
```
4점대 이후에서는 리턴 타입이 ListenableFuture일 시 자동으로 DefferdResult 만들고 콜백 만드는 과정을 해줌.

<br>

ListenableFuture<T>의 한계
- 두가지 이상의 비동기 작업을 순차적으로 혹은 동시에 수행하고 결과를 조합해서 @MVC의 리턴 값으로 넘기려면?
```aidl
@GetMapping("/")
String listenableFutre(){
    String res1 = myService.async()
    return myService.sync2(res1); // 동기

@GetMapping("/")
ListenableFuture<String> listenableFutre(){
    ListenableFuture<String> res1 = myService.async();
    ListenableFuture<String> res2 = myService.async2(res1.???); //비동기 작업은 어떻게 ?? 
    return myService.async();
}

//해결책??
ListenableFuture<String> f1 = myService.async();
f1.addCallback(res2->{
    ListenableFuture<String> f2 = myService.async2(res1);
    f2.addCallback(res2->{
        dr.setResult(res2);
    },e->{
        dr.setErrorResult(e);
    });
    },e->{
        dr.setErrorResult(e);
    }
)

```
ListenableFuture<T> 조합
- 두개 이상의 비동기 작업을 결합할 때는 다시 콜백 + DeferredResult 방식으로
- 비동기 작업의 콜백에서 다음 비동기 작업 시도
- 최종 비동기 작업의 성공 콜백에서 DeferredResult에 결과 전달

-> 콜백의 중첩으로 코드가 복잡해짐, 예외 콜백의 내용이 동일한 경우 중복이 발생. 
-> 콜백헬! 

그래서 등장한 것이 자바8의 CompletableFuture<T> 조합

- 함수형 스타일 접근 방법
- CompletionStage(하나의 비동기 작업이 끝났을때 그 다음 동기 혹은 비동기 작업을 체이닝)의 서브타입.
- 중복되는 예외 처리 한번에
- 비동기 동기 작업의 변환 조합 결합 가능
- CompletableFuture/CompletionStage사용에 대한 학습이 필요
- ExecutorService의 활용 기법도 익혀야 함.


```
// 위 코드와 같은 작동
@GetMapping("")
CompletableFuture<String> cfCompose(){
    CompletableFuture<String> f1 = myService.casync();
    CompletableFuture<String> f2 = 
                           f1. thenCompose(res1-> myService.casync2(res1));
                           // 비동기 작업의 결과를 받아 이를 이용해 다음 비동기 작업을 수행하는 비동기 작업 생성                                                 
}

// 함수형으로 간결하게
@GetMapping("")
CompletableFuture<String> cfCompose(){
    CompletableFuture<String> f1 = myService.casync()
                                            .thenCompose(myService::casync2);
}
```
<br>

비동기 작업의 결합
- 2개 이상의 비동기 작업을 병렬적으로 실행하고 결과를 모아 결과 값을 만들어 내는 비동기 작업.
-> CompletableFuture와 combine을 이용해 가능
  

<br>

### 비동기 논블로킹 API 호출 (AsyncRestTemplate)

- RestTemplate는 동기-블로킹 방식
- 호출 작업 동안 쓰레드 점유
- 블로킹으로 인한 컨텍스트 스위칭 발생 (한번 블로킹마다 2번)

#### 문제
만약 100번의 요청을 비동기로 요청시 순간적으로 쓰레드 100개를 사용함.

AsyncRestTemplate
- 비동기이고 논블로킹(콜백)으로 결과를 돌려받지만 논블로킹IO를 사용하지는 않음.
- 이럴바에는 @Async에 RestTemplate쓰고 말지! 

-> 논블러킹 IO를 지원하는 리퀘스트 팩토리를 사용해야함. ex) Netty

#### 비동기 API 호출의 조합과 결합은 어떻게?

- AsyncRestTemplate은 ListenableFuture로만 리턴!
- 콜백 지옥~
- AsyncRestTemplate에는 CompletableFuture를 리턴하는 메소드가 없음.

```
public <T> CompletableFuture<T> toCFuture(ListenableFuture<T> lf){
    // cf는 특이하게 비동기 작업을 쓰레드 풀에서 실행하는 코드 없이 직접 이것은 비동기의 결과라고 만들어 낼 수 있음
    CompletableFuture<T> cf = new CompletableFuture<>(); 
    lf.addCallback((r)->{
        cf.complete(); 
    },(e) -> {
        cf.completeExceptionally(e);
    });
    return cf;
}
```
lf를 cf로 위와 같이 바꿀 수 있음!(java8 이상)


<br> 

#### TaskExecutor(쓰레드풀)의 전략적 활용이 중요! 
- 스프링의 모든 비동기 기술에는 ExecutorService의 세밀한 설정이 가능 
- CompletableFuture도 ExecutorService의 설계가 중요
- 코드를 보고 각 작업이 어떤 쓰레드에서 어떤 방식으로 동작하는지, 그게 어떤 효과와 장점이 있는지 설명할수 있어야 함.
  - 자바의 Concurrency library 이해도 높아야 함. 
- 벤치마킹 / 모니터링 중요! 

<br>

#### 비동기 기술을 사용하는 이유
- IO가 많은 서버 앱에서 서버 자원 효율적으로 사용해 성능 향상(낮은 레이턴시, 높은 처리율)
- 서버 외부의 이벤트를 받아 처이하는 것과 같은 비동기 작업이 필요해서
- ~~유행이라서 / 폼나 보여서~~

단점: 모르면 그냥 쓰지 말자, 디버깅 힘들고 복잡한데 혜택이 없을 수도!

추가로 볼만한 것
- HTTP Streaming
- ResponseBodyEmitter
- Async Listener
- Async Message Reception
- @Scheduled
- @MVC에 Rx, Reactor 사용하기 

