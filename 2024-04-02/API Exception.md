# API Exception

스프링으로 RestApi를 설계하던 도중 고민이 생겼다.

요청에 성공했을 때는 json으로 요청에 해당하는 데이터를 응답해주었는데 오류가 났을 때는 어떠한 값을 응답해줘야 하는지 정확히 감이 오지 않았다. 따라서 처음에는 

```ResponseEntity.status(HttpServletResponse.SC_CONFLICT).body("이미 존재하는 아이디입니다.")```

와 같이 코드를 작성하여 에러를 텍스트로 반환해주었다. 하지만 몇몇 응답을 만들면서 느껴졌던 부분은 응답에서 성공을 json으로 받았다면 아무리 한 줄 텍스트여도 에러 **역시 json으로** 받아야지 않을까? 하는 의문이 생겼다. 더 나아가 에러에 대한 상태 코드만 넘기면 되지 않나? 굳이 메시지를 담아주어야 하나? 라는 고민도 하게 되었다.

1. Exception도 json으로 응답해주어야 하는가?
2. Exception에도 message를 담아야 하는가? 그냥 에러 상태 코드만 응답하면 안되나?



일단 고민 끝에 내린 결론은 Exception의 상태 코드 외에도 message를 담아주어야 한다는 결론이 섰다.

그 이유는 다음과 같은 상황을 가정해보았기 때문이다.

1. front-end 개발자가 특정 페이지 경로에 해당하는 요청을 서버로 보냄
2. 서버는 해당 경로에 요청한 클라이언트가 로그인 했는지 여부를 검증
3. back-end 개발자는 로그인 하지 않은 사용자에게 401 Unauthorized 상태 코드를 반환
4. front-end 개발자는 해당 상태 코드를 보고 클라이언트가 로그인을 했지만 권한이 부족한 것인지 로그인을 해야만 볼 수 있는 경로인지 한 번에 알기 어려움

따라서 에러 메시지를 상태 코드와 함게 응답해주어야 Front-end 개발자가 조금 더 편리하게 개발을 진행할 수 있다고 생각하였다.

또한 이때 성공한 요청에 대한 응답(Json)과 실패한 요청에 대한 응답이 다르다면 front-end 개발자 입장에서는 두 가지 응답에 대해 조금이라도 다른, 부가적인 처리가 필요해지기에 `두 응답 모두 json으로 응답`해야 된다고 생각하였다.

### 해결 방법 1

> 모든 요청에 HashMap 생성하여 오류 담고 Objectmapper를 사용하여 json형태로 변경하여 반환

처음에는 Exception에 넣어줄 메시지를 Map에 넣은 후 Objectmapper를 통하여 json 형식으로 변경하여 반환하였다. 하지만 모든 Exception 응답에 계속해서 반복적으로 동일한 작업을 진행하는 것이 매우 비효율적이라는 생각이 얼마 지나지 않아 생기게 되었다.

* 동일한 작업 반복으로 인한 비효율

### 해결 방법 2

> ErrorMessageResponse 객체를 만들어 놓고 spring bean으로 등록하여 사용

모든 Exception에 대해 동일한 형태로 응답을 돌려주고, 요청에 따른 응답마다 새로운 객체를 계속해서 생성하지 않기 위해 미리 class를 선언하고 spring bean으로 등록하여 사용하는 것에 대해 고민하였다. 매우 순조롭게 진행되고 있었다고 생각했지만 한 가지 의문이 생겼다.

Servlet과 Spring에서 Exception에 대한 처리를 지원해주는데 굳이 이렇게 class를 만들고 객체를 만들어 직접 값을 넣어가며 사용하는 것이 과연 권장하는 방법일까?

스프링의 구조를 살펴보면 Exception이 발생했을 때 해당 exception을 계속해서 던지며 결국 was 까지 넘어가게 되며 이것을 다시 필터 -> 서블릿 -> 인터셉터 -> 컨트롤러(BasicErrorController) 까지 전달하여 spring이 만들어 놓은 Error를 돌려주게 되어있는데 이 구조를 잘 파악하면 상황에 따른 에러를 조금 더 편리하게 Custom하여 사용하며 체계적으로 사용할 수 있을 것 같다라는 생각이 들었다.

* spring Exception 처리 구조에 대해 정확한 이해 부족(비효율적 exception 처리 설계)



과연 스프링에서 권장하는 방식으로 해당 Exceptiond 처리를 구현하면 내가 구현한 것과 얼마나 많은 차이가 있을지 궁금해졌다.

### 해결 방법 3 #

> ExceptionResolver

Spring에서는 Exception이 발생하면 계속해서 Exception 위로 던지며 결국 was까지 도달하게 되고 이 exception 다시 컨트롤러로 돌아와서 처리하게 된다.

하지만 ExceptionResolver를 사용하면 위와 같이 너무 복잡하고 긴 과정을 줄일 수 있게 된다.

컨트롤러에서 에러가 발생하여 DispatcherServlet에 전달되면 DispatcherServlet은 이것을 ExceptionResolver로 넘기게 된다. 그러면 ExceptionResolver가 해당 에러를 해결 할 수 있다면 해당 에러를 해결하고 마치 정상 처리된 것처럼 다시 와스에 응답을 넘기게 된다. 이로써 Exception을 잡아 처리했지만 아까와 같이 다시 긴 과정을 거치는 작업이 사라지게 되었다. 

특히나 @ExceptionHandler 어노테이션을 사용하면 코드를 굉장히 간편하게 사용할 수 있다.

```java
 @ExceptionHandler(HttpClientErrorException.class)
    public ResponseEntity<ErrorMessageResponse> httpClientConflictErrorHandler(HttpClientErrorException e) {
        if (e.getStatusText() == "CONFLICT") {
            log.info("Coflict handler");
            ErrorMessageResponse errorMessageResponse = new ErrorMessageResponse(e.getStatusText(), "userName already exists");
            return ResponseEntity.status(HttpServletResponse.SC_CONFLICT).body(errorMessageResponse);
        }
```

위와 같이 에러를 던지고 해당 에러를 dispatcherServlet에서 ExceptionResolver를 호출해 처리해주면 이는 정상적인 응답으로 간주해 서블릿 컨테이너로 넘어가게 된다.

또한 ControllerAdvicef를 함께 사용하면 정상 응답과 Exception 처리를 나누어서 진행할 수 있다.

```java
@Slf4j
@RestControllerAdvice
public class ErrorControllerAdvice {

    @ExceptionHandler(HttpClientErrorException.class)
    public ResponseEntity<ErrorMessageResponse> httpClientConflictErrorHandler(HttpClientErrorException e) {
        if (e.getStatusText() == "CONFLICT") {
            log.info("Coflict handler");
            ErrorMessageResponse errorMessageResponse = new ErrorMessageResponse(e.getStatusText(), "userName already exists");
            return ResponseEntity.status(HttpServletResponse.SC_CONFLICT).body(errorMessageResponse);
        }

        if (e.getStatusText() == "UNAUTHORIZED") {
            log.info("Unauthorized handler");
            ErrorMessageResponse errorMessageResponse = new ErrorMessageResponse(e.getStatusText(), "Invalid userName or password");
            return ResponseEntity.status(HttpServletResponse.SC_UNAUTHORIZED).body(errorMessageResponse);
        }

        if (e.getStatusText() == "FORBIDDEN") {
            log.info("FORBIDDEN handler");
            ErrorMessageResponse errorMessageResponse = new ErrorMessageResponse(e.getStatusText(), "Login required");
            return ResponseEntity.status(HttpServletResponse.SC_FORBIDDEN).body(errorMessageResponse);
        }
        ErrorMessageResponse errorMessageResponse = new ErrorMessageResponse("Bad Request", "Invalid request");
        return ResponseEntity.status(HttpServletResponse.SC_BAD_REQUEST).body(errorMessageResponse);
    }
```



### 미해결

클라이언트 요청에 따른 에러를 생성할 때 다음과 같이 에러를 발생 시켜 주었다.

* 회원 가입 시 아이디 중복 에러 발생

  * ```java
    throw new HttpClientErrorException(HttpStatus.CONFLICT);
    ```

* 로그인 시 아이디 또는 비밀번호 에러 발생

  * ```java
    throw new HttpClientErrorException(HttpStatus.UNAUTHORIZED);
    ```

* 로그인 하지 않은 사용자의 특정 경로 접근 금지 에러 발생

  * ```java
    throw new HttpClientErrorException(HttpStatus.FORBIDDEN);
    ```



여기서 문제는 모든 클라이언트 요청에 따른 에러가 HttpClientErrorException으로 발생하기에 exception응답을 돌려줄 때 세부적인 코드를 반환해주기 어렵다는 문제가 발생했다.

ControllerAdivce를 사용할 때 @ResponseStatus 어노테이션을 통해 특정 에러 코드를 반환해 줄 수 있는데 그러기 위해서는 에러를 정확하게 잡아와서 처리해야 한다.

즉 HttpClientErrorException.Conflict.class를 정확하게 잡아와서 처리할 수 있어야 하는데 HttpClientErrorException(HttpStatus.CONFLICT), HttpClientErrorException(HttpStatus.UNAUTHORIZED), HttpClientErrorException(HttpStatus.FORBIDDEN) 모두 HttpClientErrorException.class 로 ExceptionHandler로 잡아와져서 각각 따로 처리하기 불편하였다. 결국 하나의 에러로 받고 에러에 담겨있는 statusText를 가져와서 조건문을 통해 해결하였는데 이게 괜찮은 방법인가 잘 모르겠다.

* exception 처리 코드

* ```java
  @Slf4j
  @RestControllerAdvice
  public class ErrorControllerAdvice {
      
      
      @ExceptionHandler(HttpClientErrorException.class)
      public ResponseEntity<ErrorMessageResponse> httpClientConflictErrorHandler(HttpClientErrorException e) {
          if (e.getStatusText() == "CONFLICT") {
              log.info("Coflict handler");
              ErrorMessageResponse errorMessageResponse = new ErrorMessageResponse(e.getStatusText(), "userName already exists");
              return ResponseEntity.status(HttpServletResponse.SC_CONFLICT).body(errorMessageResponse);
          }
  
          if (e.getStatusText() == "UNAUTHORIZED") {
              log.info("Unauthorized handler");
              ErrorMessageResponse errorMessageResponse = new ErrorMessageResponse(e.getStatusText(), "Invalid userName or password");
              return ResponseEntity.status(HttpServletResponse.SC_UNAUTHORIZED).body(errorMessageResponse);
          }
  
          if (e.getStatusText() == "FORBIDDEN") {
              log.info("FORBIDDEN handler");
              ErrorMessageResponse errorMessageResponse = new ErrorMessageResponse(e.getStatusText(), "Login required");
              return ResponseEntity.status(HttpServletResponse.SC_FORBIDDEN).body(errorMessageResponse);
          }
          ErrorMessageResponse errorMessageResponse = new ErrorMessageResponse("Bad Request", "Invalid request");
          return ResponseEntity.status(HttpServletResponse.SC_BAD_REQUEST).body(errorMessageResponse);
      }
  
  }
  
  ```

  