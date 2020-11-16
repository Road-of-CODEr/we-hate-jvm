# 복잡한 자바의 제네릭 (과연 Deep Dive 일까?)
ArrayList<T> 여기서 ```ArrayList```클래스를 제네릭이라고 하고, ```T```를 타입 파라미터라고 합니다.<br>
기본 개념은 간단한데 알고보면 매우 복잡한 녀석이라고 하네요...?

## 목차
* [서론](#서론)
* [제네릭 클래스](#제네릭-클래스)
* [제네릭 메소드](#제네릭-메소드)
* [타입 가변성과 와일드카드](#타입-가변성과-와일드카드)
* [타입 변수와 함께 사용하는 와일드 카드](#타입-변수와-함께-사용하는-와일드-카드)
* [와일드카드 캡쳐](#와일드카드-캡쳐)
* [JVM에서의 제네릭](#JVM에서의-제네릭)
    * [타입 소거](#타입-소거)
    * [타입 변환 연산자 삽입](#타입-변환-연산자-삽입)
* [제네릭의 제약](#제네릭의-제약)

## 서론
자바가 태어나고 한참 후인 ```JDK 1.5```에서야 제네릭이 추가되었습니다.<br>
당연하듯이 하위 버전도 컴파일이 가능해야 하기 때문에 하위 호환이 되도록 설계되었는데...<br><br>
일단 간단하게 제네릭을 알아가볼까요?

## 제네릭 클래스
```java
public class Entry<K, V> {
    private K key;
    private V value;

    public Entry(K key, V value) {
        this.key = key;
        this.value = value;
    }
    
    public K getKey() { return key; }
    public V getValue() { return value; }
}
```

## 제네릭 메소드
```java
public class Arrays {
    public static <T> void swap(T[] array, int i, int j) {
        T temp = array[i];
        array[i] = array[j];
        array[j] = temp;
    }
}
```
* 제네릭 메소드를 선언할 때, 타입 파리미터를 제어자(public || static)와 리턴 타입 사이에 명시합니다.
* 컴파일러는 메소드 파라미터와 리턴 타입에서 타입 파라미터를 유추할 수 있습니다.
    * 명시적으로 지정 가능 ex) Arrays.<String>swap(friends, 0, 1);
    
## 타입 가변성과 와일드카드
```java
public static void process(Employee[] staff) { ... }
```
Manager가 Employee의 서브 클래스라면 Manager[]를 위 메소드에 전달할 수 있고, 이것을 ```공변(covariance)```라고 합니다. 배열은 요소 타입과 같은 방식으로 변합니다.<br>
<br>
이번에는 ArrayList를 처리한다고 생각해봅시다! ArrayList<Manager>는 ArrayList<Employee>의 서브 타입이 아니죠.<br>
ArrayList<Manager>를 ArrayList<Employee> 타입 변수에 할당할 수 있게 하면 배열 리스트가 손상될 수 있겠죠?<br>
<br>
자바에서는 와일드카드로 메소드 파라미터와 리턴 타입이 변하는 방식을 지정합니다.
```java
public static void printNames(ArrayList<? extends Employee> staff) {
    for (int i = 0; i < staff.size(); ++i) {
        Employee employee = staff.get(i);
        System.out.println(employee.getName());
    }
}
```
```ArrayList<? extends Employee>```에 데이터 저장을 시도하면 어떻게 될까?<br>
* staff.add(val);<br>
add 메소드의 파라미터 타입은 ```? extends Employee```이므로 이 메소드에 전달할 수 있는 객체는 존재하지 않습니다.<br>
**```? extends Employee```를 Employee로 변환할 수는 있지만, 어떤 것도 ```? extends Employee```로는 변환될 수 없어요!**<br>
결론적으로 읽을 수는 있지만 쓸 수는 없다~

## 타입 변수와 함께 사용하는 와일드 카드
조건을 만족하는 element를 출력하는 메소드의 일반화 방법을 생각해볼까요?<br>
### 예시 1
```java
public static <T> void printAll(T[] elements, Predicate<T> filter) {
    for (T e : elements) {
        if (filter.test(e)) System.out.println(e.toString());
    }
}
```
printAll 메소드는 모든 타입의 배열에 동작하는 제네릭 메소드이고, 타입 파라미터는 전달 받는 배열의 타입이네요.<br>
하지만, Predicate의 타입 파라미터는 메소드의 타입 파라미터와 정확히 일치해야만 하는 제한이 있습니다.
해결방안은 다음과 같은데요.
```java
public static <T> void printAll(T[] elements, Predicate<? super T> filter) { ... }
```

### 예시 2
```java
public void addAll(Collection<? extends E> c) { ... }
```
위 메소드는 다른 컬렉션에 들어있는 모든 element를 추가할 수 있습니다. (element 타입이 E 또는 E의 서브 타입이어야 합니다.)

## 예시 3
```java
public static <T extends Comparable<? super T>> void sort(List<T> list) { ... }
```
위 sort 메소드는 T가 Comparable의 서브 타입이라면 어떤 List<T>든 정렬합니다. 하지만 Comparable 인터페이스도 제네릭이죠.
```java
public interface Comparable<T> {
    int compareTo(T other);
}
```
Comparable의 타입 파라미터는 compareTo 메소드의 인자 타입을 명시합니다. 따라서 위 sort 메소드를 다음과 같이 선언할 수도 있습니다.
```java
public static <T extends Comparable<T>> void sort(List<T> list) { ... }
```
하지만 위 방법은 매우 제한적입니다. Employee 클래스가 Comparable<Employee>를 구현하고, Manager 클래스가 Employee 클래스를 extends 한다고 가정했을 때, 
Manager 클래스는 Comparable<Manager>가 아닌 Comparable<Employee>를 구현합니다. 그러므로 Manager는 Comparable<Manager>의 서브 타입이 아니라 
Comparable<? super Manager>의 서브 타입이라는 사실...!

복잡하면서 어렵네요...ㄷㄷ

## 와일드카드 캡쳐
```java
public static void swap(ArrayList<?> elements, int i, int j) {
    ? temp = elements.get(i);   // 동작 X
    elements.set(i, elements.get(j));
    elements.set(j, temp);
}
```
```?```를 타입 인자로 사용할 수는 있지만 타입으로는 사용할 수 없습니다.<br>
아래와 같이 우회해서 해결할 수는 있대요.
```java
public static void swap(ArrayList<?> elements, int i, int j) {
    swapHelper(elements, i, j);
}

private static <T> void swapHelper(ArrayList<T> elements, int i, int j) {
    T temp = elements.get(i);
    elements.set(i, elements.get(j));
    elements.set(j, temp);
}
```
컴파일러는 ?가 뭔지 모르지만, ?는 타입을 의미하므로 제네릭 메소드를 호출할 수 있죠.<br>
swapHelper 메소드의 타입 파라미터 T는 와일드카드 타입을 ```캡쳐(capture)```합니다.<br>
이렇게 했을 때의 이점은 제네릭 메소드 대신 이해하기 쉬운 ArrayList<?>를 본다는 점?

## JVM에서의 제네릭
제네릭 설계자들은 클래스의 제네릭 형태가 기존 버전 클래스와 호환되게 하고 싶었다고 합니다.<br>
ArrayList를 받는 메소드가 있는데, 제네릭이 없던 시절에 만든 ArrayList는 Object타입으로 요소를 받습니다.<br>
근데 이 메소드에 ArrayList<String>을 전달할 수 있게 하고 싶었고,<br>
그래서 JVM에서 타입을 지우는 방식으로 기존 버전 클래스와 호환성을 유지했다고 하네요. 하지만 호환성 관점에서 만든 타협이 너무 자주 일어나 문제가 많이 남아있대요.

### 타입 소거
제네릭 타입을 정의하면 해당 타입은 raw 타입으로 컴파일됩니다.
```java
public class Entry {
    private Object key;
    private Object value;
    
    public Entry(Object key, Object value) {
        this.key = key;
        this.value = value;
    }
        
    public Object getKey() { return key; }
    public Object getValue() { return value; }
}
```
K, V가 모두 ```Object```로 교체됩니다.

```java
public class Entry<K extends Comparable<? super K> & Serializable, V extends Serializable> { ... }
```
타입 변수에 경계가 있으면 첫번째 경계로 교체됩니다.
```java
public class Entry {
    private Comparable key;
    private Serializable value;
}
```

### 타입 변환 연산자 삽입
타입 소거는 위험해보이지만 매우 안전합니다. Entry<String, Integer> 객체를 사용할 때, 이 객체를 생성하려면 반드시 String 타입 Key와 Integer 타입 value(Integer로 변환되는 타입 값)을 
전달해야 합니다. 그렇지 않으면 컴파일조차 할 수 없기 때문에 getKey 메소드에서 String을 반환한다는 것을 보장받습니다.<br>
<br>
하지만, 제네릭과 raw Entry 타입을 섞어서 사용하여 unchecked 경고 옵션으로 컴파일됐다면 Entry<String, Integer>에 서로 다른 타입으로 된 값이 포함될 수 있겠죠? 
따라서, 실행 시간에 안전성 검사를 해야 하는데... 컴파일러는 소거된 타입이 있는 표현식을 읽어올 때마다 타입 변환 연산자를 삽입합니다.
```java
Entry<String, Integer> entry = ...;
String key = entry.getKey();
```
타입이 소거된 getKey 메소드는 Object를 리턴하르모 컴파일러는 다음과 같은 코드를 만듭니다.
```java
String key = (String) entry.getKey();
```

## 제네릭의 제약
**실행 시간에는 모든 타입이 raw 형태다.**
* JVM에는 오직 raw type만 있다. 그래서 실행 시간에 ArrayList가 String을 담고 있는지 알 수 없다.

**파라미터화된 타입의 배열을 생성할 수 없다.**
```java
Entry<String, Integer>[] entries = new Entry<String, Integer>[100];
```
배열 생성자는 타입을 지우고 raw Entry 배열을 생성하므로 위 코드는 오류다.<br>
Entry<String, Integer>[] 타입은 규칙에 맞는 타입이다. 해당 변수를 초기화하려면 아래와 같이 하면 된다.
```java
@SuppressWarnings("unchecked")
Entry<String, Integer>[] entries = (Entry<String, Integer> new Entry<?, ?>[100];
```

하지만 ArrayList를 사용하는 것이 더 간단하다.
```java
List<Entry<String, Integer>> entries = new ArrayList<>(100);
```

가변 인자 파라미터는 사실 배열이다. 이런 파라미터가 제네릭이면 제네릭 배열 생성 제한을 우회할 수 있다.
```java
public static <T> ArrayList<T> asList(T... elements) {
    List<T> result = new ArrayList<>();
    for (T e : elements) result.add(e);
    return result;
}

Entry<String, Integer> entry1 = ...;
Entry<String, Integer> entry2 = ...;
List<Entry<String, Integer>> entries = Lists.asList(entry1, entry2);
```
T의 추론 타입이 Entry<String, Integer>라는 제네릭 타입이므로 elements는 Entry<String, Integer> 타입의 배열이다.<br>
하지만 이런 종류의 배열 생성은 개발자가 직접 할 수 없다.<br>
<br>
이 경우 컴파일러가 경고한다. 메소드가 파라미터 배열에서 element를 읽기만 한다면 ```@SafaVarargs``` 어노테이션으로 경고를 방지해야 한다.

**정적 컨텍스트에서는 클래스 타입 변수가 유효하지 않다.**
```java
public class Entry<K, V> {
    private static V defaultValue;
    public static void setDefaultValue(V value) { defaultValue = value; }
    // 오류: 정적 컨텍스트에서 V 사용
}
```

**제네릭 클래스의 객체는 예외로 던지거나 캐치할 수 없다.**
```java
public class Problem<T> extends Exception // 오류: 제네릭 클래스는 Throwable의 서브 타입이 될 수 없다.
```
```java
public static <T extends Throwable> void doWork(Runnable r, Class<T> cl) {
    try {
        r.run();
    } catch (T ex) { // 오류: 타입 변수는 catch 할 수 없다.
        ...
    }
}
```
```java
public static <V, T extends Throwable> V doWork(Callable<V> c, T ex) throws T { // throws 선언에는 타입 변수 사용 가능
    ...
}
```

## 끝맺음
알고보면 심오한 Generic... 
제네릭이 많으면 읽는 것도 힘든데 쓰는 것도 잘 써야하는구나를 느낄 수 있었던 것 같네요.
JDK 하위 호환도 흥미로운 점일 수 있겠군요.
