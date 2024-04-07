---
title: JPA 벌크 연산(Bulk Operation)
date: 2023-07-20 12:00:00 +09:00
categories: [Springboot, JPA]
tags: [Java, SpringBoot, DB, JPA]    
---

JPA에서 벌크 연산(Bulk Operation)이란 여러 레코드에 대한 대량의 변경(업데이트, 삭제) 작업을 단일 쿼리로 수행하는 것을 의미합니다.

개별 엔티티를 하나씩 업데이트하거나 삭제하는 것보다 훨씬 효율적이기 때문에 많이 사용합니다.

## Bulk update, delete

<hr style="height: 2px; border: none; background-color: white;" />

JPA에서 데이터를 조작하는 데는 주로 두 가지 방법이 사용됩니다.

JPQL(Java Persistence Query Language)을 사용하는 방법과 QueryDSL을 사용하는 방법입니다.

이 두 방법을 활용한 Bulk Update에 대한 예시를 보겠습니다.

### JPQL



```java
@Modifying
@Query("UPDATE User u SET u.status = :status WHERE u.id IN :userIds")
long updateStatusByIdIn(@Param("userIds") List<Long> userIds, @Param("status") String status);
```

- User 엔티티의 status 필드를 업데이트하는 쿼리입니다. 특정 조건 (u.id IN :userIds)에 맞는 모든 User 엔티티를 대상으로 합니다.
- @Modifying 어노테이션은 쿼리가 데이터를 변경하는 작업(즉, update 또는 delete 연산)임을 나타냅니다.


### QueryDSL

```java
@Override
public long updateStatusByIdIn(List<Long> userIds, String status) {
    return queryFactory
        .update(qUser)
        .set(qUser.status, status)
        .where(qUser.id.in(userIds))
        .execute();
}
```

- queryFactory.update(qUser): User 엔티티를 대상으로 하는 update 쿼리를 시작합니다.
- .set(qUser.status, status): status 필드를 새로운 값으로 설정합니다.
- .where(qUser.id.in(userIds)): 조건을 지정하여 특정 사용자 ID 목록에 속하는 엔티티만을 대상으로 합니다.
- .execute(): 쿼리를 실행합니다.

> **<font color='dodgerblue'>bulk delete는 update를 delete로 바꿔주면 됩니다.</font>**

<br>

## Bulk update, delete 주의사항

<hr style="height: 2px; border: none; background-color: white;" />

벌크 연산은 영속성 컨텍스트를 무시하고 직접 데이터베이스에 쿼리를 실행합니다. 이는 영속성 컨텍스트가 현재 관리하고 있는 엔티티의 상태와 실제 데이터베이스의 상태 사이에 불일치를 초래할 수 있습니다.

이를 해결하기 위해 entityManager.flush()와 entityManager.clear()를 사용하여 영속성 컨텍스트를 동기화해야 합니다.

- entityManager.flush() : 영속성 컨텍스트의 변경 내용을 데이터베이스에 반영
- entityManager.clear() : 영속성 컨텍스트를 초기화

일반적으로 @Transactional 어노테이션을 사용하는 경우, 트랜잭션이 자동으로 관리되며 flush() 호출은 필요하지 않습니다.

> **<font color='dodgerblue'>하지만 벌크 연산과 같이 영속성 컨텍스트를 우회하는 연산을 수행하기 전에, 영속성 컨텍스트에 변경사항이 있다면 명시적으로 flush()를 호출하여 이러한 변경사항을 먼저 데이터베이스에 반영하는 것이 안전할 수 있습니다.</font>**

아래 내용은 영속성 컨텍스트 동기화를 진행한 예시 코드입니다.

### JPQL

```java
@Modifying(clearAutomatically = true)
@Query("UPDATE User u SET u.status = :status WHERE u.id IN :userIds")
long updateStatusByIdIn(@Param("userIds") List<Long> userIds, @Param("status") String status);
```

@Modifying(clearAutomatically = true) 설정을 사용함으로써, updateStatusByIdIn 메서드가 실행된 후에 EntityManager.clear()가 호출된 것과 유사한 효과를 얻을 수 있습니다.

> **<font color='dodgerblue'>이 옵션을 사용할 때는 영속성 컨텍스트에 변경사항이 있다면 flush()로 반영하고 사용하는것이 안전합니다.</font>**


### QueryDSL

```java
@Override
public long updateStatusByIdIn(List<Long> userIds, String status) {
  long updatedCount = queryFactory
      .update(qUser)
      .set(qUser.status, status)
      .where(qUser.id.in(userIds))
      .execute();

  // 영속성 컨텍스트를 동기화
  entityManager.flush();
  entityManager.clear();
  
  return updatedCount;
}
```

entityManager.flush()와 entityManager.clear()를 사용하여 영속성 컨텍스트를 동기화합니다.


<br>

## Bulk insert (batch insert)

<hr style="height: 2px; border: none; background-color: white;" />

Bulk Insert는 다량의 데이터를 단일 작업으로 삽입하는 효율적인 방법입니다. 이번에는 JPA를 사용하여 Bulk Insert를 수행할 때 고려해야 할 중요한 사항들에 대해 살펴보겠습니다.

### IDENTITY 전략과 batch insert 비활성화

JPA saveAll()은 여러 엔티티를 한 번의 연산으로 저장할 것처럼 보이지만 동작하는 내용을 살펴보면 어떤경우에는 개별 insert 쿼리를 실행합니다.

그 이유는 id전략을 IDENTITY를 사용하면 Hibernate에서 [batch insert를 비활성화](https://www.baeldung.com/jpa-hibernate-batch-insert-update){: target="_blank"} 하기 때문입니다.

IDENTITY 전략을 사용하는 경우, 각 insert 연산을 개별적으로 수행하고 각각에 대해 생성된 ID 값을 확인해야 합니다. 이러한 요구 사항은 batch insert의 특성과 충돌합니다.

> **<font color='dodgerblue'>따라서, Hibernate는 IDENTITY 전략을 사용하는 경우 JDBC 수준에서의 batch insert를 비활성화합니다.</font>**

### JdbcTemplate.batchUpdate 사용하여 bulk insert

bulk insert를 사용하는 방법으로는 id 전략을 변경하는 방법이 있지만, 이는 테이블 변경이 필요하고 이미 진행 중인 프로젝트에 적용하기 어렵습니다.

그래서 대안으로 JdbcTemplate를 사용하여 bulk Insert를 사용할 수 있습니다.

```java
String sql = "INSERT INTO my_table (column1, column2) VALUES (?, ?)";

jdbcTemplate.batchUpdate(sql, new BatchPreparedStatementSetter() {
    public void setValues(PreparedStatement ps, int i) throws SQLException {
        MyObject object = myObjects.get(i);
        ps.setString(1, object.getColumn1());
        ps.setString(2, object.getColumn2());
    }

    public int getBatchSize() {
        return myObjects.size();
    }
});
```

1. 먼저 실행할 SQL 문을 작성합니다. 이 SQL 문은 PreparedStatement에 사용될 수 있는 파라미터를 포함할 수 있습니다.
2. BatchPreparedStatementSetter 구현: BatchPreparedStatementSetter 인터페이스를 구현하여 배치 처리할 각 항목에 대한 PreparedStatement를 설정합니다.
  - setValues(PreparedStatement ps, int i)
    - 각 배치 항목에 대한 PreparedStatement 설정을 담당합니다. 여기서 i는 현재 처리 중인 항목의 인덱스입니다.
  - getBatchSize()
    - 배치 처리할 항목의 총 개수를 반환합니다.
3. JdbcTemplate의 batchUpdate 메소드를 호출하며, SQL 문과 BatchPreparedStatementSetter 구현체를 인자로 제공합니다.

<br>

<hr style="height: 2px; border: none; background-color: white;" />

## Reference 

- <https://dkswnkk.tistory.com/682>{: target="_blank"}
- <https://velog.io/@blackbean99/SpringBoot-Bulk-Insert-%EC%95%8C%EC%95%84%EB%B3%B4%EA%B8%B0-Insert-%EC%BF%BC%EB%A6%AC-%EC%B5%9C%EC%A0%81%ED%99%94>{: target="_blank"}
