# MultipartFile 빈 파일 처리 및 SQL Syntax Error 해결

## 1. 문제 상황
- 게시글 작성 시 첨부파일을 선택하지 않았음에도 불구하고, 서버 측에서 `SQLSyntaxErrorException`이 발생하며 DB 인서트에 실패함.
- **에러 메시지**: `Cause: java.sql.SQLSyntaxErrorException: ... right syntax to use near '' at line 2`
- **발생 지점**: `boardMapper.insertFiles` 호출 시
- **실제 생성된 SQL**: values 뒤에 데이터 세트 (...)가 생성되지 않아 유효하지 않은 SQL 문법이 됨
  ```sql
  insert into tbl_board_file(board_id, file_name, uuid, sort_order, image) values
  ```

## 2. 원인 분석

### 비정상적인 로직 흐름
1. `BoardService`의 `writeBoard` 메서드 실행 중, `FileUploader`의 `uploadFiles(files)`에서 **비어있는 리스트**가 반환됨.
2. `boardDTO`의 `files` 필드가 **빈 ArrayList**인 채로 `boardMapper.insertFiles(boardDTO)`가 실행됨.
3. **MyBatis의 `<foreach>` 문**은 대상 리스트가 비어있을 경우 `VALUES` 이후의 구문을 생성하지 못함.
4. 결국 **문법이 불완전한 상태**로 DB에 전송되어 `SQLSyntaxErrorException` 발생.

## 3. 해결 방법

### 3.1 물리 파일 저장 전 검증 강화
파일을 업로드하지 않은 경우 사전에 필터링하여 불필요한 로직 수행 방지.

```java
if(files == null || files.length == 0 || (files.length == 1 && files[0].isEmpty())) {
    return fileDTOList;
}
```

### 3.2 BoardService의 boardWrite에서 파일 존재 여부 체크
첨부파일이 없는 경우 파일정보를 insert하는 매퍼 호출 자체를 차단하여 SQL 에러 방지.

```java
if(boardDTO.getFiles().isEmpty()) {
    return boardDTO.getBoardId();
}
boardMapper.insertFiles(boardDTO);
```

## 4. 배운 점

- HTML의 `<input type="file">` 요소는 파일을 선택하지 않더라도, 서버 컨트롤러에서 `MultipartFile[]` (배열/리스트) 형태로 파라미터를 받으면 크기(`length`)가 1인 배열이 전달됨을 이해함.
- 이때 0번째 요소는 내용이 없는 **빈 파일 객체**(`isEmpty() == true`)이므로, 실질적인 파일 존재 여부를 파악하려면 배열의 크기뿐만 아니라 `isEmpty()` 메서드를 반드시 병행 체크해야 함을 깨달음.
- MyBatis `<foreach>` 사용 시, 리스트가 비어 있으면 불완전한 SQL(`VALUES` 구문 누락 등)이 생성되어 `SQLSyntaxErrorException`이 발생할 수 있음을 학습함.
- 이를 방지하기 위해 서비스 계층에서 리스트의 유효성을 사전에 검증하는 **방어적 프로그래밍(Validation)**의 중요성을 체감함.
