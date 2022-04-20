
# private 생성자나 열거 타입으로 싱글턴임을 보증하라
싱글턴(singleton) - 인스턴스를 오직 하나만 생성할 수 있는 클래스 

## public static final 필드 방식의 싱글턴
```java
public class Elvis {
    public static final Elvis INSTANCE = new Elvis();
    private Elvis() {}
    
    public void leaveTheBuilding() {}
}
```
장점
- 해당 클래스가 싱글턴임이 API에 명백히 드러난다.  
  public static 필드가 final이니 절대로 다른 객체를 참조할 수 없다.
- 간결하다

## static factory method 방식의 싱글턴
```java
public class Elvis {
    private static final Elvis INSTANCE = new Elvis();
    private Elvis() {}
    public static Elvis getInstance() {
        return INSTANCE;
    }

    public void leaveTheBuilding() {}
}
```
장점
- API를 바꾸지 않고도 싱글턴이 아니게 변경할 수 있다
- 정적 팩토리를 제네릭 싱글턴 팩터리로 만들 수 있다
- 정적 팩터리의 메서드 참조를 공급자(supplier)로 사용할 수 있다
  - Elvis::getInstance를 Supplier<Elvis>로 사용

## 싱글턴을 우회하는 문제
1. 리플렉션 API를 통해 `AccessibleObject.setAccessible(ture)`을 사용해 private 생성자를 호출
   - ??? 생성자를 수정하여 두번째 객체가 생성되려 할 때 예외를 던지게 한다.
   - Enum 클래스로 구현하여 reflection API 호출이 불가능하게 수정
     - Cannot reflectively create enum objects - 
     - 단점 - lazy loading이 불가, 애플리케이션이 올라올때 static하게 인스턴스화
```java
public class Main {
    public static void main(String[] args) {
        Constructor<Elvis> constructor = Elvis.class.getDeclaredConstructor();
        constructor.setAccessible(true);
        Elvis elvis = constructor.newInstance();
    }
}

public enum Elvis {
    INSTANCE;
    
    public void leaveTheBuilding() {}
}
```

2. 싱글턴 클래스 직렬화 후 역직렬화
   - 역직렬화에 사용하는 readResolve 메서드를 구현한다.
   - Enum 클래스로 구현하여 동일 객체 반환 
     - Enum은 java.lang.Enum 클래스를 상속 받고 있다
     - java.lang.Enum 클래스는 Serializable 이미 구현
     - JVM이 보장?? - enum이란 궁극적 목적 달성
   - Enum 상속 불가
   - 인터페이스 구현 가능
```java
public class Elvis {
    private static final Elvis INSTANCE = new Elvis();
    private Elvis() {}
    public static Elvis getInstance() {
        return INSTANCE;
    }

    public void leaveTheBuilding() {}
    
    protected Object readResolve() {
        return getInstance();
    }
}
```

