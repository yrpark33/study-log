# 렌더링 방식에 따른 권한 체크 방법 차이

## 1. 문제 제기: "왜 JSTL이 작동하지 않을까?"

게시글 수정/삭제 버튼은 작성자에게만 보이도록 JSTL로 처리했습니다. 댓글도 같은 방식으로 하려고 했는데 의도대로 작동하지 않았습니다.

## 2. 원인 분석: 렌더링 방식의 차이

### ① 게시글 - 서버 사이드 렌더링

게시글 상세 화면은 서버에서 JSP를 렌더링해서 클라이언트에 전달합니다. JSP가 서버에서 먼저 실행되기 때문에 JSTL과 EL 표현식을 사용할 수 있습니다.

```jsp
<c:if test="${!board.deleted && (secInfo.username == board.writer || fn:contains(roles, 'ROLE_ADMIN'))}">
    <a class="btn btn-warning" href="/board/modify/${board.boardId}?${dto.toQueryString()}">수정</a>
</c:if>
```

### ② 댓글 - 클라이언트 사이드 렌더링 (Ajax)

댓글 목록은 Ajax로 데이터를 받아서 JavaScript 템플릿 리터럴로 동적으로 화면을 그립니다. JavaScript가 실행되는 시점은 JSP 렌더링이 끝난 이후라서 JSTL이 템플릿 리터럴 안에서 의도대로 작동하지 않습니다.

```javascript
// JSTL을 템플릿 리터럴 안에 쓰면 작동 안 함
liStr += `<li>
    <c:if test="\${!board.deleted && (secInfo.username == board.writer || fn:contains(roles, 'ROLE_ADMIN'))}">
        <button>수정</button>
    </c:if>
</li>`;
```

## 3. 해결: 렌더링 방식에 맞는 권한 체크

댓글 목록 Ajax 응답에 현재 로그인한 사용자의 `username`과 `admin` 여부를 함께 담아 내려주고, JavaScript에서 조건 처리했습니다.

### CommentPageResponseDTO에 인증 정보 추가:

```java
public class CommentPageResponseDTO extends PageResponseDTO {
    private List<CommentDTO> commentDTOList;
    private String username;
    private boolean admin;
}
```

### JavaScript에서 조건 처리:

```javascript
function printComments(data) {
  const { commentDTOList, username, admin } = data;

  for (commentDTO of commentDTOList) {
    const canModify =
      !commentDTO.deleted && (username === commentDTO.writer || admin);

    liStr += `<li>
            ${canModify ? '<button class="commentModifyBtn">수정</button>' : ""}
            ${canModify ? '<button class="commentRemoveBtn">삭제</button>' : ""}
        </li>`;
  }
}
```

## 4. 결론 및 회고

- **렌더링 방식 이해**: 서버 사이드 렌더링은 JSTL/EL, 클라이언트 사이드 렌더링은 JavaScript로 조건 처리해야 한다는 것을 직접 경험했습니다.
- **응답 설계**: Ajax 응답에 화면 렌더링에 필요한 인증 정보를 함께 담아 내려주는 방식을 적용했습니다.
