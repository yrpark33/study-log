# DB 트랜잭션과 물리 파일의 원자성(Atomicity) 불일치 해결

## 1. 문제 제기: "DB는 되돌아가는데, 파일은 왜 그대로인가?"

게시글 수정 로직은 DB 정보 갱신과 물리 파일 저장이라는 두 가지 이질적인 작업이 동시에 일어납니다.

- Spring의 @Transactional: DB 작업 중 예외가 발생하면 SQL 작업(Insert, Update, Delete)을 이전 상태로 되돌립니다.

- 파일 시스템의 한계: 하지만 이미 서버 하드디스크에 저장된(Write) 물리 파일은 DB 트랜잭션의 범위 밖입니다. DB가 롤백되어도 파일은 삭제되지 않고 서버에 '유령 파일(Orphan File)'로 남게 됩니다.

**핵심 과제: DB와 파일 시스템 간의 상태를 일치시켜 전체 프로세스의 원자성(Atomicity)을 어떻게 보장할 것인가?**

## 2. 기술적 분석: 원자성 보장의 장애물

### ① 트랜잭션의 불일치 현상
1. 기존 파일 정보 DB 삭제 성공 (Rollback 가능)

2. 새로운 파일 물리 저장소 업로드 성공 (Rollback 불가)

3. 게시글 수정 DB 반영 중 예외 발생 (Rollback 실행)

**결과: DB는 수정 전으로 복구되었으나, 서버에는 2번에서 업로드한 파일이 그대로 남아있어 스토리지 낭비와 데이터 불일치 발생.**


### ② 해결 전략 수립
스프링 프레임워크가 자동으로 지원하지 않는다면, 개발자가 애플리케이션 레벨에서 수동 롤백 로직을 설계해야 합니다.


## 3. 설계 및 구현: 수동 롤백(Manual Rollback) 아키텍처
로직의 흐름을 try-catch로 감싸고, 실패 시 업로드된 파일을 직접 추적하여 삭제하는 방식을 채택했습니다.

**[구현 코드 핵심]**

```java
  public void modifyBoard(...) {
    List<BoardFileDTO> addedFileList = new ArrayList<>(); // 새로 추가된 파일을 추적하기 위한 리스트
    
    try {
        // 1. 기존 파일 DB 정보 삭제
        boardMapper.deleteFiles(boardId);
        
        // 2. 새 파일 물리 저장소 저장 및 리스트에 기록
        addedFileList = fileUploader.uploadFiles(addedFiles);
        
        // 3. 파일 정보 및 게시글 DB 반영 (예외 발생 시 물리 파일은 이미 저장된 상태)
        boardMapper.insertFiles(boardDTO);
        boardMapper.update(boardDTO);
        
    } catch (Exception e) {
        // [수동 롤백 로직]
        // DB 작업 실패 시, 업로드에 성공했던 파일들을 즉시 삭제하여 원자성 보장
        if (!addedFileList.isEmpty()) {
            fileUploader.deleteFiles(addedFileList);
            log.warn("DB 트랜잭션 실패로 인한 업로드 파일 수동 롤백 완료");
        }
        throw e; // Controller 및 트랜잭션 매니저에게 예외 전달 (DB 롤백 유도)
    }
}
```

## 4. 검증: 의도적 예외 발생을 통한 롤백 테스트
단순히 코드를 짜는 것에 그치지 않고, 실제로 사고가 났을 때 파일이 잘 지워지는지 통합 테스트를 통해 증명했습니다.

### ① 테스트 시나리오
1. 게시글 수정 요청 시 정상적인 파일을 함께 첨부.

2. boardMapper.update() 호출 시점에 DB 제약 조건 위반(글자 수 초과)을 강제로 발생시킴.

3. catch 블록이 실행된 후, 서버의 업로드 폴더(original, thumbnail)에 파일이 삭제되었는지 확인.

### ② 검증 결과
```java
@Test
void DB_에러_발생_시_물리파일_삭제_검증() {
    // 1. Given: 비정상적인 길이의 제목을 가진 DTO와 파일 준비
    // 2. When: 수정 로직 호출 (Exception 발생 예상)
    assertThrows(Exception.class, () -> boardService.modifyBoard(invalidDto, ...));

    // 3. Then: 원본/썸네일 폴더 모두 비어있어야 함
    assertEquals(0, new File(uploadPath, "original").listFiles().length);
    assertEquals(0, new File(uploadPath, "thumbnail").listFiles().length);
}
```
- 결과: 예외 발생 직후 수동 롤백 로직이 작동하여, 서버 폴더가 깨끗하게 비워지는 것을 확인했습니다.


## 5. 결론 및 회고

- **원자성에 대한 이해**: 트랜잭션의 범위가 DB를 넘어 외부 리소스(파일, 외부 API 등)로 확장될 때의 위험성을 인지하고 대응책을 마련했습니다.

- **방어적 로직의 중요성**: 해피 케이스(Happy Path)보다 예외 상황(Edge Case)에서 시스템이 얼마나 견고하게 버티는지가 좋은 소프트웨어의 기준임을 배웠습니다.


