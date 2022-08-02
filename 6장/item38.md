
# [Item38] 확장할 수 있는 열거 타입이 필요하면 인터페이스를 사용하라

열거 타입은 거의 모든 상황에서 타입 안전 열거 패턴(typesafe enum pattern)보다 우수하다.  
단, 예외가 하나 있으니, 타입 안전 열거 패턴은 확장할 수 있으나 열거 타입은 그럴 수 없다는 점이다.  
타입 안전 열거 패턴은 열거한 값들을 그대로 가져온 다음 값을 더 추가하여 다른 목적으로 쓸 수 있는 반면, 열거 타입을 그렇게 할 수 없다는 뜻이다.

```java
// 타입 안전 열거 패턴(typesafe enum pattern)
public class DSymbolType{
  private final String type;
  private DSymbolType(String type){
    this.type = type;
  }
  public String toString(){
    return type;
  }
  public static final DSymbolType Terminal = new DSymbolType("Terminal");
  public static final DSymbolType Process = new DSymbolType("Process");
  public static final DSymbolType Decision = new DSymbolType("Decision");
}
```

**열거 타입을 확장하는 건 좋지 않은 생각이다**  
확장한 타입의 원소는 기반 타입의 원소로 취급하지만 그 반대는 성립하지 않는다면 이상하지 않은가  
기반 타입과 확장된 타입들의 원소 모두를 순회할 방법도 마땅치 않다.  
마지막으로 확장성을 높이려면 고려할 요소가 늘어나 설계와 구현이 더 복잡해진다.

그러나 열거 타입에서 확장할 수 있는 쓰임중 하나가 연산 코드 이다.


```java
// 코드 38-1 인터페이스를 이용해 확장 가능 열거 타입을 흉내 냈다. (232쪽)
public interface Operation {
    double apply(double x, double y);
}
```
```java
// 코드 38-1 인터페이스를 이용해 확장 가능 열거 타입을 흉내 냈다. - 기본 구현 (233쪽)
public enum BasicOperation implements Operation {
    PLUS("+") {
        public double apply(double x, double y) { return x + y; }
    },
    MINUS("-") {
        public double apply(double x, double y) { return x - y; }
    },
    TIMES("*") {
        public double apply(double x, double y) { return x * y; }
    },
    DIVIDE("/") {
        public double apply(double x, double y) { return x / y; }
    };

    private final String symbol;

    BasicOperation(String symbol) {
        this.symbol = symbol;
    }

    @Override public String toString() {
        return symbol;
    }
}
```

열거 타입인 BasicOperation은 확장할 수 없지만 인터페이스인 Operation은 확장할 수 있고, 이 인터페이스를 연산의 타입으로 사용하면 된다.  
이렇게 하면 Operation을 구현한 또 다른 열거 타입을 정의해 기본 타입인 BasicOperation을 대체할 수 있다.  

```java
// 코드 38-2 확장 가능 열거 타입 (233-235쪽)
public enum ExtendedOperation implements Operation {
    EXP("^") {
        public double apply(double x, double y) {
            return Math.pow(x, y);
        }
    },
    REMAINDER("%") {
        public double apply(double x, double y) {
            return x % y;
        }
    };
    private final String symbol;
    ExtendedOperation(String symbol) {
        this.symbol = symbol;
    }
    @Override public String toString() {
        return symbol;
    }

}
```

타입 수준에서도, 기본 열거 타입 대신 확장된 열거 타입을 넘겨 확장된 열거 타입의 원소 모두를 사용하게 할 수도 있다.

```java
    // 열거 타입의 Class 객체를 이용해 확장된 열거 타입의 모든 원소를 사용하는 예 (234쪽)
    public static void main(String[] args) {
        double x = Double.parseDouble(args[0]);
        double y = Double.parseDouble(args[1]);
        test(ExtendedOperation.class, x, y);
    }
    private static <T extends Enum<T> & Operation> void test(
            Class<T> opEnumType, double x, double y) {
        for (Operation op : opEnumType.getEnumConstants())
            System.out.printf("%f %s %f = %f%n",
                    x, op, y, op.apply(x, y));
    }
```

`T extends Enum<T> & Operation` Class 객체가 열거 타입인 동시에 Operation의 하위 타입이어야 한다는 뜻

```java
// 컬렉션 인스턴스를 이용해 확장된 열거 타입의 모든 원소를 사용하는 예 (235쪽)
public static void main(String[] args) {
    double x = Double.parseDouble(args[0]);
    double y = Double.parseDouble(args[1]);
    test(Arrays.asList(ExtendedOperation.values()), x, y);
}
private static void test(Collection<? extends Operation> opSet,
        double x, double y) {
    for (Operation op : opSet)
        System.out.printf("%f %s %f = %f%n",
                        x, op, y, op.apply(x, y));
}
```

Class 객체 대신 한정적 와일드카드 타입인 `Collection<? extends Operation>` 을 넘기는 방법도 존재  
덜 복잡하고 test 메서드가 살짝 더 유연해졌다.  
다시 말해 여러 구현 타입의 연산을 조합해 호출할 수 있게 되었다.  
반면, 특정 연산에서는 EnumSet과 EnumMap을 사용하지 못한다.

