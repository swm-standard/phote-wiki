# BaseTimeEntity

각 Entity에서 공통적으로 사용되는 Jpa Auditing 사용 필드들을 `BaseTimeEntity` 으로 분리 시켰습니다.


```Kotlin
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

