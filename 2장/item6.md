
# 불필요한 객체 생성을 피하라
똑같은 기능의 객체를 매번 생성하기보다는 객체 하나를 재사용하자
특히 불변 객체(아이템 17)는 언제든 재사용할 수 있다.

```java
    String s = new String("bikini");

    String newStringA1 = new String("a");
    String newStringA2 = new String("a");
    System.out.println(newStringA1 == newStringA2); // false
```
이 문장은 실행될 때마다 String 인스턴스를 새로 만든다.
이 문장이 반복문이나 빈번히 호출되는 메서드 안에 있다면 쓸데없는 String 인스턴스가 수백만 개 만들어진다.

```java
    String s = "bikini";

    String stringA1 = "a";
    String stringA2 = "a";
    System.out.println(stringA1 == stringA2); // true
```
이 코드는 새로운 인스턴스를 매번 만드는 대신 하나의 String 인스턴스를 사용한다.  
나아가 이 방식을 사용한다면 같은 가상 머신 안에서 이와 똑같은 문자열 리터럴을 사용하는 모든 코드가 같은 객체를 재사용함이 보장된다.
[[JLS, 3.10.5](https://docs.oracle.com/javase/specs/jls/se8/html/jls-3.html#jls-3.10.5)]

> Moreover, a string literal always refers to the same instance of class String. 
> This is because string literals - or, more generally, strings that are the values of constant expressions (§15.28) - are "interned" so as to share unique instances, using the method String.intern.

## 정적 팩터리 메서드 사용
생성자 대신 정적 팩터리 메서드(아이템 1)를 제공하는 불변 클래스에서는 정적 팩터리 메서드를 사용해 불필요한 객체 생성을 피할 수 있다.  
불변 객체만이 아니라 가변 객체라 해도 사용 중에 변경되지 않을 것임을 안다면 재사용할 수 있다.  
ex) Boolean(String) 생성자 대신 Boolean.valueOf(String) 팩토리 메서드를 사용하는 것이 좋다.  

## 객체 캐싱
생성 비용이 비싼 객체의 경우 캐싱하여 재사용하길 권한다.  

```java
public class RomanNumerals {
    static boolean isRomanNumeralSlow(String s) {
        return s.matches("^(?=.)M*(C[MD]|D?C{0,3})"
                + "(X[CL]|L?X{0,3})(I[XV]|V?I{0,3})$");
    }
}
```
Pattern 인스턴스는, 한 번 쓰고 버려져서 곧바로 가비지 컬렉션 대상이 된다.  
Pattern은 입력받은 정규표현식에 해당하는 유한 상태 머신(finite state machine)을 만들기 때문에 인스턴스 생성 비용이 높다.  

```java
public class RomanNumerals {
    private static final Pattern ROMAN = Pattern.compile(
            "^(?=.)M*(C[MD]|D?C{0,3})"
                    + "(X[CL]|L?X{0,3})(I[XV]|V?I{0,3})$");

    static boolean isRomanNumeralFast(String s) {
        return ROMAN.matcher(s).matches();
    }
}
```
성능을 개선하려면 필요한 정규표현식을 표현하는 (불변인) Pattern 인스턴스를 클래스 초기화(정적 초기화) 과정에서 직접 캐싱해두고 재사용

```java
public class RomanNumerals {
    public static void main(String[] args) {
        int numSets = Integer.parseInt(args[0]);
        int numReps = Integer.parseInt(args[1]);
        boolean b = false;

        for (int i = 0; i < numSets; i++) {
            long start = System.nanoTime();
            for (int j = 0; j < numReps; j++) {
                // 성능 차이를 확인하려면 xxxSlow 메서드를 xxxFast 메서드로 바꿔 실행해보자.
                b ^= isRomanNumeralSlow("MCMLXXVI");
            }
            long end = System.nanoTime();
            System.out.println(((end - start) / (1_000. * numReps)) + " μs.");
        }

        // VM이 최적화하지 못하게 막는 코드
        if (!b)
            System.out.println();
    }
}
```

## 오토 박싱(auto boxing)으로 인한 불필요 객체 생성
오토 박싱은 기본타입과 그에 대응하는 박싱된 기본 타입의 구분을 흐려주지만, 완전히 없애주는 것은 아니다.

```java
public class Sum {
    private static long sum() {
        Long sum = 0L;
        for (long i = 0; i <= Integer.MAX_VALUE; i++)
            sum += i;
        return sum;
    }
}
```
sum 변수를 Long으로 선언하여 long 타입인 i가 Long 타입인 sum에 더해질 때마다 불필요한 Long 인스턴스 생성  
**박싱된 기본 타입보다는 기본타입을 사용하고, 의도치 않은 오토박싱이 숨어들지 않도록 주의하자.**

- "객체 생성은 비싸니 피해야 한다"로 오해하지 말 것. 
  - JVM GC 성능이 좋아져 객체 생성, 회수의 부담이 적다.
  - 프로그램의 명확성, 간결성, 기능을 위해서 객체를 추가로 생성하는 것이라면 일반적으로 좋은 일
- 무거운 객체가 아닌 다음에야 단순히 객체 생성을 피하자고 커스텀 객체 풀(pool)을 만들지 말자.
  - 데이터베이스 커넥션의 경우 생성 비용이 워낙 비싸니 재사용하는 편이 낫다.
  - 하지만 일반적으로 자체 객체 풀은 코드를 헷갈리게 만들고 메모리 사용량을 늘리고 성능을 떨어뜨린다.
- 방어적 복사(defensive copy, 아이템 50)와 대조적
  - 이번 아이템이 "기존 객체를 재사용해야 한다면 새로운 객체를 만들지 마라"라면, 
  - 방어적 복사는 "새로운 객체를 만들어야 한다면 기존 객체를 재사용하지 마라"이다
  - 방어적 복사가 필요한 상황에서 객체를 재사용했을 때의 피해가, 필요 없는 객체를 반복 생성했을 때의 피해보다 훨씬 크다.
