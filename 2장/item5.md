
# 자원을 직접 명시하지 말고 의존 객체 주입을 사용하라

> 클래스 내부적으로 하나 이상의 자원에 의존하고, 그 자원이 클래스 동작에 영향을 준다면 싱글텅과 정적 유틸리티 클래스는 사용하지 않는 것이 좋다.
> 이 자원들을 클래스가 직접 만들게 해서도 안된다.
> 대신 필요한 자원을 (혹은 그 자원을 만들어주는 팩터리를) 생성자에 (혹은 정적 팩터리나 빌더에) 넘겨주자.
> 의존 객체 주입이라 하는 이 기법은 클래스의 유연성, 재사용성, 케스트 용이성을 개선해준다.

많은 클래스가 하나 이상의 자원에 의존  
가령 맞춤법 검사기(SpellChecker)는 사전(dictionary)에 의존

### 정적 유틸리티를 잘못 사용한 예
```java
public class SpellChecker {
    private static final Lexicon dictionary = ...;
    
    private SpellChecker() {} // 객체 생성 방지
    
    public static boolean isValid(String word) {...}
    public static List<String> suggestion(String typo) {...}
}
```

### 싱글턴을 잘못 사용한 예
```java
public class SpellChecker {
    private static final Lexicon dictionary = ...;

    private SpellChecker(...) {}
    public static SpellChecker INSTANCE = new SpellChecker(...);

    public boolean isValid(String word) {...}
    public List<String> suggestion(String typo) {...}
}
```

두 방식 모두 사전을 단 하나만 사용한다고 가정한다는 점에서 좋지 않은 설계  
사전이 언어별로 따로 있고 특수 어휘용 사전을 별도로 두기도 한다.  
즉 유연하지 않고 테스트하기 어렵다.  
**사용하는 자원에 따라 동작이 달라지는 클래스에는 정적 유틸리티 클래스나 싱글턴 방식이 적합하지 않다.**

## 의존 객체 주입은 유연성과 테스트 용이성을 높여준다.
```java
public class SpellChecker {
private final Lexicon dictionary;

    private SpellChecker(Lexicon dictionary) {
        this.dictionary = Objects.requireNonNull(dictionary);
    }
    
    public boolean isValid(String word) {...}
    public List<String> suggestion(String typo) {...}
}
```
클래스(SpellChecker)가 여러 자원 인스턴스를 지원해야 하며, 클라이언트가 원하는 자원(dictionary)을 사용해야 한다.  
**인스턴스를 생성할 때 생성자에 필요한 자원을 넘겨주는 방식으로 이를 해결한다. - 의존 객체 주입 패턴**

- 불변(아이템 17)을 보장하여 (같은 자원을 사용하려는) 여러 클라이언트가 의존 객체들을 안심하고 공유할 수 있기도 하다.
- 생성자, 정적 팩터리(아이템1), 빌더(아이템2) 모두에 똑같이 응용 가능
 
## 팩터리 메서드 패턴
팩터리가 생성한 타일(Tile)들로 구성된 모자이크(Mosaic)를 만드는 메서드
```java
Mosaic create(Supplier<? extends Tile> tileFactory) { ... }
```
의존 객체 주입 패턴의 변형으로, 생성자에 자원 팩터리를 넘겨주는 방식이 있다.  
팩터리란 호출할 때마다 특정 타입의 인스턴스를 반복해서 만들어주는 객체  
즉 팩터리 메서드 패턴(Factory Method pattern)을 구현한 것  

의존성이 수천 개나 되는 큰 프로젝트에서는 코드를 어지럽게 만들기도 한다.
Dagger, Guice, Spring 같은 의존 객체 주입 프레임워크를 사용하면 이런 어질러짐을 해소 할 수 있다.

