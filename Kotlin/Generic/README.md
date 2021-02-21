# Generic

java의 비슷하지만 아닌 거 같은 kotlin의 `generic`에 대해 알아보자 ❔

## 목차

- [Generic](#generic)
  - [목차](#목차)
  - [java와 비슷한 제네릭 타입](#java와-비슷한-제네릭-타입)
    - [nullable한 타입 파라미터?](#nullable한-타입-파라미터)
  - [java와 다른 제네릭 타입](#java와-다른-제네릭-타입)
    - [refied??](#refied)
  - [하위 타입](#하위-타입)
  - [추가정리](#추가정리)

## java와 비슷한 제네릭 타입

코틀린의 제네릭도 처음에는 자바와 많이 비슷해요
> 자바의 제네릭은 뭘까요.? [게시글](../../Generic/README.md) 참조

간단하게 배열을 짤라주는 `slice` 함수를 가져와보았어요

```kotlin
public fun <T> List<T>.slice(indices: IntRange): List<T> {
    if (indices.isEmpty()) return listOf()
    return this.subList(indices.start, indices.endInclusive + 1).toList()
}
```

정말 많이 비슷하죠?  
자바와 마찬가지로 제네릭 타입에 대한 상한 🌐 을 지정할 수 있어요!  

```kotlin
fun <T : Number> oneHalf(value: T): Double = value.toDouble() / 2.0
```

`Number` 타입의 제네릭을 상위에 두고, 하위에 있는 타입만 허용하겠다는 의미죠  
> 참고로 [Number](https://kotlinlang.org/docs/basic-types.html#numbers) 는 실수에 대한 최상위 클래스입니다.

```kotlin
oneHalf(32) // 정상
oneHalf("string") // 컴파일 에러
```

그렇기 때문에 다른 타입의 파라미터가 온다면 컴파일 에러가 발생하게 되요 ㅠㅠ 😢  

### nullable한 타입 파라미터?

java와는 다르게 코틀린은 기본 타입이 nullable인지 아닌지를 항상 먼저 생각해야 되요!  
그래서 generic도 마찬가지로 nullable한 코드를 지원하는데요  

```kotlin
class Processor<T> {
    fun process(value: T) {
        value?.hashCode()
    }
}

val processor = Processor<String?>()
processor.process(null) // 컴파일성공, 런타임시 에러가 발생하지 않음
```

value가 벌써 `nullable`하고 안전한 호출을 하고 있어서  
컴파일 에러도 없고, 런타임시 에러도 없어요 🚫  
<br>
*정말 이렇게 만들어도 될까요?*  
<br>
저는 개인적으로 조금 지양했으면 좋겠다는 생각이에요  
문제가 터졌을 때 원인을 알기가 힘들 뿐더러, `null-safe`를 보장하기 위해서는  
더 명시적으로 하는게 낫다고 생각합니다  

> 다만, 꼭 nullable 하게 쓰고 싶다면 써야죠 뭐... 어쩌겠습니까 하하

어떻게 하면 `nullable`을 막을 수 있을까요?  

```kotlin
class Processor<T : Any> {
    fun process(value: T) {
        value.hashCode()
    }
}
```

바로 java의 `Object` 타입과 똑같은 `Any` 타입을 명시하는 것입니다  
상한선을 두어서 이 제네릭 타입은 `null` 이 될 수 없다고  
컴파일러에게 알려주는 것이죠!  

```kotlin
val processor = Processor<String?>() // 컴파일 에러
// Type argument is not within its bounds.
// Expected:
// Any
// Found:
// String?

val safeProcessor = Processor<String>() // 성공
safeProcessor.process(null) // 컴파일 에러, Null can not be a value of a non-null type String
safeProcessor.process("abc") // 성공
```

요렇게 상한을 제시해서 안전한 코드를 만들 수 있습니다  

## java와 다른 제네릭 타입

음 저도 정말 어렵게 느껴지는 항목중의 하나였는데요  
우선 조금 이해하기 쉽게 예시로 시작해보도록 할게요  

```kotlin
val list1: List<String> = listOf("a", "b")
val list2: List<Int> = listOf(1, 2, 3)
```

2개의 서로 다른 제네릭 타입을 가지는 리스트가 있다고 해볼까요?  
이 `list`들은 각각의 리스트 안에 **문자로 선언되었는지 정수로 선언되었는지**  
런타임에 알 수 있는 방법이 없습니다  
> 실제로 get 해서 타입을 가져오기 전에는 말이죠  

여기까지는 모두 이해했죠.?  
그럼 이제 나아가서 `Collection` 의 입장에서 한번 저들을 바라볼까요?  

`list`는 `Collection`을 상속하고 있는 하위 클래스 입니다  
그러면 어떻게 하면 `Collection` 의 입장에서 `list` 인지 `map` 인지 `set` 인지  
판별할 수 있을까요?  

이를 해결해주는 것이 바로 `star projection` 입니다  

```kotlin
fun printSum(c: Collection<*>) {
    val intList = c as? List<Int> ?: throw IllegalArgumentException() // List<Int> 타입 검사후 캐스팅, 다만 unchecked cast 발생
    
    println(intList.sum())
}
```

java로 따지면 `?` 제네릭 같은 의미랑 비슷해요.!  
> 다만, 차이는 ? 키워드는 제네릭으로 쓸 수 있지만, * 키워드는 불가능해요

```kotlin
public inline fun <reified R, C : MutableCollection<in R>> Iterable<*>.filterIsInstanceTo(destination: C): C {
    for (element in this) if (element is R) destination.add(element) // for loop 돌면서 R 타입의 객체면 dest에 추가
    return destination // dest 리턴
}
```

대표적인 예시가 `filterIsInstacneTo` 함수인데요  
여러 개의 타입을 가진 `collection`이 존재할 때  
특정 타입으로 filtering 하고 걸러내는 함수에요  

### refied??

코드를 보다보니까 `refied` 라는 특이한 친구가 등장했어요  
저 친구는 도대체 뭐하는 녀석일까요.?  

뭐 참 어렵고 어려운 설명이 많지만, 쉽게 얘기하면 런타임에 `Class<>` 정보를   
얻어오기 위해서 `inline` 키워드와 같이 사용한답니다  
> 컴파일러는 **inline 함수의 본문을 구현한 바이트코드**를  
> 그 **함수가 호출되는 모든 지점** 에 삽입해요 `<- 핵심`    
> 그래서 일반 함수의 경우에는 제네릭의 정확한 타입을 알 수 없지만  
> inline 함수의 경우에는 추론할 수 있게 되는 것이죠

참 어렵습니다. 저도 몇 번을 돌이켜보고 이해했어요 🤔  

그래서 `refied` 라는 녀석은 제네릭 타입에 사용된 실제 타입을 알고  
만들어진 바이트코드를 **직접 클래스에 대응** 되도록 바꿔주는 역할을 맡게 되는거죠!  

*주의할 점은 인라인 함수는 자바에서 절대 사용할 수 없습니다*  

대표적인 예시가 바로 `objectMapper` 가 있어요  
```kotlin
inline fun <reified T : Any> String.toKotlinObject(): T {
  val mapper = jacksonObjectMapper()
  return mapper.readValue(this, T::class.java)
}
```

요렇게 인라인 함수를 만들고, json String을 kotlin객체로 변환하고 싶다면

```kotlin
json.toKolinObject<Car>()
```

요런식으로 간단하게 호출하면 된다는 거죠.!

## 하위 타입

위에서도 잠깐 짚고 넘어갔었지만, `kotlin` 에는 nullable 한 객체인 `?` 가 존재하죠  
그럼 어떻게 이에 대해 교통정리를 해야 될까요.?  
<br>
`nullable` 한 객체는 `non-null` 객체보다 무조건 상위타입입니다  
왜냐하면, `non-null`이 `nullable` 하지만  
`nullable`은 `non-null` 하지 않기 때문이죠  
( 마치 객체지향의 LSP 와 비슷한 ㅋㅋ 느낌이네요 )  

*이게 무슨 말장난 같은 소리냐...*  

간단합니다 코드 작성해보시죠 ㅎㅎ
```kotlin
val s: String = "abc"
val s2: String? = s // 정상 대입 가능
```

위 처럼 `non-null` 인 객체는 `nullable` 해질 수 있어요

```kotlin
val s2: String? = "abc"
val s: String = s2 // 컴파일 에러 발생, Type mis matched
```

반대로, `nullable`한 객체는 `non-null`이 될 수 없죠  
> 방법은 존재해요. 정말 null이 아니면 `!!` 를 쓰면 됩니다  
> 다만 이 방법은 결국 `nullable`을 `non-null`로 바꾸는 것이기에 ㅎㅎ  

그래서 이러한 하위 타입 관계들을 되게 어려운 용어로 설명한 것이 바로  
`공변성` 이라는 개념이에요  

저는 어려운 용어는 안쓰기로..ㅎㅎ  

## 추가정리

사실 많은 심오한 내용들이 더 있어요  
공변성과 관련이 있는 `in` `out` 키워드, star projection에 대한 고찰  
등이 있지만, 저는 여기까지 하려고합니다..ㅎㅎ  

어려운 것도 있고, 실제로 사용하기에는 유지보수 하기 쉽지 않을 것 같아서  
`1절` 에서 마무리하고, `2절`이 필요할 때 다시 정리하도록 할게요!

