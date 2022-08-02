
# [Item34] int 상수 대신 열거 타입을 사용하라

열거 타입은 일정 개수의 상수 값을 정의한 다음, 그 외의 값은 허용하지 않는 타입이다.

## 1. 정수 열거 패턴

```java
public class IntegerEnumPattern {
    public static final int APPLE_FUJI = 0;
    public static final int APPLE_PIPPIN = 1;
    public static final int APPLE_GRANNY_SMITH = 2;
    
    public static final int ORANGE_NAVEL = 0;
    public static final int ORANGE_TEMPLE = 1;
    public static final int ORANGE_BLOOD = 2;
}
```

#### 단점
- 타입 안전을 보장할 방법 없으며 표현력도 좋지 않다.
  - 오렌지를 건네야 할 메서드에 사과를 보내고 == 비교하더라고 컴파일러는 경고를 출력하지 않음
- 별도의 namespace를 지원하지 않아 접두어를 써서 이름 충돌을 방지한다.
- 정수 상수는 문자열로 출력하기 까다롭다.
- 같은 정수 열거 그룹에 속한 모든 상수를 한바퀴 순회하는 방법도 마땅치 않다. (상수가 몇개인지도 알 수 없다.)

## 2. 문자열 열거 패턴

```java
public class StringEnumPattern {
    public static final String APPLE_FUJI = "APPLE_FUJI";
    public static final String APPLE_PIPPIN = "APPLE_PIPPIN";
    public static final String APPLE_GRANNY_SMITH = "APPLE_GRANNY_SMITH";
}
```

#### 단점
- 정수 열거 패턴보다 더 나쁘다
- 상수의 의미를 출력할 수 있다는 점은 좋으나, 문자열 값을 그대로 하드코딩하게 만듦
- 하드코딩한 문자열 오타가 있을시 런타임 버그로 이어짐
- 문자열 비교에 따른 성능 저하

## 3. 열거 타입

[Enum.java](./Enum.java)

```java
public enum Apple {FUJI, PIPPIN, GRANNY_SMITH}
public enum Orange {NAVEL, TEMPLE, BLOOD}
```

#### 특성
- 열거 타입 자체는 클래스이며, 상수 하나당 자신의 인스턴스를 하나씩 만들어 public static final 필드로 공개한다.
- 열거 타입은 밖에서 접근할 수 있는 생성자를 제공하지 않으므로 사실상 final이다.
- 따라서 열거 타입 선언으로 만들어진 인스턴스들은 싱글턴이 보장된다.
- 거꾸로 열거 타입은 싱글턴을 일반화한 형태라고 볼 수 있다.

#### 장점
- 컴파일타임 타입 잔전성을 제공
  - 다른 타입의 값을 넘기려 하면 컴파일 오류
  - 타입이 다른 열거 타입 변수에 할당 불가
- 열거 타입에는 각자의 namespace가 있어서 이름이 같은 상수도 평화롭게 공존한다.
- 열거 타입에 새로운 상수를 추가하거나 순서를 바꿔도 다시 컴파일하지 않아도 된다.
  - 공개되는 것이 오직 필드의 이름뿐이라, 정수 열거 패턴과 달리 상수 값이 클라이언트로 컴파일되어 각인되지 않기 때문 ?
- 열거 타입의 toString 메서드는 출력하기에 적합한 문자열을 내어준다.
- 임의의 메서드나 필드를 추가할 수 있다.
- 인터페이스를 구현할 수 있다.
- Object 메서드들을 높은 품질로 구현해 놨다.
- Comparable과 Serializable을 구현했으며, 그 직렬화 형태도 웬만큼 변형을 가해도 문제없이 동작한다.


### 데이터와 메서드를 갖는 열거 타입

````java
// 코드 34-3 데이터와 메서드를 갖는 열거 타입 (211쪽)
public enum Planet {
    MERCURY(3.302e+23, 2.439e6),
    VENUS  (4.869e+24, 6.052e6),
    EARTH  (5.975e+24, 6.378e6),
    MARS   (6.419e+23, 3.393e6),
    JUPITER(1.899e+27, 7.149e7),
    SATURN (5.685e+26, 6.027e7),
    URANUS (8.683e+25, 2.556e7),
    NEPTUNE(1.024e+26, 2.477e7);

    private final double mass;           // 질량(단위: 킬로그램)
    private final double radius;         // 반지름(단위: 미터)
    private final double surfaceGravity; // 표면중력(단위: m / s^2)

    // 중력상수(단위: m^3 / kg s^2)
    private static final double G = 6.67300E-11;

    // 생성자
    Planet(double mass, double radius) {
        this.mass = mass;
        this.radius = radius;
        surfaceGravity = G * mass / (radius * radius);
    }

    public double mass()           { return mass; }
    public double radius()         { return radius; }
    public double surfaceGravity() { return surfaceGravity; }

    public double surfaceWeight(double mass) {
        return mass * surfaceGravity;  // F = ma
    }
}
````

- 열거 타입 상수 각각을 특정 데이터와 연결지으려면 생성자에서 데이터를 받아 인스턴스 필드에 저장하면 된다.
- 열거 타입은 근본적으로 불변이라 모든 필드는 final이어야 한다.
- 필드를 public으로 선언해도 되지만, private으로 두고 별도의 public 접근자 메서드를 두는게 낫다.

```java
// 어떤 객체의 지구에서의 무게를 입력받아 여덟 행성에서의 무게를 출력한다. (212쪽)
public class WeightTable {
   public static void main(String[] args) {
      double earthWeight = Double.parseDouble(args[0]);
      double mass = earthWeight / Planet.EARTH.surfaceGravity();
      for (Planet p : Planet.values())
         System.out.printf("%s에서의 무게는 %f이다.%n",
                           p, p.surfaceWeight(mass));
   }
}
```

- 열거 타입은 자신 안에 정의된 상수들의 값을 배열에 담아 반환하는 정적 메서드인 values를 제공한다.
- 각 열거 타입 값의 toString 메서드는 상수 이름을 문자열로 반환하므로 println과 printf로 출력하기에 안성맞춤이다.

### 상수마다 동작이 달라져야 하는 상황

```java
public enum Operation {
    PLUS,
    MINUS,
    TIMES,
    DIVIDE;

    public double apply(double x, double y) {
        switch (this) {
          case PLUS: return x + y;
          case MINUS: return x - y;
          case TIMES: return x * y;
          case DIVIDE: return x / y;
        }
        
        throw new AssertionError("알 수 없는 연산: " + this);
    };
}
```

#### switch문을 이용해 상수의 값에 따라 분기
- 마지막 throw 문은 실제로는 도달할 일이 없지만 기술적으로 도달 가능 따라서 생략하면 컴파일 오류
- 새로운 상수를 추가하면 해당 case 문 추가해야한다. 빼먹으면 위 AssertionError 런타임 오류 발생

```java
public enum Operation {
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

  Operation(String symbol) { this.symbol = symbol; }

  @Override public String toString() { return symbol; }

  public abstract double apply(double x, double y);


  public static void main(String[] args) {
    double x = Double.parseDouble(args[0]);
    double y = Double.parseDouble(args[1]);
    for (Operation op : Operation.values())
      System.out.printf("%f %s %f = %f%n",
              x, op, y, op.apply(x, y));
  }

  // 2.000000 + 4.000000 = 6.000000
  // 2.000000 - 4.000000 = -2.000000
  // 2.000000 * 4.000000 = 8.000000
  // 2.000000 / 4.000000 = 0.500000


  // 코드 34-7 열거 타입용 fromString 메서드 구현하기 (216쪽)
  private static final Map<String, Operation> stringToEnum =
          Stream.of(values()).collect(
                  toMap(Object::toString, e -> e));

  // 지정한 문자열에 해당하는 Operation을 (존재한다면) 반환한다.
  public static Optional<Operation> fromString(String symbol) {
    return Optional.ofNullable(stringToEnum.get(symbol));
  }

}
```

#### 상수별 메서드 구현 (constant-specific method implementation)
열거 타입에 추상 메서드를 선언하고 각 상수별 클래스 몸채(constant-specific class body) 재정의
- 상수 추가시 메서드를 필수로 재정의하게 함. 컴파일 오류 발생
- 상수별 메서드 구현을 상수별 데이터와 결합할 수도 있다.

```java
// 코드 34-9 전략 열거 타입 패턴 (218-219쪽)
enum PayrollDay {
    MONDAY, TUESDAY, WEDNESDAY, THURSDAY, FRIDAY,
    SATURDAY, SUNDAY;
    
    private static final int MINS_PER_SHIFT = 8 * 60;

    int pay(int minutesWorked, int payRate) {
        int basePay = minutesWorked * payRate;
        
        int overtimePay;
        switch (this) {
            case SATURDAY: case SUNDAY:
                overtimePay = basePay / 2;
                break;
            default:
                overtimePay = minutesWorked <= MINS_PER_SHIFT ?
                        0 : (minutesWorked - MINS_PER_SHIFT) * payRate / 2;
        }
        
        return basePay + overtimePay;
    }
}
```

#### 상수별 메서드 구현에는 상수끼리 코드를 공유하기 어렵다.

```java
// 코드 34-9 전략 열거 타입 패턴 (218-219쪽)
enum PayrollDay {
    MONDAY, TUESDAY, WEDNESDAY, THURSDAY, FRIDAY,
    SATURDAY(PayType.WEEKEND), SUNDAY(PayType.WEEKEND);

    private final PayType payType;

    PayrollDay(PayType payType) { this.payType = payType; }
    
    int pay(int minutesWorked, int payRate) {
        return payType.pay(minutesWorked, payRate);
    }

    // 전략 열거 타입
    enum PayType {
        WEEKDAY {
            int overtimePay(int minsWorked, int payRate) {
                return minsWorked <= MINS_PER_SHIFT ? 0 :
                        (minsWorked - MINS_PER_SHIFT) * payRate / 2;
            }
        },
        WEEKEND {
            int overtimePay(int minsWorked, int payRate) {
                return minsWorked * payRate / 2;
            }
        };

        abstract int overtimePay(int mins, int payRate);
        private static final int MINS_PER_SHIFT = 8 * 60;

        int pay(int minsWorked, int payRate) {
            int basePay = minsWorked * payRate;
            return basePay + overtimePay(minsWorked, payRate);
        }
    }

    public static void main(String[] args) {
        for (PayrollDay day : values())
            System.out.printf("%-10s%d%n", day, day.pay(8 * 60, 1));
    }
}
```

#### 전략 열거 타입 패턴
