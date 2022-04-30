#item 13 clone 재정의는 주의해서 진행하라

---

#### 서론 

- Cloneable은 복제해도 되는 클래스임을 명시하는 용도의 mixin interface지만, 아쉽게도 의도한 목적을 이루지 못함
    - clone 메서드가 선언된 곳이 Cloneable이 아닌 Object이고 메서드 접근자 마저 protected에 원인이 있음
    - 리플렉션을 이용하면 가능하지만 100% 성공을 보장할 수 없다.
    - 객체가 접근을 허용하는 clone 메서드를 제공한다는 보장이 없음
- 그럼에도 불구하고 Cloneable 방식은 널리 쓰이고 있음

---

#### Cloneable 인터페이스 형태
```
package java.lang;

/**
 * A class implements the {@code Cloneable} interface to
 * indicate to the {@link java.lang.Object#clone()} method that it
 * is legal for that method to make a
 * field-for-field copy of instances of that class.
 * <p>
 * Invoking Object's clone method on an instance that does not implement the
 * {@code Cloneable} interface results in the exception
 * {@code CloneNotSupportedException} being thrown.
 * <p>
 * By convention, classes that implement this interface should override
 * {@code Object.clone} (which is protected) with a public method.
 * See {@link java.lang.Object#clone()} for details on overriding this
 * method.
 * <p>
 * Note that this interface does <i>not</i> contain the {@code clone} method.
 * Therefore, it is not possible to clone an object merely by virtue of the
 * fact that it implements this interface.  Even if the clone method is invoked
 * reflectively, there is no guarantee that it will succeed.
 *
 * @author  unascribed
 * @see     java.lang.CloneNotSupportedException
 * @see     java.lang.Object#clone()
 * @since   1.0
 */
public interface Cloneable {
}

```
- 메서드 하나 없는 인터페이스
  - 메서드는 없지만 Cloneable구현한 클래스의 인스턴스에서 clone을 호출하면 객체의 필드를 하나하나 복사한 객체를 반환
    - 그렇지 않은 클래스의 인스턴스에서 호출하면 CloneNotSupportedException 을 던짐
- 인터페이스를 상당히 이례적으로 사용한 예이니 따라하지 말자.
  - 인터페이스를 구현한다는 것은 일반적으로 해당 클래스가 인터페이스에서 정의한 기능을 제공한다고 선언하는 행위
    - 그렇지만 Cloneable은 상위 클래스에 정의된 protected 메서드의 동작 방식을 변경
- 실무에서 Cloneable을 구현한 클래스는 clone 메서드를 pulbic으로 제공하고 사용자는 당연히 복제가 제대로 이루어지리라 기대
  - 기대를 만족하려면 클래스와 모든 상위 클래스는 복잡하고 강제할 수 없고 허술하게 기술된 프로토콜을 지켜야만 함
    - 깨지기 쉽고 위험하고 모순적 메커니즘이 탄생한다 : 생성자 호출하지 않고도 객체를 생성하는 메커니즘
  
---

#### clone 메서드 일반 규약
- 반드시 참
  - x.clone() != x 
- 반드시까지는 아닌 참
  - x.clone().getClass() == x.getClass()
  - x.clone().equals(x)
- 관례상 clone() 메서드가 반환하는 객체는 super.clone을 호출해 얻어야 한다. 
- 어떤 클래스 a와 a 클래스의 모든 상위 클래스(object를 제외한)가 아래 관례를 따른다면 x.clone().getClass() == x.getClass() 는 참

---

#### clone 에 관해
- 강제성이 없다는 점만 빼면 [생성자 연쇄](https://www.geeksforgeeks.org/constructor-chaining-java-examples/) (constructor chaining)과 살짝 비슷한 메커니즘
- clone 메서드가 super.clone이 아닌 생성자를 호출해 얻은 인스턴스를 반환해도 컴파일러는 통과함
- 하지만 이 클래스의 하위 클래스에서 super.clone을 호출한다면 잘못된 클래스의 객체가 만들어져 결국 하위 클래스의 clone 메서드가 동작하지 않게 됨
  ```
  @AllArgsConstructor
  @Getter
  public class Parent implements Cloneable {
      private int x;
      private int y;
  
      @Override
      public Parent clone() {
          return new Parent(this.x, this.y);
      }
  }
  
  void testClone() {
      Parent original = new Parent(2, 2);
      Parent clone = original.clone();
  
      assertNotSame(original, clone);
  
      assertEquals(original.getClass(), clone.getClass());
  
      assertEquals(original.getX(), clone.getX());
      assertEquals(original.getY(), clone.getY());
  }
  
  @Getter
  public class Child extends Parent{
      int childNumber;
  
      public Child(int x, int y, int childNumber) {
          super(x, y);
          this.childNumber = childNumber;
      }
  
      @Override
      public Child clone() {
          Child newChild = (Child) super.clone();
          return newChild;
      }
  }
  
  // java.lang.ClassCastException: class com.vsfe.sandbox.playground.effective.item14.Parent cannot be cast to class com.vsfe.sandbox.playground.effective.item14.Child
  ```
  [참고 글](https://github.com/dolly0920/Effective_Java_Study/issues/33)

- 클론을 재정의한 클래스가 final이라면 걱정해야 할 하위 클래스가 없으니 안전하다.
  - 그러나 final 클래스의 clone 메서드가 super.clone()을 호출하지 않는다면 굳이 Clonable을 구현할 필요도 없다.

- 모든 필드가 기본 타입이거나 불변 객체를 참조한다면 괜찮겠지만 쓸데없는 복사를 지양한다는 관점에서는 굳이 clone 메서드를 제공하지 않는 것이 좋다.

```
@Override
public PhoneNumber clone() {
  try {
    return (PhoneNumber) super.clone();
  } catch(CloneNotSupportedException e) {
    throw new AssertionError();
  }
}
```

- 공변 반환 타이핑을 이용해 (PhoneNumber) super.clone() 과 같은 형태로 하는 것이 가능하고 권장함
  - 재정의한 메서드의 반환 타입은 상위클래스의 메서드가 반환하면 타입의 하위 타입일 수 있음 (Parent)
- clone 메서드는 unchecked exception을 던질 수 있게 설계되었다.

---

#### 가변 객체 참조의 clone

- clone할 클래스가 가변 객체를 참조하는 순간 신경 써야 할 부분이 생김
  - 복사 후에도 원본과 같은 주소를 참조할 것임
  - 하나를 수정하면 둘 다 수정되어 불변식을 해침
- clone 메서드는 사실상 생성자와 같은 효과를 냄
  - 원본 객체에 아무런 해를 끼치지 않는 동시에 복제된 객체의 불변식을 보장해야 함
  - 제대로 동작하려면 내부 정보까지 복사하는 것
  
#### 가변 객체 참조 clone 방식

- 재귀적 호출
  ```
  public class Stack implements Cloneable {
    Obejct[] elements; 
    @Override
    public Stack clone() {
      try {
        Stack result = (Stack) super.clone();
        result.elements = elements.clone();
        return result;
      } catch (CloneNotSupportedException e) {
        throw new AsssertionError();
      }
    }
  }
  ```
  
  - 배열 elements.clone의 결과를 따로 형변환할 필요는 없음 배열 복제 시 원본 배열과 똑같은 배열을 반환하기 때문
    - clone 메서드를 사용하는 것을 권장한다 (배열은 clone 기느응ㄹ 제대로 사용하는 유일한 예)
    
  - elements 필드가 final 이었다면 clone이 되지 않았을 것이다. final 필드에는 새로운 값을 할당할 수 없기 때문
    - Cloneable 아키텍처의 '가변 객체를 참조하는 필드는 final로 선언하라' 의 일반 용법과 충돌
        - 원본과 복제된 객체가 그 가변 객체를 공유해도 안전하다면 괜찮음
  
  - 복제할 수 있는 클래스를 만들기 위해 final 을 제거해야 할 수도 있음

- 재귀적 호출로 부족한 경우
  ```
  public class HashTable implements Cloneable {
    private Entry[] buckets;
  
    private static class Entry {
      final Object key;
      Object value;
      Etnry next;
  
      Entry(Object key, Object value, Entry next) {
        this.key = key;
        this.value = value;
        this.next = next;
      } 
    }
  
    @Override
    public HashTable clone() {
      try {
        HashTable result = (HashTable) super.clone();
        result.buckets = buckets.clone();
        return result;
      } catch (CloneNotSupportedException e) {
        throw new AssertionError();
      }
    }
  }
  ```
  - 위의 경우 복제본이 자신만의 버킷을 갖지만 배열 갖지만 버킷 배열안의 Entry가 같은 곳을 참조하여 예기치 않게 동작할 가능성이 생김
  - 해결법
    1. 재귀 호출로 deepCopy를 이용
        - 재귀 호출 때문에 원소 수 만큼 스택 프레임을 소비
        - 리스트가 길면 스택 오버플로를 일으킬 위험이 있음
      ```
      public class HashTable implements Cloneable {
        private Entry[] buckets;
      
        private static class Entry {
          final Object key;
          Object value;
          Etnry next;
      
          Entry(Object key, Object value, Entry next) {
            this.key = key;
            this.value = value;
            this.next = next;
          } 
          
          // deepCopy 메서드 추가
          Entry deepCopy() {
            return new Entry(key, value, next == null ? null : next.deepCopy());
          }
        }
      
        @Override
        public HashTable clone() {
          try {
            HashTable result = (HashTable) super.clone();
            result.buckets = new Entry[buckets.length];
            // deepCopy 추가
            for (int i = 0; i < buckets.length; i++) {
              if (buckets[i] != null) {
                result.buckets[i] = buckets[i].deepCopy();
              }
            }
            return result;
          } catch (CloneNotSupportedException e) {
            throw new AssertionError();
          }
        }
      }
      ```
    2. 반복자 이용
      ```
      public class HashTable implements Cloneable {
        private Entry[] buckets;
      
        private static class Entry {
          final Object key;
          Object value;
          Etnry next;
      
          Entry(Object key, Object value, Entry next) {
            this.key = key;
            this.value = value;
            this.next = next;
          } 
          
          // deepCopy 반복자로 이용 
          Entry deepCopy() {
            Entry result = new Entry(key, value, next);
            for (Entry p = result; p.next != null; p = p.next) {
              p.next = new Entry(p.next.key, p.next.value, p.next.next);
            }
            return result;
          }
        }
      
        @Override
        public HashTable clone() {
          try {
            HashTable result = (HashTable) super.clone();
            result.buckets = new Entry[buckets.length];
            // deepCopy 추가
            for (int i = 0; i < buckets.length; i++) {
              if (buckets[i] != null) {
                result.buckets[i] = buckets[i].deepCopy();
              }
            }
            return result;
          } catch (CloneNotSupportedException e) {
            throw new AssertionError();
          }
        }
      }
      ```
    3. super.clone을 호출해 얻은 객체의 모든 필드를 초기 상태로 설정하고 원본 객체의 상태를 다시 생성하는 고수준 메서드를 호출
      - HashTable의 경우 모든 키-값에 대해 put(key, vaue) 메서드를 호출 해 둘의 내용이 똑같게 해주면 됨
        - 고수준 API를 활용해 복제하면 보통은 간단하고 제법 우아한 코드를 얻게 되지만 저수준에서 처리할 때보다는 느림
      - 필드 단위 객체 복사를 우회하기 때문에 Cloneable 아키텍처와는 어울리지 않는 방식
      - put 메서드는 final이거나 private이어야 한다 (private이라면 final이 아닌 public 메서드가 사용하는 도우미 메서드)
  
---

#### Cloneable 재정의

- 생성자에서는 재정의될 수 있는 메서드를 호출하지 않아야 하는데 clone 메서드도 마찬가지로 호출하지 않아야 함.
  - clone이 하위 클래스에서 재정의한 메서드를 호출하면 하위 클래스는 복제 과정에서 자신의 상태를 교정할 기회를 잃음
    - 원본과 복제본의 상태가 달라질 가능성이 큼
  - Object의 clone 메서드는 CloneNotSupportedException을 던진다고 선언했지만 재정의한 메서드는 아님
    - public 인 clone 메서드에서는 throws 절을 없애야 한다.
- 상속용 클래스는 Cloneable을 구현해서는 안 됨
  - [링크](https://stackoverflow.com/questions/47274777/java-why-class-designed-for-inheritance-should-contain-protected-clone-method)
- Cloneable을 구현하는 모든클래스는 clone을 재정의해야 한다.
  - 가장 먼저 super.clone을 호출한 후 필요한 필드를 적절히 수정한다.
  - 객체 내부의 숨어있는 가변 객체를 복사하고 복제본이 가진 객체 참조 모두가 복사된 객체들을 가리키게 함을 뜻함
  
---

#### 쓰레드 안전 Cloneable

- Cloneable을 구현한 스레드 안전 클래스를 작성할 때는 clone 메서드 역시 적절히 동기화 해줘야 한다.
  - Object의 clone메서드는 동기화를 신경쓰지 않았다.
  ```
  @Override
  public synchronized Object clone() {
    try {
      Object result = super.clone();
    } catch(CloneNotSupportedException e) {
    }
  }
  ```

---

#### 복사 생성자와 복사 팩터리 방식

- 모든 경우에 cloneable을 구현할 필요가 없고 이러한 경우 복사 생성자와 복사 팩터리 방식으로 객체 복사를 제공할 수 있다.

- 복사 생성자
```
public class Yum {
  int a;
  String b;
  public Yum(Yum yum) {
   this.a = yum.a;
   this.b = new String(yum.b); 
  }
}
```

- 복사 팩터리
```
public class Yum {
  int a;
  String b;
  public static Yum newInstance(Yum yum) {
    return new Yum(yum.a, new String(yum.b));
  }
}
```
---

#### 복사 생성자(변환 생성자) & 복사 팩터리(변환 팩터리)의 장점

- 생성자를 쓰는 방식을 사용하고 있음
- 엉성하게 문서화된 규약에 기대지 않을 수 있음
- 정상적인 final 필드 용법과도 충돌하지 않음
- 불필요한 검사 예외를 던지지도 않고 형변환도 필요하지 않음
- 해당 클래스가 구현하는 '인터페이스' 타입의 인스턴스를 인수로 받을 수 있음
- 원본 구현 타입에 얽매이지 않고 복제본의 타입을 직접 선택 가능
  - clone으로는 불가능 
```
public class A {
  public Set<String> set;
  
  public A (List<String> set) {
    this.set = new HashSet<>(set);
  }
}
```

---

#### 정리

- Cloneable은 새로운 인터페이스를 만들 때는 절대 Cloneable을 확장해서는 안되고 새로운 클래스도 이를 구현해서는 안 된다.
- final 클래스라면 위험이 크지 않지만 성능 최적화 관점에서 검토한 후 문제가 없을 경우에만 드물게 허용해야 한다.
- 기본 원칙은 복사 생성자와 복사 팩터리를 이용하는 식이 최고
  - 단, 배열만은 clone 메서드 방식이 가장 깔끔한 예외다.
  

