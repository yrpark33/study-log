# Fail-Fast 전략을 통한 서버 자원 최적화 및 안정성 향상

## 1. 문제 제기: "잘못된 요청을 어디까지 허용할 것인가?"

BoardService.boardWrite 메서드에서 수정하려는 Board객체가 존재하지 않는 경우, json 파싱 작업이 실패하는 경우의 예외처리를 하던 중 더 효율적인 코드를 만들고자 고민했습니다.
물론 메서드 실행중 예외가 발생하면 DB와 물리적 저장소 역시 모두 원상복구 되지만 에러가 발생할 수 있는 위험요소를 초반에 배치하면 효과적이리라는 판단을 했습니다.
- 비효율의 발생: 만약 DB 삭제 후 100MB의 파일을 업로드까지 마쳤는데, 이후 JSON 파싱 에러나 권한 오류가 발견된다면?
- 리소스 낭비: 디스크 I/O, DB 커넥션 등은 서버 자원의 손실로 이어집니다.

**핵심 과제: 유효하지 않은 요청을 로직의 가장 최전방에서 차단하여 불필요한 리소스 소모를 최소화할 방법은 무엇인가?**

## 2. 기술적 해법: Fail-Fast (즉시 실패) 원칙 적용

Fail-Fast란 시스템이 오류를 최대한 빨리 감지하고 즉시 응답을 중단하는 설계 원칙입니다. 이를 `modifyBoard` 로직에 적용하여 단계별 방어막을 구축했습니다.

### 1단계: Board객체의 존재 여부 
- **검증**: `boardId` 존재 여부 확인 및 작성자 권한 체크.
- **이유**: DB 커넥션을 맺거나 파일을 건드리기 전, 가장 가벼운 체크로 부적절한 접근을 차단합니다.

### 2단계: 데이터 형식 검증
- **검증**: JSON 문자열 파싱 (`parseJsonToList`).
- **이유**: 파일 업로드(I/O 작업)는 매우 무거운 작업입니다. 파일이 서버에 쓰이기 전, 전달된 메타데이터(JSON)의 형식이 올바른지 먼저 확인하여 물리적인 디스크 쓰기 작업을 원천 차단합니다.

## 3. 구현: 로직의 순서가 곧 성능이다
수정 로직의 순서를 전략적으로 배치하여 방어막을 형성했습니다.

```java
public void modifyBoard(...) {
    // 1. [Fail-Fast] 유효성 검사
    if(boardMapper.selectById(boardDTO.getBoardId()) == null) {
        throw new ApplicationException(404, "BOARD_NOT_FOUND");
    }
    // 2. [Fail-Fast] 무거운 작업(I/O) 전 데이터 파싱 검증
    List<BoardFileDTO> oldFiles = parseJsonToList(oldFileInfosJson);
    List<BoardFileDTO> deletedFiles = parseJsonToList(deletedFileInfosJson);

    //아래 작업들은 파싱 성공시에만 실행

    //물리 파일 시스템 접근
    List<BoardFileDTO> addedFileList = fileUploader.uploadFiles(addedFiles);
    
    try {
        // DB 반영 작업 (예외 발생 시 물리 파일은 이미 저장된 상태)
        boardMapper.deleteFiles(boardDTO.getBoardId());
        // ... (중략)
    } catch (Exception e) {
        // [Manual Rollback] 파일 시스템 원자성 보장
        if(!addedFileList.isEmpty()) {
            fileUploader.deleteFiles(addedFileList);
        }
        throw e;
    }
}

```

## 4. 검증: 예외 발생에 따른 로직 중단 확인
단위 테스트를 통해 잘못된 데이터(JSON)가 들어왔을 때, 이후의 무거운 작업(파일 업로드, DB 삭제)이 실행되지 않고 즉시 예외를 던지는지 확인했습니다.
```java
@Test
	@DisplayName("잘못된 형식의 JSON이 전달되면 파싱 에러(RuntimeException)가 발생해야 한다")
	void testModifyBoard_ParsingError() throws IOException {
	    // given: 잘못된 JSON 데이터 준비
		BoardDTO board = createBoardDTO();
		boardService.writeBoard(board, createMockFiles("EMPTY_FILE"));
		BoardDTO updateDTO = BoardDTO.builder()
                            .boardId(board.getBoardId())
                            .title("Json 테스트")
                            .content("Json 테스트").build();
		String invalidJson = "!!!INVALID_JSON!!!";

	    // when & then: 예외가 발생하는지 검증
		RuntimeException exception = assertThrows(RuntimeException.class, () -> {
	        boardService.modifyBoard(updateDTO, invalidJson, "[]", createMockFiles("MIXED"));
	    });

	    // 추가 검증: 에러 메시지가 내가 설정한 것과 일치하는지 확인
	    assertEquals("파일 정보 형식이 올바르지 않습니다.", exception.getMessage());
	    
	    File originalDir = new File(uploadPath, "original");
	    File thumbnailDir = new File(uploadPath, "thumbnail");

	    assertFalse(originalDir.exists(), "원본 폴더가 존재하지 않아야합니다.");
	    assertFalse(thumbnailDir.exists(), "썸네일 폴더가 존재하지 않아야합니다.");
	    
	}
```
- **결과**: JSON 파싱 단계에서 즉시 예외가 발생하고, 후속 `uploadFiles` 호출이 차단되어 
자원이 보호됨을 확인했습니다.
검증 방식으로 `assertFalse(exists())`를 사용한 이유는, `uploadFiles`는 유효한 파일이 
1개 이상일 때만 디렉토리를 생성하기 때문입니다. `writeBoard` 호출시에는 빈 파일을 전달했으므로 여전히 디렉토리가 생성되지 않습니다.
`modifyBoard`에는 유효한 파일 2개를 전달했음에도 디렉토리가 생성되지 않았다면, `uploadFiles`가 실행되지 않았음이 증명됩니다.


## 5. 결론 및 회고
 - **비용 중심적 사고**: 코드를 짤 때 단순히 기능의 성공/실패만 보는 것이 아니라, 각 라인이 소모하는 '컴퓨팅 비용'을 고려하게 되었습니다.
 - **안정성 향상**: 시스템의 앞단에서 에러를 잡아내면, 디스크 I/O, DB 커넥션등으로 인한 서버 자원의 손실을 막고 사후 처리(롤백) 로직이 작동해야 할 빈도를 줄여 전체 시스템의 안정성을 향상시킬 수 있음을 체득했습니다.




