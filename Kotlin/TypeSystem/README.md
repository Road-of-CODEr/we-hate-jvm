# Kotlin의 Type System

코틀린의 타입에 대해서 한번 알아보자.! 🖌️

## 목차
- [Kotlin의 Type System](#kotlin의-type-system)
  - [목차](#목차)
  - [Null](#null)
    - [체이닝 방식](#체이닝-방식)
    - [Null이 아니라고 단정짓기](#null이-아니라고-단정짓기)
    - [Null이 아닌 값을 함수에 넘기기](#null이-아닌-값을-함수에-넘기기)
  - [Casting](#casting)
  - [lazy loading](#lazy-loading)

## Null

보통은 가장 일반적으로 쓰는 방식이 바로 `?` 방식이다  
어떻게 사용하는 것이냐면  
```kotlin
foo?.toUpperCase()
```
의 의미는
```java
if (foo != null) {
    foo.toUpperCase()
} else {
    null
}
```
와 같은 의미를 가지게 된다!  

### 체이닝 방식

`java8` 에서 나오는 **Optional** 객체를 기억하시는가요?  
kotlin에서 `?` 문법은 쉽게 생각하면 Optional 객체를 요약한 것이라 볼 수 있다  

그렇기 때문에 chaining 기능도 쉽게 제공이 되는데요.  
```kotlin
data class Address(val streetAddress: String?, val zipCode: String))
data class Company(val address: Address?)
data class Person(val company: Company?)

fun Person.streetAddress(): String 
= this.company?.address?.streetAddress ?: "Unknown" // orElse 문법과 동일

fun main() {
    val person = Person(null)
    println(person.streetAddress()) // Unknown
}
```
요런식으로 편리하게 활용할 수 있습니다.!

만일 Address가 `nullable`한 객체지만, 비즈니스 로직에 NotNull 조건으로 활용하고 싶은 경우에는 아래와 같이도 활용할 수 있어요!  

```kotlin
fun Person.printZipCode() {
    val address = this.company?.address ?: throw IllegalStateException("No Address") // orElseThrow 와 동일
    print(address.zipCode)
}
```

### Null이 아니라고 단정짓기

어떤 경우에는 `null` 이 아니라고 단정지어야 하는 경우도 있어요!  
바로 `!!` 키워드인데요  
해당 객체가 만약 Null이 아니면 더 이상 null이 아니라고 컴파일러에게 단정짓고,  
null 이라면 그 자리에서 바로 `NPE` 가 발생하게 됩니다! ⚠️  
<br>
*그래서 정말 웬만한 경우에는 사용하지 않는게 좋겠죠?*

```kotlin
fun ignoreNulls(s: String?) {
    val sNotNull: String = s!!
    print(sNotNull.length)
}

ignoreNulls(null) // NPE 발생!!
```

### Null이 아닌 값을 함수에 넘기기

해당 키워드는 바로 `let` 인데요  
바로 코드부터 보는게 역시 이해가 빠릅니다 ㅎㅎ

먼저 `java`로 되어 있는 코드부터 볼까요?
```java
public void sendEmailTo(String email) {
    System.out.println("Sending Email to " + email);
}

Optional<String> email = Optional.ofNullable("yolo@example.com")
email.ifPresent(e -> sendEmailTo(e))
```

**email 이라는 객체가 null이 아니면 메일을 보내겠다** 라는 로직인데요  
파라미터를 계속 검사해야되는 불편함이 존재하는데  
이를 변환하면~?

```kotlin
fun sendEmailTo(email: String) {
    println("Sending Email to $email")
}

var email: String? = "yolo@example.com"
email?.let { sendEmailTo(it) }
```

정말 간단하게 적용할 수 있습니다!

## Casting

java에서 `instanceOf` 라는 키워드로 검사하고 Class를 casting하게 되는데요  
kotlin에서는 조금 더 편리하게 `as?` 라는 키워드로 할 수 있습니다!

한마디로 캐스팅에 성공하면 캐스팅한 객체를 돌려주고,  
실패하면 null을 받게 되는데요

```kotlin
interface Shape {
    fun name(): String
}

class Rectangle : Shape {
    override fun name(): String = "Rectangle"
}

class Triangle : Shape {
    override fun name(): String = "Triangle"
}

fun main() {
    val rectangle = Rectangle()

    val castResult = rectangle as? Triangle // casting fail
    println(castResult) // null
}
```

요렇게 되는 것이에요!  

`as` 키워드가 똑똑하게 바꾸는 걸 의미한다면,  
`is` 키워드는 `instanceOf` 와 유사합니다  

다만 판별이 들어가고 나서는 알아서 캐스팅되는 방식이죠 ㅎㅎ  

```kotlin
fun casting(value: Any) {
    when (value) {
        is String -> println("hi, $value")
        is Int -> println("hi, ${value * 2}")
        else -> return
    }
}

fun main() {
    casting("myName") // hi, myName 
    casting(2) // hi, 4
}
```

요렇게 안전하고 똑똑하게 캐스팅할 수 있습니다!

## lazy loading

때로는 지연 로딩(객체를 사용할 때 생성) 하고 싶은 경우가 있는데요  
`kotlin` 에서는 `lateinit` 이라는 키워드가 있습니다!  

**Spring** 을 많이 사용하시는 개발자라면 아마 자주 접해보셨을거라서  
지연 로딩에 대한 설명은 생략하고 어떻게 사용하는지에 대해서 알아보겠습니다!  

```kotlin
@Service
class MyService {
    fun performAction(): String = "foo"
}

@SpringBootTest
class MyServiceTest {
    
    @Autowired
    private lateinit var myService: MyService

    @Test
    fun testAction() {
        assertThat(myService.performAction()).isEqualTo("foo")
    }
}
```

지연 로딩이라는 특성떄문에  
1. 생성자 안에서 지연 로딩이면 `val`  
2. 생성자 밖에서 지연 로딩이면 `var`

라는 키워드로 해야됩니다!  

만일 초기화 되지 않은 경우에 접근하게 된다면?  
> lateinit property [ ] has not been initialized 

라는 `UninitializedPropertyAccessException`이 발생하게 됩니다!

