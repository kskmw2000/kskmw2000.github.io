---
title: Spring JPA의 Entity EventListener 사용-2
date: 2022-01-10 12:01:89
category: spring
thumbnail: { thumbnailSrc }
draft: false
---

어제 JPA의 Entity EventListener에 대해서 활용방법을 알아보았습니다.  
Entity EventListener의 기능을 통해서 모든 테이블에 존재하는 등록일, 수정일을 변경되는 부분을 확인해 보았습니다.  
하지만, 뭔가 꺼림직합니다. 그 이유는 모든 Entity에 다음과 같은 문법을 넣어야 한다는 사실이 좀 아쉽기만 합니다.

```java
// domain/Student.java
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

그럼, 중복을 제거할 수 있는 방법은 없을까요?

EventListener를 사용하면서 제거하는 방법을 한번 살펴보겠습니다.

```java
// listener/Auditable.java
public interface Auditable {

    LocalDateTime getCreatedAt();
    LocalDateTime getUpdatedAt();

    void setCreatedAt(LocalDateTime createdAt);
    void setUpdatedAt(LocalDateTime updatedAt);
}

// listener/MyEntityListner.java
public class MyEntityListner {

    @PrePersist
    public void prePersist(Object o) {
        if(o instanceof Auditable) {
            ((Auditable) o).setCreatedAt(LocalDateTime.now());
            ((Auditable) o).setUpdatedAt(LocalDateTime.now());
        }
    }

    @PreUpdate
    public void preUpdate(Object o) {
        if(o instanceof Auditable) {
            ((Auditable) o).setUpdatedAt(LocalDateTime.now());
        }
    }
}

// domain/Student.java
@Data
@NoArgsConstructor
@ToString
@Entity
@EntityListeners(value = MyEntityListner.class)
public class Student implements Auditable {

    @Id
    @GeneratedValue(strategy = GenerationType.AUTO)
    private Long id;
    private String name;

    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
}
```

- 테스트 코드

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
Student(id=6, name=angeloper, createdAt=2022-01-10T19:51:52.186, updatedAt=2022-01-10T19:51:52.187)
Student(id=6, name=kskmw2000, createdAt=2022-01-10T19:51:52.186, updatedAt=2022-01-10T19:51:52.440)
```

그럼에도 Model의 클래스에 `@EntityListeners(value = MyEntityListner.class)` 구문과 `implements Auditable` 그리고 또한 `createdAt`, `updatedAt`을 계속적으로 넣어준다는 부분이 존재합니다. 이에, Spring의 상속, Auditing 기능을 사용하여 중복코드를 없애는 작업을 해 도록 하겠습니다.

```java
// App.java
@SpringBootApplication
@EnableJpaAuditing    // Spring 전체적으로 Auditing 기능을 사용할 수 있도록 설정하였습니다.
public class App {
    public static void main(String[] args) {
        SpringApplication.run(App.class, args);
    }
}

// domain/BaseEntity.java
// createAt과 updatedAt의 중복 코드를 없애는 작업 및 @createdDate와 @LastModifiedDate를 사용하여 감시가 되도록 합니다.
@Data
@MappedSuperclass       // 상속 받는 클래스의 속성으로 포함하도록 하겠다라는 의미
@EntityListeners(value = AuditingEntityListener.class)
public class BaseEntity {

    @CreatedDate
    @Column(updatable = false)
    private LocalDateTime createdAt;

    @LastModifiedDate
    private LocalDateTime updatedAt;
}

// listener/Auditable.java
// 인터페이스를 통해서 getter/setter Method를 구현하도록 합니다.
public interface Auditable {
    LocalDateTime getCreatedAt();
    LocalDateTime getUpdatedAt();
    void setCreatedAt(LocalDateTime createdAt);
    void setUpdatedAt(LocalDateTime updatedAt);
}

// model/Student.java
@Data
@NoArgsConstructor
@ToString(callSuper = true)
@Entity
public class Student extends BaseEntity implements Auditable {
    @Id
    @GeneratedValue(strategy = GenerationType.AUTO)
    private Long id;
    private String name;
}
```

- 테스트코드

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
Student(super=BaseEntity(createdAt=2022-01-10T20:02:25.649, updatedAt=2022-01-10T20:02:25.649), id=6, name=angeloper)
Student(super=BaseEntity(createdAt=2022-01-10T20:02:25.649, updatedAt=2022-01-10T20:02:25.880), id=6, name=kskmw2000)

```

확인하셨나요?? 이로써 모델객체에 규칙이 생기긴 했지만 필요한 컬럼만 넣으면 되는 효과를 보았습니다. 더불어, 개발자의 경우에는 등록일, 수정일을 신경쓸 필요가 없어졌습니다.

또한, 이 기능을 활용하여 테이블의 이력을 남기는 부분 또한 쉽게 가능하다고 합니다.
이 부분은 다음 부분에서 확인해 보도록 하겠습니다.

> Fastcampus의 스프링 강좌를 보고 정리한 내역입니다.
