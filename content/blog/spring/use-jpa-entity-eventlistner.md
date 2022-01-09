---
title: Spring JPA의 Entity EventListener 사용
date: 2022-01-09 19:01:64
category: spring
thumbnail: { thumbnailSrc }
draft: false
---

안녕하세요. 오늘은 Spring JPA의 Entity EventListener에 대해서 알아보려고 합니다.

## Entity EventListener

- Entity EventListener의 종류는 @PostLoad, @PrePersist, @PostPersist, @PreUpdate, @PostUpdate, @PreRemove, @PostRemove 가 존재합니다. 문구에서와 같이 Pre(이전), Post(이후), Load, Persist, Update, Remove로 분별해서 보시는 좋을 것 같습니다.
  - `@PostLoad` : 엔티티가 영속성 컨텍스트에 조회된 직후 또는 refresh를 호출한 후 동작됩니다.
  - `@PrePersist` : persist() 메서드를 호출해서 엔터티를 영속성 컨텍스트에 관리하기 직전에 호출됩니다.
  - `@PreUpdate` : flush나 commit을 호출해서 엔티티를 데이터베이스에 수정하기 직전에 호출됩니다.
  - `@PreRemove` : remove() 메서드를 호출해서 엔티티를 영속성 컨텍스트에서 삭제하기 직전에 호출됩니다.
  - `@PostPersist` : flush나 commit을 호출해서 엔티티를 데이터베이스에 저장한 직후에 호출됩니다.
  - `@PostUpdate` : flush나 commit을 호출해서 엔티티를 데이터베이스에 수정한 직후에 호출됩니다.
  - `@PostRemove` : flush나 commit을 호출해서 엔티티를 데이터베이스에 삭제한 직후에 호출됩니다.

### 예제소스

- domain/Student.java

```java
@Data
@NoArgsConstructor
@ToString
@Entity
public class Student {

    @Id
    @GeneratedValue(strategy = GenerationType.AUTO)
    private Long id;

    private String name;
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;

    @PostLoad private void postLoad() {System.out.println("=== postLoad ===");}
    @PrePersist private void prePersist() {System.out.println("=== prePersist ===");}
    @PreUpdate private void PreUpdate() {System.out.println("=== PreUpdate ===");}
    @PreRemove private void PreRemove() {System.out.println("=== PreRemove ===");}
    @PostPersist private void PostPersist() {System.out.println("=== PostPersist ===");}
    @PostUpdate private void PostUpdate() {System.out.println("=== PostUpdate ===");}
    @PostRemove private void PostRemove() {System.out.println("=== PostRemove ===");}
}
```

- repository/StudentRepository.java

```java
public interface StudentRepository extends JpaRepository<StudentRepository, Long> {
}
```

- 테스트소스 : StudentRepositoryTest.java

```java
@SpringBootTest
class StudentRepositoryTest {

    @Autowired
    StudentRepository studentRepository;

    @Test
    void entityListenerTest() {

        System.out.println("======= 1 =========");
        Student student = new Student();
        student.setName("angeloper");
        student.setCreatedAt(LocalDateTime.now());
        student.setUpdatedAt(LocalDateTime.now());

        studentRepository.save(student);

        System.out.println("======= 2 =========");
        student.setName("kskmw2000");
        student.setUpdatedAt(LocalDateTime.now());

        studentRepository.save(student);
        System.out.println("======= 3 =========");

        studentRepository.delete(student);
        System.out.println("======= 4 =========");
    }
}
```

- 테스트 결과

```
======= 1 =========
=== prePersist ===
Hibernate:
    call next value for hibernate_sequence
Hibernate:
    insert
    into
        student
        (created_at, name, updated_at, id)
    values
        (?, ?, ?, ?)
=== PostPersist ===
======= 2 =========
Hibernate:
    select
        student0_.id as id1_2_0_,
        student0_.created_at as created_2_2_0_,
        student0_.name as name3_2_0_,
        student0_.updated_at as updated_4_2_0_
    from
        student student0_
    where
        student0_.id=?
=== postLoad ===
=== PreUpdate ===
Hibernate:
    update
        student
    set
        created_at=?,
        name=?,
        updated_at=?
    where
        id=?
=== PostUpdate ===
======= 3 =========
Hibernate:
    select
        student0_.id as id1_2_0_,
        student0_.created_at as created_2_2_0_,
        student0_.name as name3_2_0_,
        student0_.updated_at as updated_4_2_0_
    from
        student student0_
    where
        student0_.id=?
=== postLoad ===
=== PreRemove ===
Hibernate:
    delete
    from
        student
    where
        id=?
=== PostRemove ===
======= 4 =========

```

## 시용예시

- 이 기능을 보자면 테이블에 데이터를 입력할시 생성일, 수정일을 입력하고, 수정할 시에는 수정일만 변경할 수 있을 것 같다라는 생각을 할 것입니다.
  이에, 이 기능을 다음과 같이 변경이 가능합니다.

### 예제소스

- domain/Student.java

```java
@Data
@NoArgsConstructor
@ToString
@Entity
public class Student {

    @Id
    @GeneratedValue(strategy = GenerationType.AUTO)
    private Long id;

    private String name;
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;

    @PrePersist
    private void prePersist() {
        this.setCreatedAt(LocalDateTime.now());
        this.setUpdatedAt(LocalDateTime.now());
    }

    @PreUpdate
    private void PreUpdate() {
        this.setUpdatedAt(LocalDateTime.now());
    }
}
```

- repository/StudentRepository.java

```java
public interface StudentRepository extends JpaRepository<StudentRepository, Long> {
}
```

- 테스트소스 : StudentRepositoryTest.java

```java
@SpringBootTest
class StudentRepositoryTest {

    @Autowired
    StudentRepository studentRepository;

    @Test
    void entityListenerTest() {
        Student student = new Student();
        student.setName("angeloper");
        studentRepository.save(student);
        studentRepository.findAll().forEach(System.out::println);

        student.setName("kskmw2000");
        studentRepository.save(student);
        studentRepository.findAll().forEach(System.out::println);
    }
}
```

- 테스트 결과

```
Hibernate:
    call next value for hibernate_sequence
Hibernate:
    insert
    into
        student
        (created_at, name, updated_at, id)
    values
        (?, ?, ?, ?)
Hibernate:
    select
        student0_.id as id1_2_,
        student0_.created_at as created_2_2_,
        student0_.name as name3_2_,
        student0_.updated_at as updated_4_2_
    from
        student student0_
Student(id=6, name=angeloper, createdAt=2022-01-09T20:05:16.210, updatedAt=2022-01-09T20:05:16.210)
Hibernate:
    select
        student0_.id as id1_2_0_,
        student0_.created_at as created_2_2_0_,
        student0_.name as name3_2_0_,
        student0_.updated_at as updated_4_2_0_
    from
        student student0_
    where
        student0_.id=?
Hibernate:
    update
        student
    set
        created_at=?,
        name=?,
        updated_at=?
    where
        id=?
Hibernate:
    select
        student0_.id as id1_2_,
        student0_.created_at as created_2_2_,
        student0_.name as name3_2_,
        student0_.updated_at as updated_4_2_
    from
        student student0_
Student(id=6, name=kskmw2000, createdAt=2022-01-09T20:05:16.210, updatedAt=2022-01-09T20:05:16.478)
```

- 보셨나요? 테스트 데이터를 입력할 시 또는 수정할 시에는 setCreatedAt, setUpdatedAt를 사용하지 않았지만, Entity EventListener를 사용하므로써 값이 정상적으로 들어간 부분을 확인할 수 있었습니다. 그러나, 모든 Entity Class에 넣는 것은 Don`t Repeat Yourself(DRY원칙) 및 중복코드가 너무 많이 생기는 부분이 있습니다.

이 부분에 대해서는 다음 섹션을 통해서 중복을 제거할 수 있는 방법을 알아보겠습니다.
