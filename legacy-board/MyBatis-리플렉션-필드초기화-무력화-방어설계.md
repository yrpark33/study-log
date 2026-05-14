# MyBatis 환경에서 필드 초기화 유지에 대한 고찰과 방어적 설계

## 1. 문제 제기: "필드 초기값이 왜 무시될 위험이 있는가?"
자바에서 필드를 선언하며 즉시 초기화(private List<T> files = new ArrayList<>();)하면 객체 생성 시 당연히 빈 리스트가 할당될 것으로 기대합니다. 하지만 MyBatis와 같은 프레임워크가 DB 데이터를 매핑하는 과정에서 이 초기값이 유실될 수 있다는 의문이 생겼습니다.

**가설**: MyBatis가 DB 데이터를 매핑할 때 결과가 null이라면, 내가 설정한 초기 인스턴스를 null로 덮어씌울 수도 있지 않을까?

**핵심 의문**: 객체 생성 시점의 초기화와 MyBatis의 리플렉션 주입 중 무엇이 우선순위를 갖는가?

## 2. 기술적 분석: 메커니즘의 충돌

### ① 객체 생성과 필드 초기화
MyBatis는 SQL 결과를 매핑할 때 기본 생성자를 호출하여 객체를 생성합니다. 자바의 객체 생성 규칙에 따라, 이 시점에 필드의 초기화 코드(= new ArrayList<>())가 실행되며 빈 리스트 인스턴스가 만들어집니다.

### ② MyBatis의 주입 방식 (Reflection vs 필드 초기값)
진짜 문제는 객체 생성 이후, 데이터를 필드에 채워 넣는 주입(Injection) 단계에서 발생합니다.

* 리플렉션(Reflection)의 권한: MyBatis는 리플렉션을 사용하여 private 필드에 직접 접근합니다. 이는 자바의 접근 제어자를 우회하여 필드 값을 강제로 할당(Assignment)할 수 있는 강력한 권한입니다.

* 초기값 덮어쓰기(Overwrite): 만약 DB 조회 결과가 null이라면, MyBatis는 리플렉션을 통해 기존의 빈 리스트 인스턴스를 null로 교체해버릴 위험이 있습니다. 이 경우 우리가 설계한 초기화 로직은 무력화됩니다.

## 3. 검증: 단위 테스트를 통한 동작 확인
MyBatis가 실제로 초기값을 유지해주는지, 아니면 `null`로 덮어씌우는지 확인하기 위해 테스트 코드를 작성했습니다.

```java
@Test
	void 필드_초기값이_조회시에도_유지되는지_확인() {
	    // given: 파일 없는 게시글 저장
	    BoardDTO board = setUpBoard();

	    // when: 조회
	    BoardDTO savedBoard = boardMapper.selectById(board.getBoardId());

	    // then
	    // 1. MyBatis가 초기값을 유지해주면 결과는 Not Null (빈 리스트)
	    // 2. MyBatis가 덮어씌우면 결과는 Null
	    assertNotNull(savedBoard.getFiles(), "MyBatis가 초기값(ArrayList)을 유지해주는지 확인합니다.");
	    log.info("MyBatis 조회 후 files 필드 상태: " + savedBoard.getFiles());
	}
```

* 결과: 현재 프로젝트 환경에서는 MyBatis가 기존 필드의 인스턴스를 유지하는 것을 확인했습니다. 하지만 이는 프레임워크의 설정이나 버전에 따라 변할 수 있는 환경 의존적인 결과였습니다.

## 4. 해결: 환경에 의존하지 않는 방어적 설계
프레임워크의 동작 방식을 100% 통제할 수 없음을 인정하고, 데이터가 나가는 통로(Getter)를 제어하는 전략을 선택했습니다.

```java
public List<BoardFileDTO> getFiles() {
    // MyBatis가 리플렉션으로 필드에 null을 꽂았더라도, 
    // 사용하는 쪽에서는 언제나 빈 리스트를 받도록 보장함.
    return this.files == null ? new ArrayList<>() : this.files;
}
```

이 방식은 MyBatis가 기존 필드의 인스턴스를 유지하든, 리플렉션으로 들어왔든 상관없이 최종 소비 시점에서의 안전성을 보장합니다.

### [방어 로직의 무결성 검증]

프레임워크가 리플렉션으로 필드를 직접 조작하는 상황을 가정하여, 단위 테스트에서 리플렉션을 이용해 강제로 null을 주입한 뒤 Getter의 동작을 확인했습니다.

```java
@Test
	@DisplayName("리플렉션 등으로 필드에 강제로 null이 할당되어도 Getter는 빈 리스트를 반환해야 한다")
	void getter_null_방어_테스트() throws Exception {
	    // 1. Given: 객체 생성
	    BoardDTO board = setUpBoard();

	    // 2. 리플렉션을 사용하여 강제로 필드에 null 주입 (MyBatis의 동작을 흉내냄)
	    java.lang.reflect.Field field = BoardDTO.class.getDeclaredField("files");
	    field.setAccessible(true);
	    field.set(board, null);

	    // 3. When & Then: Getter 호출 시 null이 아닌 빈 리스트가 나오는지 확인
	    assertNotNull(board.getFiles(), "필드가 null이어도 Getter는 빈 리스트를 반환해야 함");
	    assertEquals(0, board.getFiles().size(), "반환된 리스트는 비어있어야 함");
	}
```

#### ① 테스트 시나리오 작성

<img width="821" height="413" alt="image" src="https://github.com/user-attachments/assets/9d3803f9-77c1-45a6-92c5-667d88f14d3e" /><br>
MyBatis를 통해 DB 조회를 마친 객체의 필드를 리플렉션으로 강제 오염(null)시킨 후, Getter의 반환값을 검증합니다.

#### ② 검증 결과 (JUnit)
<img width="1024" height="177" alt="image" src="https://github.com/user-attachments/assets/3e8eb17c-fe28-48aa-8090-7ae0874f0982" /><br>
테스트 실행 결과, 모든 Assertion을 통과하며 초록불(Green Bar)이 점등되었습니다.

#### ③ 최종 로그 확인
<img width="1407" height="23" alt="image" src="https://github.com/user-attachments/assets/cc1f6455-a484-4d3d-bc41-ec25706db873" /><br>
실제 런타임 로그에서도 null이 아닌 빈 리스트([])가 안전하게 반환되고 있음을 확인할 수 있습니다.




## 5. 결론 및 회고
* 도구의 내부 원리 이해: 단순히 기능을 구현하는 것을 넘어, 사용하는 라이브러리가 리플렉션을 통해 어떻게 객체를 조작하는지 깊이 있게 파고든 경험이었습니다.

* 방어적 프로그래밍(Defensive Programming): "설마 null로 교체하겠어?"라는 낙관보다, "프레임워크가 어떻게 작동하든 내 객체는 안전한 응답을 보장해야 한다"는 책임감을 실천했습니다.

* 지적 호기심의 가치: "명확하지 않은 동작"을 불편해하고 끝까지 논리적 근거를 찾아내는 과정이 견고한 소프트웨어를 만드는 밑바탕임을 깨달았습니다.
