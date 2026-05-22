# ControllerAdvice와 스프링 시큐리티 예외 충돌 해결

## 1. 문제 제기: "권한 없는 페이지 접근 시 왜 엉뚱한 화면으로 가지?"

`@PreAuthorize("isAuthenticated()")` 가 걸린 페이지에 비로그인 상태로 접근했을 때 로그인 페이지로 가야 하는데, `ControllerAdvice`에 등록된 예외처리가 먼저 낚아채서 홈 화면으로 리다이렉트 되는 문제가 발생했습니다.

## 2. 원인 분석
### ① ControllerAdvice의 Exception 포괄 처리
`@ExceptionHandler(Exception.class)` 가 시큐리티가 던지는 `AccessDeniedException`, `AuthenticationException` 까지 전부 가로채버렸습니다.

```java
@ExceptionHandler(Exception.class)
	public String handleAll(Exception ex, RedirectAttributes rttr) {
		log.error("Internal Server Error: ", ex);
		rttr.addFlashAttribute("errorMsg", "서비스 이용에 불편을 드려 죄송합니다. 잠시 후 다시 시도해 주세요.");
		return "redirect:/";
	}
```
### ② import 문제로 인한 instanceof 오작동
시큐리티 예외를 다시 던지도록 조건문을 추가했습니다.
```java
if (ex instanceof AuthenticationException || ex instanceof AccessDeniedException) {
	        throw new RuntimeException(ex); // 시큐리티 예외는 다시 던지기
}
```
그런데 여전히 ControllerAdvice가 처리하는 문제가 발생했습니다. 로그로 예외 타입을 직접 찍어서 확인했습니다.

```java
log.error("exception type: " + ex.getClass().getName());
// 출력: org.springframework.security.access.AccessDeniedException
```

예외 타입은 맞는데 조건문이 `false`로 평가되고 있었습니다. 원인은 `AccessDeniedException`이 **잘못된 패키지로 import** 돼있어서 `instanceof` 조건이 항상 `false`로 빠지고 있었던 것이었습니다. 올바른 import는 아래와 같습니다.

```java
import org.springframework.security.access.AccessDeniedException;
import org.springframework.security.core.AuthenticationException;
```
## 3. 해결
올바른 패키지로 import를 수정하여 `instanceof` 조건이 제대로 작동하도록 했습니다.
```java
@ExceptionHandler(Exception.class)
public String handleAll(Exception ex, RedirectAttributes rttr) {

    if (ex instanceof AuthenticationException || ex instanceof AccessDeniedException) {
        throw new RuntimeException(ex); // 시큐리티 예외는 다시 던지기
    }

    log.error("Internal Server Error: ", ex);
    rttr.addFlashAttribute("errorMsg", "서비스 이용에 불편을 드려 죄송합니다.");
    return "redirect:/";
}
```

## 4. 결론 및 회고
* **예외 처리 범위 주의**: `@ExceptionHandler(Exception.class)` 는 모든 예외를 잡기 때문에 시큐리티 예외와 충돌할 수 있음을 체득했습니다.
* **import 문제의 위험성**: 같은 이름의 클래스가 여러 패키지에 존재할 때 잘못된 import로 인해 `instanceof` 조건이 오작동할 수 있습니다. 로그로 예외 타입을 직접 찍어서 원인을 찾았습니다.
* **책임 분리**: 시큐리티 예외는 시큐리티가, 애플리케이션 예외는 ControllerAdvice가 처리하도록 책임을 분리하는 것이 중요함을 배웠습니다.
