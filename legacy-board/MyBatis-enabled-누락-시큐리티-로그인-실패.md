# MyBatis resultMap 누락으로 인한 시큐리티 로그인 실패
## 1. 문제 제기: "아이디와 비밀번호가 맞는데 왜 로그인이 안 되지?"
DB에 정상적으로 데이터가 등록된 상태에서 올바른 아이디와 비밀번호를 입력했음에도 "유효하지 않은 사용자입니다" 메시지가 출력되며 로그인에 실패했습니다. 콘솔에는 별다른 에러 로그도 찍히지 않아 원인 파악이 어려운 상황이었습니다.

## 2. 원인 분석
### ① 스프링 시큐리티의 `isEnabled()` 동작
스프링 시큐리티는 `UserDetails`의 `isEnabled()`가 `false`를 반환하면 인증 자체를 거부합니다. 별도의 예외 로그 없이 "유효하지 않은 사용자" 로 처리하기 때문에 콘솔에 에러가 찍히지 않는 게 의도된 동작입니다.
### ② boolean 기본값
`AccountDTO`의 `enabled` 필드는 `boolean` 타입이라 초기화하지 않으면 기본값이 `false`입니다.
```java
@Data
public class AccountDTO extends BaseTimeDTO implements UserDetails {
	
	private String username;
	private String password;
	private String name;
	private String email;
	private List<AccountRole> roleNames;
	private boolean enabled;
	private LocalDateTime updatedAt;
}
```

### ③ MyBatis resultMap 누락
`AccountMapper`의 `resultMap`과 `SELECT` 쿼리에서 `enabled` 컬럼을 매핑하지 않았기 때문에, DB에서 `enabled = true`로 저장된 값이 DTO에 반영되지 않고 기본값인 `false`가 유지됐습니다.
```java
@Override
public boolean isEnabled() {
    return enabled; // enabled가 매핑 안 되면 항상 false
}
```


```xml
<!-- 누락된 상태 -->
<resultMap type="AccountDTO" id="selectMap">
    <id property="username" column="username"/>
    <result property="password" column="password"/>
    <result property="name" column="name"/>
    <result property="email" column="email"/>
    <!-- enabled 누락 -->
    <collection property="roleNames" ofType="AccountRole">
        <result column="rolename" typeHandler="org.apache.ibatis.type.EnumTypeHandler" javaType="AccountRole"/>
    </collection>
</resultMap>

<select id="selectOne" resultMap="selectMap">
    SELECT a.username, password, name, email, rolename FROM tbl_account a
    LEFT JOIN tbl_account_roles r
    ON a.username = r.username
    WHERE a.username = #{username}
</select>
```

## 3. 해결
`resultMap`과 `SELECT` 쿼리에 `enabled` 추가
```xml
  <resultMap type="AccountDTO" id="selectMap">
    <id property="username" column="username"/>
    <result property="password" column="password"/>
    <result property="name" column="name"/>
    <result property="email" column="email"/>
    <result property="enabled" column="enabled"/>
    <collection property="roleNames" ofType="AccountRole">
        <result column="rolename" typeHandler="org.apache.ibatis.type.EnumTypeHandler" javaType="AccountRole"/>
    </collection>
</resultMap>

<select id="selectOne" resultMap="selectMap">
    SELECT a.username, password, name, email, rolename, enabled FROM tbl_account a
    LEFT JOIN tbl_account_roles r
    ON a.username = r.username
    WHERE a.username = #{username}
</select>
```
## 4. 결론 및 회고
* **시큐리티의 보안 정책 이해**: 스프링 시큐리티는 인증 실패 원인을 외부에 노출하지 않아 에러 로그 없이 "유효하지 않은 사용자" 메시지만 표시됩니다. 이는 의도된 보안 동작임을 이해했습니다.
* **boolean 기본값 주의**: `boolean` 타입은 초기화하지 않으면 `false`로, MyBatis 매핑 누락 시 의도치 않은 동작으로 이어질 수 있음을 체득했습니다.
* **직관적 디버깅**: 에러 로그 없이 증상만으로 원인을 추론하는 경험을 통해 프레임워크 동작 원리를 이해하는 것이 디버깅에 얼마나 중요한지 느꼈습니다.
