# @EqualsAndHashCode

@EqualsAndHashCode는 롬복 라이브러리에서 제공하는 어노테이션 중 하나로 자바 클래스에서 ```equals()```와 `hashcode()` 매서드를 생성하는데 사용된다.

무슨 말인지 글로는 정확히 이해가 힘들어 코드를 보며 간단한 예로 살펴보았다.

* @EqualsAndHashCode 사용 예

  ```java
  import lombok.EqualsAndHashCode;
  
  @EqualsAndHashCode
  public class User {
      private String name;
      private int age;
  
      // Constructor, getters, and setters
  }
  ```

위 코드는 사용자 정보를 표현하는 'User' 클래스가 있다고 상상하고, 클래에서는 사용자 이름('name')과 나이 ('age')를 저장하는 두 가지 필드가 존재한다.

이 클래스에 @EqualsAndHashCode 어노테이션을 추가하면 롬복을 컴파일 시 ```equals()``` 와 ```hashcode()``` 매서드를 생성해준다.

이 메서드들은 객체의 **동등성**을 비교할 때 사용된다.

## equal()

User 클래스를 사용해서 두 명의 사용자를 생성

```java
User user1 = new User("Alice", 30);
User user2 = new User("Bob", 25);

```

이 때 `equals()` 메서드를 통해 두 객체를 비교할 수 있다.

```java
boolean result = user1.equals(user2); // 결과는 false
```

equals() 메서드는 내부적으로 name과 age 필드를 비교하여 두 객체가 동일한지 확인!



## hashCode()

`hasCode()` 메서드를 사용하면 **'User'** 전체를 해시 맵의 키로 사용 가능

```java
Map<User, String> userMap = new HashMap<>();
userMap.put(user1, "User1");
userMap.put(user2, "User2");

String userName = userMap.get(user1); // 결과는 "User1"

```



