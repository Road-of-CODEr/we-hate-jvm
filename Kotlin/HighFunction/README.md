# 고차 함수

코틀린은 `Functional Programming` 에 특화된 언어니 한번 파헤쳐보자

## 목차
- [고차 함수](#고차-함수)
  - [목차](#목차)
  - [함수 정의하기](#함수-정의하기)
  - [함수를 리턴하는 함수](#함수를-리턴하는-함수)
  - [고차 함수 흐름제어](#고차-함수-흐름제어)


## 함수 정의하기

코틀린을 처음하다 보면 굉장히 어색한 문법이 있는데,  
```kotlin
list.filter { it % 2 == 0 }
```
와 같은 문법이다  

처음에는 정말 어색하지만, 두고 볼수록 정말 매력 있는 녀석이다  
자바에서는 
```java
list.stream()
    .filter(num -> num % 2 == 0)
```
와 같은 녀석인데, 코틀린에서 위에서 저렇게 사용하는건  
유감스럽게도 java `stream` 처럼 **lazy** 한 속성을 가지고 있지는 않다(일반적인 순차선회)  
kotlin의 stream을 쓰기 위해서는 `asSequnence` 를 선언해줘야 가능하다  

*아니 왜 함수 얘기하다가 스트림 이야기하고 있지*

어쨌든 다시 돌아와서
실제로 `.filter { }` 를 까보니

```kotlin
public inline fun <T> Iterable<T>.filter(predicate: (T) -> Boolean): List<T> {
    return filterTo(ArrayList<T>(), predicate)
}
```

많이 익숙한 것들이 보인다  
`(T) -> Boolean` 제네릭을 입력받아서 boolean을 리턴하는 람다함수다  

대충 느낌이 오시는가?  
함수를 인자로 넘기는 것이 가능하다는 것이다!  

```kotlin
fun twoAndThree(operation: (Int, Int) -> Int) {
    val result = operation(2, 3)
    println("The result is $result")
}
```
다음과 같은 함수가 있다 쳐보자  
twoAndThree는 함수를 인자로 받는 함수에요  
일명 이것이 바로 `고차함수`라는 개념!  

*어떻게 사용할까?*

```kotlin
twoAndThree { a, b -> a + b } // 5 출력
twoAndThree { a, b -> a - b } // -1 출력
```

그리고 코틀린은 `수신 객체 함수 정의` 가 가능하다!  
수신 객체 함수가 무엇이냐면, 클래스 외부에서 함수정의가 가능하다  

```kotlin
fun String.filter(predicate: (Char) -> Boolean): String {
    val sb = StringBuilder()
    (0 until length) // 0 ~ length - 1 까지
        .asSequence() // 자바 stream
        .map { get(it) } // character 가져오고
        .filter { predicate(it) } // 조건에 filter
        .forEach { sb.append(it) } 
    return sb.toString()
}

val result = "abef".filter { it in 'a'..'c' } // 결과는 ab
```

요런식으로 커스터마이징도 가능하다는 것이다 

## 함수를 리턴하는 함수

오잉? 정말 어려운 개념이다  
어떤 조건에 따라 취해야하는 행동이 다를 때 우리는 동작을 먼저 정의할 수 있다  

```kotlin
enum class Delivery {
    STANDARD, EXPEDITED
}

class Order(val itemCount: Int)

fun getShippingCostCalculator(
    delivery: Delivery,
): (Order) -> Double = when (delivery) {
    Delivery.EXPEDITED -> { order -> 6 + 2.1 * order.itemCount }
    Delivery.STANDARD -> { order -> 1.2 * order.itemCount }
}
```

간단한 예제이다  
`Delivery` , 배달 조건에 따라 계산을 다르게 하는 함수인데요  

```kotlin
val calculator = getShippingCostCalculator(Delivery.EXPEDITED)
println("Shipping cost ${calculator(Order(3))}") // 12.6 이 계산됨
```

코드를 보면 아시겠지만,
caculator의 객체 타입은 바로 `(Order) -> Double` 이라는 함수 타입이 된다!  

자, 조금 더 공부해보자

전략 패턴이라는 디자인 패턴을 아시는가?  

> 누군진 모르겠지만 정말 잘 정리해놓았다 [블로그 클릭](https://huisam.tistory.com/entry/Strategy)

전략 패턴은 실제 업무에서 제일 많이 쓰이는 디자인 패턴중의 하나이다  
( 프록시 패턴이 사실 제일 많지만 )  

한번 예시를 통해 어떻게 중복을 제거하면서 고차 함수로 멋있게 코딩할 수 있을까?

```kotlin
data class SiteVisit( // allArgs, get, equals, hashCode 오버라이딩 한 클래스
    val path: String,
    val duration: Double,
    val os: OS,
)

enum class OS { WINDOWS, MAC, IOS }

val log = listOf(
    SiteVisit("/", 34.0, OS.WINDOWS),
    SiteVisit("/", 22.0, OS.MAC),
    SiteVisit("/signUp", 1.0, OS.IOS),
    SiteVisit("/signUp", 10.0, OS.IOS),
    SiteVisit("/", 1.0, OS.IOS),
    SiteVisit("/login", 8.0, OS.WINDOWS),
)
```

우리는 이러한 접속정보에 대한 log 를 가지고 있다  
이때 다양한 요구사항이 있었어요

1. Window 사용자의 평균 사용시간을 가져와주세요
2. Mac 사용자의 평균 사용시간을 가져와주세요
3. IOS 사용자의 `/signUp` url에 대한 사용시간을 가져와주세요

필터링 하고 싶은 요구사항이 너무 많다는 것이다  
이걸 하나씩 다 일일이 아래처럼 코딩할 것인가?  

```kotlin
val averageWindowDuration = log
    .filter { it.os == OS.WINDOWS }
    .map(SiteVisit::duration)
    .average()
```

조금만 더 스마트하게 생각해볼까?  
우리는 함수에 함수를 넘길 수 있다는 것을 배웠다  

그럼 저 요구사항에 대한 공통함수를 추출해볼까?

```kotlin
fun List<SiteVisit>.averageDurationFor(predicate: (SiteVisit) -> Boolean): Double {
    return filter(predicate)
        .map(SiteVisit::duration)
        .average()
}
```

헉 너무나 간단해졌다  
동작에 대한 필터링 조건을 던짐으로써 더 이상 `map` 과 `average()`에 대한 중복 코드를 제거할 수 있게 되었다!  

```kotlin
println(log.averageDurationFor { it.os == OS.IOS && it.path == "/signUp" })
// (10.0 + 1.0) / 2 인 5.5가 출력
```

크 너무 멋있는 언어고, 코딩방식이다  
인터페이스를 정의하지 않아도 전략패턴은 유용하게 쓸 수 있다  

## 고차 함수 흐름제어

그럼 우리는 고차 함수에 대한 개념을 알게 되었는데,  
고차 함수에서 `return`을 하면 어떻게 될까?

```kotlin
fun main() {
    listOf("alice", "bob")
        .forEach {
            if (it == "alice") {
                println("Found!")
                return
            }
        }
    println("alice is Not Found")
}
```
결과는 어떻게 출력될까?  
`.forEach` 라는 고차함수에서 return이 될까?  
`main` 에서 return이 될까?  

정답은 바로 main이 return 이 된다  
위 코드를 실행하면 `Found!` 하고 끝나게 되버린다 ㅠㅠ  

이러한 return문이 바로 **non-local return** 이라고 부른다  

하지만 우리는 이러한걸 원한게 아니다  
`.forEach`에서 리턴하고 싶다면?

```kotlin
fun main() {
    listOf("alice", "bob")
        .forEach label@{ // 람다에 label을 붙일 수 있다
            if (it == "alice") {
                println("Found!")
                return@label // 해당 람다함수를 리턴
                // return@forEach label이 없다면 함수이름을 따라간다
            }
        }
    println("alice is Not Found")
}
```

우아하지 않은가? 요렇게 실행하게 되면  
`Found!` 와 `alice is Not Found` 가 출력되게 된다  

다른 기법도 존재하는데,  
바로 `무명함수` !!  

```kotlin
    listOf("alice", "bob")
        .forEach(fun(name) {
            if (name == "alice") return // 가장 가까운 함수인 fun(name) 을 리턴한다
            println("$name is not alice")
        })
```

무명함수를 사용하면 조금 더 return@test 와 같은 복잡한 기법을 피할 수 있다!!  
