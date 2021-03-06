# Annotation

Kotlin에서의 `annotation` 은 java와 어떻게 다를까? 💭  

## 목차
- [Annotation](#annotation)
  - [목차](#목차)
  - [Annotation Target](#annotation-target)
  - [Java와의 호환성](#java와의-호환성)
    - [@JvmName](#jvmname)
    - [@JvmStatic](#jvmstatic)
    - [@JvmField](#jvmfield)
    - [@JvmOverloads](#jvmoverloads)
    - [@Throws](#throws)

## Annotation Target

기존에 있는 java와 많이 유사하지만,  
다른 점이 있다면.. `사용 지점 대상(use-site target)` 선언으로 annotation을 지정할 수 있습니다  

사용지점 대상과 해당하는 annotation 이름이 필요한 것이죠.!  

```kotlin
@get:Rule // 사용 지점 대상 : annotation 이름
```

사용 지점 대상에 대한 목록은 아래와 같아요

|대상|설명|
|--|--|
|property|프로퍼티 전체|
|field|프로퍼티에 의해 생성되는 필드|
|get|프로퍼티 getter|
|set|프로퍼티 setter|
|receiver|확장 함수나 프로퍼티의 수신 객체|
|param|생성자 파라미터|
|setparam|세터 파라미터|
|delegate|위임 프로퍼티의 위임 인스턴스를 담아둔 필드|
|file|파일 안에 선언된 최상위 함수와 프로퍼티를 담아둔 클래스|

대상이 정말 많죠.? 💁‍♂️  

그래서 어떻게 활용할 수 있냐구요.?  
보통은 `@validation` 하고, `@Json~` 시리즈에 거의 활용되고 있습니다  

```kotlin
data class Student(
    val name: String,
    val age: Int,
    @get:JsonIgnore
    val address: String,
)
```

이러한 객체가 있다고 가정하면,

```kotlin
fun main() {
    val objectMapper = ObjectMapper()
    val student = Student(
        name = "huisam",
        age = 29,
        address = "Seoul"
    )
    val jsonString = objectMapper.writeValueAsString(student)
    // jsonString은 {"name":"huisam","age":29}
}
```

기본적으로 java 에서의 custom annotation은 반드시 2개의 annotation이 필요한데요  

```java
@Target(AnnotationTarget.PROPERTY)
@Retention(Retention.RUNTIME)
public @interface JsonIgnore
```

어떤 타겟이고, 언제 정책을 결정한지가 필요한데  
`kotlin` 에서의 Retention은 default가 `RUNTIME` 입니다  

위의 annotation을 kotlin으로 변환하면?  

```kotlin
@Target(AnnotationTarget.PROPERTY)
annotation class JsonIgnore
```

처럼 되는거죠.!  

사실 annotation 은 java와 크게 다른 것은 없습니다 :)  
만일 파라미터로 인자값을 받고 싶다면.?  

```kotlin
annotation class Run (
    val name: String
)

@Run(name = "test")
```
요런식으로 활용할 수 있게 되는 것이죠.!

## Java와의 호환성

사실 annotation하고 큰 연관은 없는 주제이지만,  
기존 `java` 코드와의 호환성을 위한 kotlin의 annotation이 있습니다  

하나씩 살펴보죠.!  

### @JvmName

코틀린을 byte 코드로 변환할 때, JVM 시그니처를 변경할 때 사용합니다  

무슨 말인지 잘 모르겠죠?  
바로 예제로 가겠습니다

```kotlin
fun foo(a: List<String>) {
    println("a = ${a}")
}
// compile error
fun foo(a: List<Int>) {
    println("a = ${a}")
}
```

위 2개의 함수는 byte 코드로 변경될 때 시그니처가 동일합니다  
왜냐하면 List의 Generic은 구분되지 않기 때문이죠  

이를 개선해서 바꾸게 되면?

```kotlin
@JvmName("fooString")
fun foo(a: List<String>) {
    println("a = ${a}")
}
// compile error
fun foo(a: List<Int>) {
    println("a = ${a}")
}
```

해당 코드를 java code로 변환하게 되면은.!

```java
   public static final void stringFoo(@NotNull List a) {
      Intrinsics.checkNotNullParameter(a, "a");
      String var1 = "a = " + a;
      boolean var2 = false;
      System.out.println(var1);
   }

   public static final void foo(@NotNull List a) {
      Intrinsics.checkNotNullParameter(a, "a");
      String var1 = "a = " + a;
      boolean var2 = false;
      System.out.println(var1);
   }
```

메서드 시그니처가 변경되서 나타나는 모습을 볼 수 있어요.!  
다만 kotlin에서 사용할 때는 동일한 메서드 이름으로 사용할 수 있어요  
> kotlin 내부적으로 알아서 연결해 주기 때문이죠 

### @JvmStatic

이름에서도 볼 수 있듯이 jvm에서 static 변수 혹은 메서드로 가져가겠다는 것을 의미해요.!  

간단한 예시를 가져와볼께요

```kotlin
data class Phone(
    val name: String
) {
    companion object {
        @JvmStatic
        fun of(name: String): Phone = Phone(name = name)
    }
}
```

companion object 안에 `of` 라는 팩터리 메서드를 만들었는데요  
해당 `@JvmStatic`을 붙이게 되면 아래와 같이 변환되는 것을 볼 수 있어요  

```java
   @JvmStatic
   @NotNull
   public static final Phone of(@NotNull String name) {
      return Companion.of(name);
   }
```

만약, 해당 키워드가 존재하지 않는다면 위 java code는 생성되지 않습니다  


### @JvmField

```kotlin
class Student(
    var name: String,
)
```

kotlin에서는 `var` 라는 키워드가 있는데요  
해당 키워드를 붙이게 되면 class 내부에 자동적으로 get, set 함수를 만들어줍니다  

그래서 일반적으로 해당 코드를 javacode로 변환하면 아래처럼 되죠.!


```java
public final class Student {
   @NotNull
   private String name;

   @NotNull
   public final String getName() {
      return this.name;
   }

   public final void setName(@NotNull String var1) {
      Intrinsics.checkNotNullParameter(var1, "<set-?>");
      this.name = var1;
   }

   public Student(@NotNull String name) {
      Intrinsics.checkNotNullParameter(name, "name");
      super();
      this.name = name;
   }
}
```

하지만, @JvmField가 붙는다면?  
get / set 이 붙지않게 됩니다  

```kotlin
class Student(
    @JvmField
    var name: String,
)
```

위 코드는.! 아래처럼!!

```java
public final class Student {
   @JvmField
   @NotNull
   public String name;

   public Student(@NotNull String name) {
      Intrinsics.checkNotNullParameter(name, "name");
      super();
      this.name = name;
   }
}
```

싹 사라지게 되는것이죠 ㅎㅎ  
**java에서는 name에 대해서 get set 함수를 사용할 수 없게 됩니다**  

### @JvmOverloads

해당 annotation은 코틀린 함수의 `overloading` 메서드들을  
생성해주는 annotation 입니다  

바로 예제로 가볼까요.?  

```kotlin
class Student(
    val name: String = "abc",
    val age: Int = 123,
)
```

kotlin에서 `student` 클래스의 생성자는 총 3개입니다  
```kotlin
Student()
Student(name = "huisam")
Student(name = "huisam", age = 29)
```

하지만 해당 코드를 자바로 바꾸면.?  

```java
   public Student(@NotNull String name, int age) {
      Intrinsics.checkNotNullParameter(name, "name");
      super();
      this.name = name;
      this.age = age;
   }
```

아쉽게도 하나밖에 생성되지 않습니다 ㅠㅠ  
어떻게 하면 될까요.?  

```kotlin
class Student @JvmOverloads constructor(
    val name: String = "abc",
    val age: Int = 123,
)
```

생성자에 오버로딩가능하다고 명시하고,  
다시 변환하게 되면.?  

```java
   @JvmOverloads
   public Student(@NotNull String name) {
      this(name, 0, 2, (DefaultConstructorMarker)null);
   }

   @JvmOverloads
   public Student() {
      this((String)null, 0, 3, (DefaultConstructorMarker)null);
   }
```

잃어버렸단 나머지 2개의 생성자도 찾게 되었습니다 ㅎㅎ  

### @Throws

`kotlin` 에서는 `java` 에서와 같이 메서드에 `throws` 라는 개념이 존재하지 않습니다  
그럼 어떻게 하면 호환성을 맞춰줄 수 있을까요.?  

```kotlin
fun parseStringToInt(value: String): Int = value.toInt()
```

해당 메서드를 바로 java로 변환하게 되면.?  

```java
   public static final int parseStringToInt(@NotNull String value) {
      Intrinsics.checkNotNullParameter(value, "value");
      boolean var2 = false;
      return Integer.parseInt(value);
   }
```

보시는 것처럼 throws 키워드가 없죠..  
한번 추가해봅시다.!

```kotlin
@Throws(NumberFormatException::class)
fun parseStringToInt(value: String): Int = value.toInt()
```

이렇게 명시해주면..?

```java
   public static final int parseStringToInt(@NotNull String value) throws NumberFormatException {
      Intrinsics.checkNotNullParameter(value, "value");
      boolean var2 = false;
      return Integer.parseInt(value);
   }
```

와후.! 드디어 생겼네요 ^-^