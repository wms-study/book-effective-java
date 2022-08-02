
# [Item39] 명명 패턴보다 애너테이션을 사용하라

도구나 프레임워크가 특별히 다뤄야 할 요소에 구분되는 명명 패턴을 적용해왔다.  
ex) JUnit 버전 3까지 테스트 메서드 이름을 test로 시작하게끔했다.

### 명명 패턴 단점
1. 오타가 나면 안된다.
   - JUnit 버전 3버전에서 메서드 이름이 tsetSafety Override로 지으면 해당 테스트 무시
2. 올바른 프로그램 요소에서만 사용되리라 보증할 방법이 없다.
   - 메서드가 아닌 클래스 이름을 TestSafetyMechanisms로 지어 JUnit에게 줘도 테스트를 무시한다.
3. 프로그램 요소를 매개변수로 전달할 마땅한 방법이 없다는 것이다.
   - 특정 예외를 던져야 성공하는 테스트가 있다고 가정, 기대하는 예외 타입을 테스트에 매개변수로 전달해야함
   - 예외의 이름을 테스트 메서드 이름에 덧붙이는 방법도 있지만, 컴파일러는 메서드 이름에 덧붙인 문자열이 예외를 가리키는지 알 도리가 없다.

--- 

### 자동으로 수행되는 간단한 테스트, 예외 발생시 테스트를 실패로 처리

[Test (마커 애너테이션)](1markerannotation/Test.java)

[Sample](1markerannotation/Sample.java)

@Test 애너 테이션이 Sample 클래스의 의미에 직접적인 영향을 주지는 않는다.  
그저 이 애너테이션에 관심 있는 프로그램에게 추가 정보를 제공할 뿐이다.  
더 넓게 이야기 하면 대상 코드의 의미는 그대로 둔 채 그 애너테이션에 관심 있는 도구에서 특별한 처리를 할 기회를 준다.  

[RunTests](1markerannotation/RunTests.java)

테스트 메서드가 예외를 던지면 리플렉션 매커니즘이 InvocationTargetException으로 감싸서 다시 던진다.  
그래서 이 프로그램은 InvocationTargetException을 잡아 원래 예외에 담긴 실패 정보를 추출해(getCause) 출력한다.  
InvocationTargetException 외의 예외가 발생한다면 @Test 애너테이션을 잘못 사용했다는 뜻이다.  
아마도 인스턴스 메서드, 매개변수가 있는 메서드, 호출할 수 없는 메서드 등에 달았을 것이다.

RunTests 결과 메세지
```
Sample.m3() failed: RuntimeException: Boom
Invalid @Test: Sample.m5()
Sample.m7() failed: RuntimeException: Crash
성공: 1, 실패: 3
```

---

### 특정 예외를 던져야만 성공하는 테스트를 지원하도록 해보자.

[ExceptionTest](2annotationwithparameter/ExceptionTest.java)

이 애너테이션의 매개변수 타입은 `Class<? extends Throwable>`이다.  
여기서의 와일드카드 타입은 모든 예외(와 오류) 타입을 다 수용한다.  

[Sample2](2annotationwithparameter/Sample2.java)

[RunTests](2annotationwithparameter/RunTests.java)

---

### 예외를 여러개 명시하고 그중 하나가 발생하면 성공하게 만든다.

[ExceptionTest](3annotationwitharrayparameter/ExceptionTest.java)

[Sample3](3annotationwitharrayparameter/Sample3.java)

[RunTests](3annotationwitharrayparameter/RunTests.java)

---

### @Repeatable 매타애너테이션을 사용한 방식

[ExceptionTestContainer](4repeatableannotation/ExceptionTestContainer.java)
[ExceptionTest](4repeatableannotation/ExceptionTest.java)

배열 매개변수를 사용하는 대신 애너테이션에 @Repeatable 매타애너테이션을 달아 여러 개의 값을 받는 애너태이션을 만들 수 있다.  
@Repeatable을 단 애너테이션은 하나의 프로그램 요소에 여러 번 달 수 있다.  

#### @Repeatable 선언 주의점
1. @Repeatable을 단 애너테이션을 반환하는 '컨테이너 애너테이션'을 하나 더 정의하고, @Repeatable에 컨테이너 애너테이션의 class 객체를 매개변수로 전달해야 한다.
2. 컨테이너 애너테이션은 내부 애너테이션 타입의 배열을 반환하는 value 메서드를 정의해야 한다.
3. 컨테이너 애너테이션 타입에는 적절한 @Retention, @Target을 명시해야 한다.


[Sample4](4repeatableannotation/Sample4.java)

[RunTests](4repeatableannotation/)

#### @Repeatable 처리 주의점
1. @Repeatable에 해당하는 애너테이션을 여러 개 달면 하나만 달았을 때와 구분하기 위해 해당 '컨테이너' 애너테이션 타입이 적용된다.  
  getAnnotationsByType 메서드는 이 둘을 구분하지 않아서 반복 가능 애너테이션과 그 컨테이너 애너테이션을 모두 가져오지만,  
  isAnnotationPresent 메서드는 둘을 명확히 구분한다.  
  따라서 반복 가능 애너테이션을 여러 번 단 다음 isAnnotationPresent로 반복 가능 애너테이션이 달렸는지 검사한다면 "그렇지 않다"라고 알려준다.(컨테이너가 달렸기 때문이다)  
  그 결과 애너테이션을 여러 번 단 메서드들을 모두 무시하고 지나친다.
2. 같은 이유로, isAnnotationPresent로 컨테이너 애너테이션이 달렸는지 검사한다면 반복 가능 애너테이션을 한번만 단 메서드를 무시하고 지나친다.  
  그래서 달려 있는 수와 상관없이 모두 검사하려면 둘을 따로따로 확인해야 한다.  

---

애너테이션이 명명 패턴보다 낫다는 점을 보여준다.  
**애너테이션으로 할 수 있는 일을 명명 패턴으로 처리할 이유는 없다.**

