# AOP

## Aspect (관점)
관점을 정의한다. (뭘 할 것인지에 대한 추상적인 정의라고 생각하면 될 것 같다.)
> Advice + Pointcut

---

## Pointcut 정의
* 언제 Advice가 실행될 지를 정하는 것!
* &&, ||, ! 를 사용해서 포인트컷을 조합할 수 있다.

포인트컷 지정자 | 예시 | Joinpoint
-----------|------|-------
execution | execution(* set*(..)) | 이름이 set으로 시작하는 모든 메소드에서 실행
@annotation | @annotation(org.springframework.transaction.annotation.Transactional) | 실행 메소드가 @Transactional 어노테이션을 갖는 모든 결합점
bean | bean(sampleService) | sampleService Bean
!bean | !bean(sampleService) | sampleService Bean을 제외한 모든 Bean

종류는 더 많은데 전부 다 알아보진 않고 위 두 녀석들(유용할 것 같은)만 예제로 만들어놓았으니 예제를 한번 돌려보는게 가장 이해하기 쉽다.

---

## Advice 정의

### Before
* @Before
* 정의된 포인트컷 전에 수행
```kotlin
@Pointcut("execution(* com.example.aop.service.execution.KotlinService.print(..))")
private fun kotlinPrint() {
}

@Before("kotlinPrint()")
fun beforePrint() {
    logger.info("Before Kotlin Print")
}

// 출력
INFO 5571 --- [nio-8080-exec-1] com.example.aop.common.PrintAspect       : Before Kotlin Print
INFO 5571 --- [nio-8080-exec-1] c.e.aop.service.execution.KotlinService  : Kotlin
```

### After (finally)
* @After
* 정의된 포인트컷 후에 수행 (정상 종료와 예외 발생 케이스를 모두 처리해야 할 때 사용)
```kotlin
@Pointcut("execution(* com.example.aop.service.execution.JavaService.print(..))")
private fun javaPrint() {
}

@After("javaPrint()")
fun afterPrint() {
    logger.info("After Java Print")
}

INFO 5571 --- [nio-8080-exec-1] c.e.aop.service.execution.JavaService    : Java
INFO 5571 --- [nio-8080-exec-1] com.example.aop.common.PrintAspect       : After Java Print
```

### AfterThrowing
* @AfterThrowing
* 정의된 포인트컷에서 예외가 발생한 후 수행
* 포인트컷에서 발생된 예외는 exception 변수에 저장되어 전달

```kotlin
@Pointcut("execution(* com.example.aop.service.execution.KotlinService.exception(..))")
private fun kotlinException() {
}

@AfterThrowing(pointcut = "kotlinException()", throwing = "exception")
fun afterThrow(exception: IllegalArgumentException) {
    logger.info("After IllegalArgumentException ${exception.message}")
    throw IllegalArgumentException("AOP Throw")
}

INFO 5571 --- [nio-8080-exec-3] com.example.aop.common.PrintAspect       : After IllegalArgumentException 예외 발생
java.lang.IllegalArgumentException: AOP Throw
        at com.example.aop.common.PrintAspect.afterThrow(PrintAspect.kt:48) ~[main/:na]
    ...
```

### AfterReturning
* @AfterReturning
* 정상적으로 메소드가 실행될 때 수행
* 실행결과는 result 변수에 저장되어 전달

```kotlin
@Pointcut("execution(* com.example.aop.service.execution.KotlinService.getValue(..))")
fun kotlinGetValue() {
}

@AfterReturning(pointcut = "kotlinGetValue()", returning = "result")
fun afterReturn(result: String) {
    logger.info(result)
}

INFO 6728 --- [nio-8080-exec-1] com.example.aop.common.PrintAspect       : 코틀린 잘하고 싶다.
```

### Around
* @Around
* 메소드 수행 전후에 수행
* ProceedingJoinPoint를 파라미터로 전달하고 proceed() 메소드 호출로 대상 포인트컷을 실행한다.
* 포인트컷의 실행결과를 AOP 내부에서 변환하여 return 할 수 있다.

```kotlin
@Around("jsPrint()")
fun aroundPrint(pjp: ProceedingJoinPoint): String {
    logger.info("Around JS Print")
    val message = pjp.proceed()
    logger.info(message.toString())
    return "$message RESULT"
}

@Pointcut("@annotation(com.example.aop.service.annotate.JsCustom)")
private fun jsPrint() {
}
```

---

## Code
* https://github.com/J-minkuk/Spring-AOP-kt
