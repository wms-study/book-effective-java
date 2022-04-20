
# 생성자 대신 정적 팩터리 메서드를 고려하라
> 정적 팩터리 매서드와 public 생성자는 각자의 쓰임새가 있으니 상대적인 장단점을 이해하고 사용하는 것이 좋다.  
> 그렇다고 하더라도 정적 팩터리를 사용하는 게 유리한 경우가 더 많으므로 무작정 public 생성자를 제공하던 습관이 있다면 고치자

정적 팩터리 메서드(static factory method)
- 해당 클래스의 인스턴스를 반환하는 단순한 정적 메서드

```java
public static Boolean valueOf(boolean b) {
    return b ? Boolean.TRUE : Boolean.FALSE;
}
```
Boolean 클래스 팩터리 메서드 예시

## static factory method 제공시 장점

### 1. 이름을 가질 수 있다.
public 생성자에 넘기는 매개변수와 생성자 자체만으로 반환될 객체의 특성을 제대로 설명하지 못한다.  
static factory method의 경우 이름을 통해 반환될 객체의 특성을 묘사할 수 있다.

```
BigInteger(int, int, Random) <> BigInteger.probablePrime()
```
소수인 BigInteger를 반환할시 probablePrime을 통해 명확한 의미전달 가능

```java
public class Test {
    private Long id;
    private String name;
    private String subName;
    private Double price;
    private Double subPrice;
    
/*
    public Test(Long id, String name, Double price) {
        this.id = id;
        this.name = name;
        this.price = price;
    }
    
    public Test(Long id, Double subPrice, String subName) {
        this.id = id;
        this.subPrice = subPrice;
        this.subName = subName;
    }
*/

    public static Test mainTest(Long id, String name, Double price) {
        Test main = new Test();
        main.id = id;
        main.name = name;
        main.price = price;
        return main;
    }

    public static Test subTest(Long id, String subName, Double subPrice) {
        Test sub = new Test();
        sub.id = id;
        sub.subName = subName;
        sub.subPrice = subPrice;
        return sub;
    }
}
```
매개변수의 순서를 다르게 한 생성자를 통해서 다양한 생성자 방식을 만들 수 있지만 명확한 의미전달 불가  
생성자가 어떤 역할을 하는지 정확히 기억하기 어려워 엉뚱한 생성자를 호출하는 실수를 할 수 있다.

### 2. 호출될 때마다 인스턴스를 새로 생성하지 않아도 된다.
이 덕분에 불변 클래스를 통해 인스턴스를 캐싱하여 재활용, 불필요한 객체 생성을 피할 수 있다.

```java
public final class Boolean implements java.io.Serializable, Comparable<Boolean> {
    public static final Boolean TRUE = new Boolean(true);

    public static final Boolean FALSE = new Boolean(false);

    public static Boolean valueOf(boolean b) {
        return (b ? TRUE : FALSE);
    }

}
```
해당 메서드는 객체를 생성하지 않고 내부 static 인스턴스를 통해 캐싱하여 제공한다.  
cf) Flyweight pattern 

static factory method를 통해 인스턴스 통제(instance-controlled)가 가능해진다.
- 싱글턴(singleton)
- 인스턴스화 불가(noninstantiable)
- 불변 값 클래스에서 동치인 인스턴스가 단 하나뿐임을 보장 (a==b 일 때만 a.equals(b) 성립)


### 3. 반환 타입의 하위 타입 객체를 반환할 수 있는 능력이 있다.
구현 클래스를 공개하지 않고 객체를 반환할 수 있어 API를 작게 유지할 수 있다.

```java
public class Collections {
    // Suppresses default constructor, ensuring non-instantiability.
    private Collections() {
    }

    public static final List EMPTY_LIST = new EmptyList<>();

    public static final <T> List<T> emptyList() {
        return (List<T>) EMPTY_LIST;
    }
}
```
핵심 인터페이스들에 수정 불가나 동기화 등의 기능을 덧붙인 유틸리티 구현체를 제공하는  
인스턴스화 불가 클래스인 java.util.Collections에서 정적 팩터리 메서드를 통해 얻도록 한다.

```java
interface TestInterface {
    String TEST = "TEST";
    
    default Test getT2Instance() {
        Test t = new Test();
        t.name = TEST;
        return t;
    }
}
```
자바 8부터 인터페이스에 정적 메서드를 가질 수 있어 인스턴스 불가 동반 클래스를 둘 이유가 없다.  
동반 클래스에 두었던 정적멤버들 상상수를 그냥 인터페이스에 두면 된다.

### 4. 입력 매개변수에 따라 매번 다른 클래스의 객체를 반환할 수 있다.

```java
public abstract class EnumSet<E extends Enum<E>> extends AbstractSet<E>
    implements Cloneable, java.io.Serializable {
    
    public static <E extends Enum<E>> EnumSet<E> noneOf(Class<E> elementType) {
        Enum<?>[] universe = getUniverse(elementType);
        if (universe == null)
            throw new ClassCastException(elementType + " not an enum");

        if (universe.length <= 64)
            return new RegularEnumSet<>(elementType, universe);
        else
            return new JumboEnumSet<>(elementType, universe);
    }
    
    public static <E extends Enum<E>> EnumSet<E> of(E first, E... rest) {
        EnumSet<E> result = noneOf(first.getDeclaringClass());
        result.add(first);
        for (E e : rest)
            result.add(e);
        return result;
    }
}
```
EnumSet클래스는 public 생성자 없이 오직 정적 팩터리만 제공한다.  
원소가 64개 이하면 RegularEnumSet 인스턴스를, 65개 이상이면 JumboEnumSet 인스턴스를 반환한다.  
RegularEnumSet을 사용할 이점이 없어진다면 다음 릴리즈 때 이를 삭제해도 클라이언트 코드는 변경될 필요가 없다.

### 5. 정적 팩터리 메서드를 작성하는 시점에는 반환할 객체의 클래스가 존재하지 않아도 된다.
서비스 제공자 프레임워크 - 구현체들을 클라이언트에 제공하는 역할을 프레임워크가 통제하여, 클라이언트를 구현체로부터 분리해준다.
- JDBC
- Spring Container

서비스 제공자 프레임워크 3개의 핵심 컴포넌트로 이뤄진다.
- 서비스 인터페이스(service interface) - 구현체 동작을 정의
  - JDBC Connection
  - Interface를 통해 동작 행위 정의

- 제공자 등록 API(provider registration API) - 제공자가 구현체를 등록할때 사용하는 제공자 등록 API
  - DriverManager.registerDriver
  - @Bean 을 통해 구현체를 Spring Context에 등록

- 서비스 접근 API(service access API) - 클라이언트가 서비스의 인스턴스를 얻을때 사용하는 서비스 접근 API
  - DriverManager.getConnection
  - ApplicationContext.getBean()

- 서비스 제공자 인터페이스(service provider interface) - 서비스 인터페이스의 인스턴스를 생성하는 팩터리 객체를 설명
  - Driver
  - Interface 구현체

클라이언트는 서비스 접근 API를 사용할 때 원하는 구현체의 조건을 명시할 수 있다.  
조건을 명시하지 않으면 기본 구현체를 반환하거나 지원하는 구현체들을 하나씩 돌아가며 반환한다.  
이 서비스 접근 API가 서비스 제공자 프레임워크의 근간이며 '유연한 정적 팩터리'의 실체이다.

cf) 브릿지 패턴(Bridge pattern), 의존 객체 주입(Dependency injection, 의존성 주입), java.util.ServiceLoader

## static factory method 제공시 단점

### 1. 상속을 하려면 public이나 protected 생성자가 필요하니 static factory method만 제공하면 하위 클래스를 만들 수 없다.
구현 클래스들은 상속할 수 없다.

### 2. static factory method는 프로그래머가 찾기 어렵다.
생성자처럼 API 설명에 명확히 나타나지 않아 static factory method 방식 클래스를 인스턴스화할 방법을 알아내야 한다.

- API 문서화
- 메서드 이름 규약
  - from: 형변환
  - of: 여러 매개변수를 받아 집계
  - valueOf: from과 of의 더 자세한 버전
  - instance, getInstance: 싱글턴 인스턴스
  - create, newInstance: 새로운 인스턴스 반환
  - getType: getInstance와 같으나 다른 클래스의 factory method반환
  - newType: newInstance와 같으나 다른 클래스의 factory method반환
  - type: getType, newType의 간결한 버전 
