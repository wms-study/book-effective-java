
# 생성자에 매개변수가 많다면 빌더를 고려하라
> 생성자나 정적 팩터리가 처리해야 할 매개변수가 많다면 빌더 패턴을 선택하는 게 더 낫다.  
> 매개변수 중 다수가 필수가 아니거나 같은 타입이면 특히 더 그렇다.  
> 빌더는 점층적 생성자보다 클라이언트 코드를 읽고 쓰기가 훨씬 간결하고, 자바빈즈보다 훨씬 안전하다.

### 점층적 생성자 패턴(telescoping constructor pattern)을 통해 해결
그러나 사용자가 설정하고 싶지 않은 매개변수까지 포함할 경우가 있다.  
**점층적 생성자 패턴도 쓸 수 있지만, 매개변수가 많아지면 클라이언트 코드를 작성하거나 읽기 어렵다** 

### 자바빈즈 패턴(JavaBeans pattern)을 통해 해결
매개변수가 없는 생성자로 객체를 만든 후, setter를 사용하여 원하는 매개변수의 값을 설정  
**자바빈즈 패턴에는 객체 하나를 만들려면 메서드를 여러 개 호출해야 하고, 객체가 완전히 생성되지 전까지는 일관성(consistency)이 무너진 상태에 놓이게 된다**
**일관성이 무너지는 문제 때문에 자바빈즈 패턴에서는 클래스를 불변으로 만들 수 없으며 스레드 안전성을 얻으려면 추가 작업이 필요하다.**

## 빌더 패턴(Builder pattern)
클라이언트는 필요한 객체를 직접 만드는 대신, 필수 매개변수만으로 생성자를 호출해 빌더 객체를 얻는다.
빌더 패턴은 명명된 선택적 매개변수(named optional parameters)를 흉내 낸 것이다.

빌더 패턴은 계층적으로 설계된 클래스와 함께 쓰기에 좋다.  
각 계층 클래스에 관련 빌더를 맴버로 정의하자.
추상 클래스는 추상 빌더를, 구체 클래스(concrete class)는 구체 빌더를 갖게 한다.
```java
public abstract class Pizza {
    public enum Topping { HAM, MUSHROOM, ONION, PEPPER, SAUSAGE}
    final Set<Topping> toppings;
    
    abstract static class Builder<T extends Builder<T>> {
        EnumSet<Topping> toppings = EnumSet.noneOf(Topping.class);
        public T addTopping(Topping topping) {
            toppings.add(Objects.requireNonNull(topping));
            return self();
        }
        
        abstract Pizza build();
        
        protected abstract T self();
    }
    
    Pizza(Builder<?> builder) {
        toppings = builder.toppings.clone();
    }
}
```

```java
public class NyPizza extends Pizza {
    public enum Size {SMALL, MEDIUM, LARGE}
    private final Size size;
    
    public static class Builder extends Pizza.Builder<Builder> {
        private final Size size;
        
        public Builder(Size size) {
            this.size = Objects.requireNonNull(size);
        }
        
        @Override
        public NyPizza build() {
            return new NyPizza(this);
        }
        
        @Override
        protected Builder self() {
            return this;
        }
    }
    
    private NyPizza(Builder builder) {
        super(builder);
        size = builder.size;
    }
}
```

```java
public class Calzone extends Pizza {
    private final boolean sauceInside;
    
    public static class Builder extends Pizza.Builder<Builder> {
        private final boolean sauceInside = false;
        
        public Builder sauceInside() {
            sauceInside = true;
            return this;
        }
        
        @Override
        public Calzone build() {
            return new Calzone(this);
        }
        
        @Override
        protected Builder self() {
            return this;
        }
    }
    
    private Calzone(Builder builder) {
        super(builder);
        sauceInside = builder.sauceInside;
    }
}
```

```java
NyPizza pizza = new NyPizza.Builder(SMALL)
    .addTopping(SAUSAGE).addTopping(ONION).build();
Calzone calzone = new Calzone.Builder()
    .addTopping(HAM).sauceInside().build();
```
빌더를 이용하면 가변인수(varargs) 매개변수를 여러 개 사용할 수 있다.

### 빌더 패턴 단점
객체를 만들려면 그에 앞서 빌더부터 만들어야 한다.  
빌더 생성 비용이 크지는 않지만 성능에 민감한 상황에 문제가 될 수 있다.
