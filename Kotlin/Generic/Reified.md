# Kotlin - Reified
Reified는 Generics로 inline function에서 사용되며 런타임에 타입을 알아내려고 사용한다.
```kotlin
fun <T> doSomething(element: T) {
    ...
}
```
* 위 코드를 컴파일 할 때 컴파일러는 T가 어떤 타입인지 알고 있다.
* 하지만 컴파일하면서 타입을 제거하기 때문에 런타임에는 T가 어떤 타입인지 모른다..!
* Reified를 사용하면 런타임에 타입을 알 수 있지만, inline function과 함께 사용해야 한다.

```kotlin
inline fun <reified T> doSomething(element: T) {
    ...
}
```
```kotlin
inline fun <reified T> printDoSomething(element: T) {
    when (T::class) {
        String::class -> {
            println("Do String")
        }
        Int::class -> {
            println("Do Int")
        }
    }
}
```
위와 같이 쓰면 된다.<br>
함수 오버로딩도 할 수 있는데, reified를 사용하면 return 타입이 다른 함수를 오버로딩할 수 있다..!
```kotlin
inline fun <reified T> getMessage(number: Int): T {
    return when (T::class) {
        String::class -> "This is $number" as T
        Int::class -> number as T
        else -> "Nothing" as T
    }
}

fun main() {
    val result1 = getMessage<Int>(10)
    println(result1)
    val result2 = getMessage<String>(10)
    println(result2)
}

10
This is 10
```
