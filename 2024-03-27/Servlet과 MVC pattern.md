# Servlet과 MVC pattern

스프링을 공부하던 도중  어느 순간부터 정해진 틀에 맞춰 코드를 작성하고 있다는 생각이 들었다. 왜, 어떤 이유로, 어떤 흐름으로 이렇게 코드가 작성되어야 하는지 보다는 그냥 이렇게 하면 되더라, 이런 상황은 이렇게 하더라처럼 나도 모르게 기계적으로 코드를 작성하게 된 것 같다.

따라서 스프링 개발을 더 잘하기 위해서는 클라이언트의 요청으로부터 스프링이 어떻게 응답을 돌려주는지 전체적인 과정과 흐름을 다시 한 번 상기시켜 보았다.

# Servlet

클라이언트에 - 서버 소통은 http 요청과 응답을 통해 이루어진다. 이때 http 요청에는 수많은 헤더값과 value가 들어있다.

![image-20240327104328460](assets/Servlet과 MVC pattern/image-20240327104328460.png)

이때 서버는 클라이언트의 요청 header 값을 읽어서 정확한 처리를 하고 요청을 다음과 같이 돌려줄 필요가 있다. 

![image-20240327104457791](assets/Servlet과 MVC pattern/image-20240327104457791.png)

하지만 서버 개발자가 직접 헤더에서 필요한 내용만을 추출하고 원하는 값을 response에 넣어 다시 해당 객체를 브라우저에 돌려주는 작업을 모두 직접한다면 굉장히 까다롭고 오래 걸릴 것이다.

Servlet은 개발자(서버)가 구현한 핵심 로직에만 집중할 수 있도록 그 외에 모든 처리를 대신 해주는 역할을 수행한다.

서버에서 네트워크 연결을 대기하며 소켓 연결을 구성하고 이를 통해 요청 받은 http 요청 메시지를 파싱해서 읽은 후 요청 방식과 Content-Type을 확인해 그에 맞게 Http 메시지 바디 내용을 파싱하여 특정 프로세스에 대한 비즈니스 로직을 실행 후 이에 대한 결과로 응답 메시지를 생성해 다시 브라우저에 돌려주는 모든 과정 중 `비즈니스 로직`을 제외한 모든 과정을 Servlet이 대신 수행해주게 된다.

요청 흐름은 다음과 같다

```
클라이언트 -> 요청 -> was에 요청 위임 -> was에서 Request, response 객체 생성 -> 서블릿 컨테이너에 있는 서블릿 객체에 우리가 만든 request, response 객체를 파라미터로 넘기며 실행 -> 서블릿 객체가 종료되면 값을 리턴 받아 Response 메시지 바디에 값을 채워넣고 클라이언트에 응답
```



![image-20240327111815589](assets/Servlet과 MVC pattern/image-20240327111815589.png)

만약 클라이언트의 요청이 "/Servlet 1"로 들어온다면 우리의 서버는 request, response 객체를 생성 후 Servlet 1 객체를 생성하면 request, response 객체를 파라미터로 넘겨주는 것이다. 이후 서블릿에서는 request, response 객체를 넘겨 받아 요청 사항에 맞는 비즈니스 로직을 수행하고 response의 메세지 바디에 내용을 담아 클라이언트에 응답하게 된다.

## 서블릿 객체 문제

하지만 위 구성에서 서블릿은 너무 많은 역할을 담당하고 있다. 예를 들어 사용자의 요청을 받아 필요한 로직을 처리하고 이를 사용자 뷰에 뿌려주는 것까지 모드 서블릿이 담당하게 되는 것이다.

### 서블릿 역할

* 요청 매핑
* http 파싱
* request에 따른 비즈니스 로직 수행
* response 메세지 바디에 내용 넣기
* 뷰 랜더링

너무 많은 역할을 서블릿에서 담당하고 있기에 향후 문제가 생겼을 때 유지 보수가 힘들 수 있고 코드에서 중복된 반복이 나타날 문제가 생길 수 있다.

따라서 우리는 서블릿의 역할을 나누어 줄 필요가 있다.

Model View Controller 라는 개념을 사용하여 하나의 서블릿에서 처리하던 것을 나누어 줄 수 있다.

* Controller : HTTP 요청을 받아 파라미터를 검증하고, 비즈니스 로직을 실행. 뷰에 전달할 결과 데이터를 조회해서 모델에 담아 준다.
  * 비즈니스 로직도 Controller에서 구현하기 보다는 Service라는 계층을 별도록 만들어서 처리하는 것이 바람직
  * Controller는 비즈니스 로직이 있는 서비스를 호출하는 역할을 담당
* Model : View에 출력할 데이터를 담아 둠. View에 필요한 데이터를 모두 모델에 담아서 전달해주기 때문에 View는 비즈니스 로직이나 데이터 접근을 몰라도 되고 랜더링에만 집중할 수 있음.
* View : Model에 담겨있는 데이터를 사용해서 화면을 그리는 일에 집중. (HTML)

![image-20240327134057106](assets/Servlet과 MVC pattern/image-20240327134057106.png)

![image-20240327134104620](assets/Servlet과 MVC pattern/image-20240327134104620.png)

## 서블릿 중복의 문제(Controller 중복)

위와 같은 구성을 통해 서블릿 객체는 Controller, Model, View 세 가지로 분리되었다. 이로써 각각의 영영마다 역할을 분명히 나누어 향후 유지 보수 등에서 많은 이점을 갖게 되었다. 그렇다면 현재의 구성에서 다른 문제는 없을까? 아직도 다음과 같은 문제가 나타날 수 있다.

### MVC 컨트롤러의 문제

* 공통 처리가 어려움

기능이 많아질 수록 컨트롤러와 컨트롤러에서 처리해야하는 공통 부분이 점점 더 많이 증가한다. 이때 공통 부분을 계속해서 컨트롤러에서 직접 호출해야 하며 수정할 때는 모든 컨트롤러에서 직접 수정해줘야 하는 번거로움이 존재한다. 

예들들어 우리는 요청에 따른 모든 서블릿 객체를 생성해야 하며 서블릿 객체를 호출하고 값을 돌려 받는 중복된 과정을 계속 실행해야 된다는 것이다.

이러한 문제를 해결하기 위한 방법은 무엇일까? 간단하다. 서블릿 처리를 딱 한 번만 해주면 되는 것이다.

## FrontController

모든 서블릿 객체에 대한 요청을 앞에서 한 번에 받아 처리해준다면 위에서 말한 문제는 사라진다.

![image-20240327113428972](assets/Servlet과 MVC pattern/image-20240327113428972.png)

위와 같이 앞단에서 먼저 요청을 처리하고 그에 해당하는 비즈니스 로직만 호출하여 그때그때 처리를 해준다면 위에서 말한 모든 문제가 해결된다.

프론트 컨트롤러에서 서블릿 하나로 클라이언트의 요청을 받고 프론트 컨트롤러가 요청에 맞는 컨트롤러를 찾아서 호출한다. 이후 나머지 컨트롤러는 서블릿을 사용하지 않아도 된다.

이렇게 구현하기 위해서는 서블릿과 비슷한 모양의 컨트롤러 인터페이스를 만들고 이를 각 컨트롤러에서 구현하여 사용하면 된다. 그러면 프론트 컨트롤러는 이 인터페이스만 호출하고 요청에 맞춰 필요한 컨트롤러가 호출되는 방식이다.

### FrontController V1 - frontController 만들기

위에서 구성한 구조를 점진적으로 발전시켜 나아가보자!

위에 구조에서 front controller가 요청을 먼저 받고 이후 컨트롤러를 호출한다. 이렇게 요청에 따른 컨트롤러를 호출하기 위해서는 컨트롤러 조회를 위한 매핑 과정이 필요하다. 따라서 매핑 정보 부분을 추가하였다. 

![image-20240327141356758](assets/Servlet과 MVC pattern/image-20240327141356758.png)

* 매핑 정보

```java
// FrontController
private Map<String, ControllerV1> controllerMap = new HashMap<>();
public FrontControllerServletV1() {
   controllerMap.put("/front-controller/v1/members/new-form", newMemberFormControllerV1());
   controllerMap.put("/front-controller/v1/members/save", newMemberSaveControllerV1());
   controllerMap.put("/front-controller/v1/members", newMemberListControllerV1());
}
```

* 컨트롤러 호출

```java
String requestURI = request.getRequestURI();
ControllerV1 controller = controllerMap.get(requestURI);
if (controller == null) {
	response.setStatus(HttpServletResponse.SC_NOT_FOUND);
	return; 
}
controller.process(request, response);
```

### FrontController V2 - View 분리

기존 Controller는 직접 뷰로 이동하는 부분이 존재한다.

따라서 각 컨트롤러에서 중복을 최대한 없애기 위해 Front Controller에서 다시 컨트트롤러로부터 반환해야 하는 View에 대한 정보를 받고 View 로 이동은 대신 처리해주도록 해보자

![image-20240327142746333](assets/Servlet과 MVC pattern/image-20240327142746333.png)

* View로 이동시키는 객체

```java
public class MyView {
     private String viewPath;
     public MyView(String viewPath) {
         this.viewPath = viewPath;
}
     public void render(HttpServletRequest request, HttpServletResponse response)throws ServletException, IOException {
         RequestDispatcher dispatcher = request.getRequestDispatcher(viewPath);
         dispatcher.forward(request, response);
     }
}

```

각각의 컨트롤러에서는 view 경로를 담은 MyView 객체를 반환하도록 할 것이며 이렇게 받은 객체를 Front Controller에서 받아 render 메서드를 대신 실행하게 된다.

이로써 각 컨트롤러는 어떻게 뷰가 호출되는지는 신경쓰지 않고 어디로 호출할 것인지 경로만 전달하게 되며, Front Controller는 어디로 호출할 것인가는 담당하지 않고 어떻게 호출할 것인지만 담당하게 된다. 또한 이렇게 함으로 각 Controller에서 중복으로 View를 호출하던 것을 제거할 수 있게 되었다.

### FrontController V3 - Model 추가

우리는 Front Controller에서 서블릿에 대한 처리를 단 한 번만 할 수 있도록 설계하였다. 그렇다면 각 컨트롤러에서 굳이 HttpServletRequest와 HttpServletResponse 객체를 파라미터 정보로 넘겨 받을 필요가 있을까? 그렇지 않다. 각각의 컨트롤러의 서블릿에 대해 알지 못해도 된다.(서블릿 의존성X) 따라서 해당 부분을 수정하고 View path에 대해 논리 이름을 반화하도록 변환하자!

> 각 컨트롤러에서 논리 이름을 반환하고 이를 FrontController에서 물리 이름으로 변환하여 사용한다면 나중에 ViewPath가 변경되더라도 FrontController에서만 변경해주면 된다는 장점이 있다.

이때 FrontController에서 논리 이름은 물리 이름으로 변환해주기 위해 viewResolver 객체를 사용하여 변환 처리를 맡기고 변환 받은 결과값만 FrontController에서 받아 View에 넘겨 주도록 한다.

![image-20240327151259892](assets/Servlet과 MVC pattern/image-20240327151259892.png)

위 그림에서 FrontController는 요청이 들어오면 해당 요청에 맞는 컨트롤러 정보를 매핑해서 찾아낸다. 이후 컨트롤러에 요청 처리에 필요한 파라미터 값을 전달하며 Controller를 호출한다. 그러면 Controller에서는 요청에 맞는 비즈니스 로직을 수행하고 ModelView객체를 생성해 viewPath의 논리 이름과 로직 수행 결과를 담은 ModelView 객체를 반환한다. 그러면 FrontController는 반환 받은 ModelView 객체에서 논리 이름을 가져와 ViewResolver에게 파라미터로 전달하며 새로운 객체를 생성한다. 이후 생성된 객체에 request, response 객체를 넘겨주며 render 메서드를 호출한다.

1. 사용자 요청
2. FrontController에서 요청에 맞는 Controller가 있는지 매핑
3. 매핑한 Controller에 파라미터에 파라미터 넘기며 호출
4. 호출 받은 Controller는 전달 받은 파라미터를 통해 비즈니스 로직 수행
5. Controller에서 ModelView 객체 생성
6. ModelView 객체에 logicalViewPath와 View에 랜더링 시 필요한 Data 담기
7. ModelView를 FrontController로 반환
8. FrontController에서는 viewResolve 매서드에 logicalViewPath를 파라미터로 넘기며 MyView 객체 생성
9. view 랜더링

### FrontController V4 - return String

현재는 특정 상황에는 String(경로)만 담은 ModelView객체를 리턴 받고 어떤 상황에는 String과 데이터를 담은 ModelVIew를 리턴 받고 있다. 이를 개발자 입장에서 조금 더 편하게 String(경로)만 return 받는 방식으로 바꿔 보자!

이를 위해서는 model객체를 FrontController에서 파라미터로 각 Controller에 넘겨주면 된다. 따라서 각 파라미터에서는 넘겨받은 model 파라미터에 값을 넣어주면 FrontController에서 그대로 담겨있기 때문에 직접 각 Controller에서 직접 ModelView 객체를 생성할 필요 없이 넘겨 받은 model에 값을 넣고 return은 논리 이름만 반환하게 하면 된다.

![image-20240327154957504](assets/Servlet과 MVC pattern/image-20240327154957504.png)

### FrontController V5 - 어댑터 도입

우리가 실제 프로젝트에서 요청을 처리할 때는 V3과 V4 두 버전 중 어떤 버전을 사용하게 될 지는 알 수 없다. 또한 새로운 버전으로 계속해서 업데이트 될 수 있으며 다른 방식으로 Controller를 처리할 수도 있다. 그러면 그럴 때마다 계속해서 FrontController를 바꿔서 처리해야 할까? 그렇지 않다.

지금까지 구현 방식을 보면 viewPath를 만들어 주는 애 따로 viewPath로 랜더링 해주는 애 따로 등 역할을 분리하여 수정을 최소화 할 수 있었다. 이처럼 Controller에서도 FrontController에서 Controller를 호출하는 애를 따로 만들어주면 된다.

Controller를 호출하는 애를 따로 만들어주면 FrontController는 Controller 호출이 필요할 때 Controller를 호출해줄 객체를 찾고 해당 객체는 그때그때 필요한 Controller를 호출하게 된다.

이때 Controller를 호출하는 객체를 HandlerAdapter라고 한다.

#### HandlerAdapter 역할

* Controller 지원 여부 확인
  * 특정 컨트롤러를 이 핸들러로 호출할 수 있는지를 확인한다.
* Controller 호출
  * 어댑터가 실제 컨트롤러를 호출하고, 그결과로 ModelView를 반환한다.

이전에는 프론트 컨트롤러가 실제 컨트롤러를 호출했지만 이제는 이 어댑터가 실제 컨트롤러를 대신 호출해주게 된다.

![image-20240327165248986](assets/Servlet과 MVC pattern/image-20240327165248986.png)

1. 사용자 요청
2. FrontController에서 요청에 맞는 Handler(Controller)가 있는지 매핑
3. 요청에 맞는 Handler가 있다면 해당 Handler를 처리할 수 있는 HandlerAdapter가 존재하는 확인
4. HandlerAdapter가 존재한다면 해당 HandlerAdapter를 통해 handler 호출
5. handler러는 요청 사항에 맞는 비즈니스 로직을 수행 후 HandlerAdapter에 ModelView 반환
6. HandlerAdapterfmf FrontController에 ModelView 반환
7. FrontController는 반환 받는 ModelView에서 ViewPath를 viewResolver로 전달하며 MyView 객체 생성
8. Myview 객체를 통해 FrontController에서 view 랜더링

이렇게 만든 최종 구조는 spring MVC와 거의 똑같다.

![image-20240327165813829](assets/Servlet과 MVC pattern/image-20240327165813829.png)