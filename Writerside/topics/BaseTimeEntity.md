# BaseTimeEntity로 JPA Auditing 필드 공통 관리하기

각 Entity에서 공통적으로 사용되는 Jpa Auditing 사용 필드들을 `BaseTimeEntity` 으로 분리 시켰습니다.

```kotlin
@EntityListeners(AuditingEntityListener::class)
@MappedSuperclass
abstract class BaseTimeEntity {
    @CreatedDate
    @Column(updatable = false)
    var createdAt: LocalDateTime = LocalDateTime.now()

    @LastModifiedDate
    var modifiedAt: LocalDateTime? = null

    @JsonIgnore
    val deletedAt: LocalDateTime? = null
}
```

- `@MappedSuperclass` : JPA Entity 클래스들이 해당 추상 클래스를 상속할 경우 createDate, modifiedDate를 컬럼으로 인식
- `@EntityListeners(AuditingEntityListener.class)` : 해당 클래스에 Auditing 기능을 포함


> ✅ JPA Auditing 필드들을 엔티티 별로 관리했을 때는 노출 필요가 없는 것들에만 `@JsonIgnore`  처리를 해줬는데 이제 공통으로 관리하므로 일괄 ignore 처리를 하고 필요한 경우에는 dto 에 필드 추가하는 것으로 처리했습니다.


### BaseTimeEntity 상속하기 예시


```kotlin
@Entity
data class Exam(
    val sequence: Int,
    val time: Int,
) : BaseTimeEntity() {
    var id: UUID? = null
```


### JPA Auditing 활성화


```kotlin
@SpringBootApplication
@EnableJpaAuditing
class PhoteApplication

fun main(args: Array<String>) {
    runApplication<PhoteApplication>(*args)
}
```

- 꼭 위와 같이 스프링부트 실행 클래스에 `@EnableJpaAuditing` 어노테이션을 붙여줘야 합니다.
