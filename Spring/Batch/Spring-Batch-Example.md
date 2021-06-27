# Spring batch의 실전 코드

spring batch를 실전에 응용해보자

## 예제와 함께하는 설명

Spring Batch는 실전 코드와 같이 가는 것이 중요하다고 생각합니다.
> 물론 저는 아직 실전에서 써먹은 적은 없습니다..ㅎㅎ 곧 사용할 예정

그래서 간단한 샘플과 함께 설명을 시작할까 하는데요. 🔥

간단한 요구사항입니다

* 휴면 멤버를 휴면 상태로 전환하는 배치를 만들어보세요
* 휴면 상태는 배치 돌리는 시작 시간보다 3일 전을 기준으로 합니다 ( 날짜 기준 )

그러면 위 배치프로그램을 구현하러 가볼까요?
<br>
먼저 가장 날짜에 대한 Column이 필요하겠죠? 한번 정의하러 가봅시다

```kotlin
@MappedSuperclass
@EntityListeners(AuditingEntityListener::class)
abstract class AuditableDate {
    @CreatedDate
    @Column(name = "created_at", updatable = false)
    var createdAt: LocalDateTime? = null

    @LastModifiedDate
    @Column(name = "modified_at")
    var modifiedAt: LocalDateTime? = null
}
```

`Jpa Auditing`을 활용한 간단한 날짜 📅 엔티티 객체며,  
위 base 클래스를 토대로 실제 `Member` 엔티티를 만들어보겠습니다.

```kotlin
@Entity
@Table(name = "member_table")
class Member(
    @Id
    @GeneratedValue
    @Column(name = "member_id")
    val id: Long? = null,

    @Column(name = "name")
    val name: String,

    @Column(name = "address")
    val address: String,

    @Column(name = "age")
    val age: Long,

    @Enumerated(EnumType.STRING)
    @Column(name = "dormant")
    var status: MemberStatus = MemberStatus.ACTIVE,

    @ManyToOne
    @JoinColumn(name = "team_id")
    var team: Team? = null,
) : AuditableDate() {
    override fun equals(other: Any?): Boolean {
        if (this === other) return true
        if (other !is Member) return false

        if (id != other.id) return false

        return true
    }

    override fun hashCode() = Objects.hashCode(id)

    override fun toString(): String {
        return "Member(id=$id, name='$name', address='$address', age=$age, team=${team?.id})"
    }
}

enum class MemberStatus {
    ACTIVE, DORMANT
}
```

대략적인 엔티티는 간단합니다.  
회원에 대한 정보들과 상태를 표시하는 `status` 객체가 있습니다.  
<br>
전체적인 준비는 끝났으니 한번 Batch 프로그램을 작성하러 가볼까요?

### Batch Configuration

Batch 🦇 에 대한 기본적인 구조와 테이블 엔티티는 알고 있다고 가정하고 작성하겠습니다

#### Job

먼저 `job`을 정의해봅시다

```kotlin
@Bean
fun dormantMemberCleanJob(): Job = jobBuilderFactory.get(JOB_NAME)
        .start(dormantMemberCleanStep())
        .build()
```

아주 간단합니다. Job 👷 정의는 심플할 수록 좋은 것 같습니다.

#### Step

다음은 `step` 입니다

```kotlin
@Bean
@JobScope
fun dormantMemberCleanStep(
    @Value("#{jobParameters[requestedAt]}") requestedAt: Date? = null,
): Step = stepBuilderFactory.get(STEP_NAME)
        .chunk<Member, Member>(chunkSize)
        .reader(dormantMemberReader())
        .processor(dormantMemberProcessor())
        .writer(dormantMemberWriter())
        .build()
```

`Step` 도 최대한 간단하게 짜봅니다.   
`Flow` 방식으로 여러 flow를 진행하는 방식도 있지만, 굳이 그럴필요는 없습니다  
<br>
다만 여기서 신기하게 보이는 것이 바로 `requestedAt` ⏰ 인데요.  
<br>
Spring Batch 의 기본 엔티티는 `jobParameter` 가 같아버리면, Job을 실행할 수 없습니다.    
별도의 엔티티로 고유값들을 관리하고 있기 때문이죠.  
<br>
그리고 실제로 `Batch Job에 대한 실행이력을 기록` 하기 위해서 언제 날짜로 실행되었는지 기록할 필요도 있습니다  
<br>
`chunkSize` 📘 의 경우에는 writer 가 몇 개 단위로 커밋을 실행할 것인지 결정하게 됩니다.  
해당 size는 실제 운영하면서 튜닝하는 작업이 추가적으로 있을 수 있습니다.

#### Reader Writer Processor

가장 비즈니스 로직의 중추가 되는 친구들입니다

먼저 `Reader` 부터 알아보도록 할게요

```kotlin
@Bean
@StepScope
fun dormantMemberReader(
    @Value("#{jobParameters[dormantDate]}") dormantDate: String? = null
): JpaPagingItemReader<out Member> = JpaPagingItemReaderBuilder<Member>()
        .name(DormantMemberCleanJobConfiguration::dormantMemberReader.name)
        .entityManagerFactory(entityManagerFactory)
        .queryString(
            """
                SELECT m FROM Member m
                WHERE m.${Member::status.name} != :status and m.${Member::modifiedAt.name} <= :dormantDate
                ORDER BY m.${Member::id.name}                
            """.trimIndent() // kotlin을 활용한 쿼리작성
        )
        .parameterValues(
            mapOf(
                "status" to MemberStatus.DORMANT,
                "dormantDate" to LocalDate.parse(
                    dormantDate,
                    DateTimeFormatter.ofPattern("yyyy-MM-dd")
                ).atStartOfDay()
            )
        )
        .entityManagerFactory(entityManagerFactory) // entityManeger 주입
        .maxItemCount(chunkSize) // Paging 처리할 chunkSize
        .build()
```

`PagingItemReader` 와 `CursorItemReader` 2가지가 존재하지만,  
항상 상황에 맞게 선택해야 합니다

왜일까요? [stackOverFlow 링크](https://stackoverflow.com/questions/20386642/spring-batch-which-itemreader-implementation-to-use-for-high-volume-low-laten)
> Cursor는 ThreadSafe 하지 않다. Paging은 병럴 처리에 아주 유용하다.    
> Cursor 는 정해진 connection 에서 지속적으로 데이터를 가져오는 방식이다. 따라서 idle 시간을 고려해야 한다  
> Paging은 paging 기준에 따라서 가져오는 데이터가 다르다. 따라서 기준을 order by 로 설정해야 한다

사실 제가 아는 부분은 여기까지입니다.  
추가적으로 아시는게 있으시면 언제든 환영입니다 😊  
<br>
다음은 `Processor` 입니다

```kotlin
@Bean
fun dormantMemberProcessor() = ItemProcessor<Member, Member> {
        it.status = MemberStatus.DORMANT
        it
    }
```

간단하죠? 꺼내온 데이터를 수정하는 작업밖에 하지 않습니다  

마지막으로 `Writer` 입니다

```kotlin
@Bean
fun dormantMemberWriter() = JpaItemWriterBuilder<Member>()
        .usePersist(false) // entityManager merge 활용
        .entityManagerFactory(entityManagerFactory)
        .build()
```

wrtier 도 간단하죠? ㅎㅎ  

이제 세상에서 제일 어려운 테스트 코드를 작성하러 가볼까요?  

### Batch Test 작성하기  

먼저 저는 기본 설정부터 시작하는 편입니다      
`JobConfiguration`이 많아지면 많아질 수록 공통화된 클래스도 필요한데요.    
하나씩 작성하러 가보겠습니다.!  

```kotlin
@TestConfiguration // 테스트 설정
@EnableAutoConfiguration // spring auto configure 활성화
@EnableBatchProcessing // batch processing 활성화
class TestBatchConfiguration
```

기본적인 설정은 위와 같고, 해당 설정을 `@Import` 해서 사용해보겠습니다  

```kotlin
@Import(TestBatchConfiguration::class) // 만든 class Import
@SpringBatchTest // spring batch 최신 버젼부터는 아주 유용한 어노테이션 입니다
abstract class BaseBatchJobTest {
    @Autowired
    protected lateinit var jobLauncherTestUtils: JobLauncherTestUtils

    @Autowired
    protected lateinit var entityManagerFactory: EntityManagerFactory

    private val entityManager by lazy { entityManagerFactory.createEntityManager() }

    protected fun <T> save(entity: T): T {
        entityManager.transaction.also {
            it.begin()
            entityManager.persist(entity)
            it.commit()
            entityManager.clear()
        }
        return entity
    }

    protected fun <T> saveAll(entities: List<T>): List<T> {
        entityManager.transaction.also {
            it.begin()
            entities.forEach { entity -> entityManager.persist(entity) }
            it.commit()
            entityManager.clear()
        }
        return entities
    }

    protected fun <T> mergeAll(entities: List<T>): List<T> {
        entityManager.transaction.also {
            it.begin()
            entities.forEach { entity -> entityManager.merge(entity) }
            it.commit()
            entityManager.clear()
        }
        return entities
    }

    protected fun deleteAll(tableName: String) {
        entityManager.transaction.also {
            it.begin()
            entityManager.createQuery(
                """
                DELETE FROM $tableName
                """.trimIndent()
            ).executeUpdate()
            it.commit()
        }
    }
}
```

위의 메서드들이 왜 굳이 작성했을까요..? 🤔  
<br>
*사실 저의 생각은 이렇습니다.*  
실제로 `JpaRepository`를 이용해서 데이터를 저장하고 삭제하는 방식을 진행할 수 있습니다.  
<br>
다만 그렇게 진행하게 된다면, 모든 엔티티들마다 `JpaRepository`를 만들어줘야 됩니다.  
문제는 이 `Repository` 들은 실제 **production에서 사용하지 않는 코드** 가 되어버립니다.    
<br>
일반적으로 `batch` 모듈과 `api` 모듈은 분리해서 사용하는 방식을 선택하기 때문이죠.    
<br>
그렇기 때문에 귀찮더라도, 공통화된 속성들을 만들어준 것입니다    
<br>
그러면 모든 준비는 끝났습니다!!

```kotlin
@SpringBootTest(classes = [DormantMemberCleanJobConfiguration::class])
internal class DormantMemberCleanJobConfigurationTest : BaseBatchJobTest() {

    @Autowired
    private lateinit var memberRepository: MemberRepository

    @BeforeEach
    fun setUp() {
        val members = listOf(
            Member(
                name = "member1",
                address = "Seoul",
                age = 11L,
                status = MemberStatus.ACTIVE
            ).apply {
                createdAt = LocalDateTime.now().minusDays(5)
                modifiedAt = LocalDateTime.now().minusDays(5)
            },
            Member(
                name = "member2",
                address = "Seoul2",
                age = 12L,
                status = MemberStatus.ACTIVE
            ).apply {
                createdAt = LocalDateTime.now().minusDays(3)
                modifiedAt = LocalDateTime.now().minusDays(3)
            },
            Member(
                name = "member3",
                address = "Seoul3",
                age = 14L,
                status = MemberStatus.ACTIVE
            ).apply {
                createdAt = LocalDateTime.now().minusDays(1)
                modifiedAt = LocalDateTime.now().minusDays(1)
            }
        )
        saveAll(members)
    }

    @AfterEach
    fun tearDown() {
        deleteAll(Member::class.simpleName!!)
    }

    @Test
    fun `마지막 수정날짜가 2일전보다 이전 Member들의 status를 Dormant 처리한다`() {
        // given
        val jobParameters = JobParametersBuilder()
            .addString(
                "dormantDate", LocalDateTime.now() // parameter 지정
                    .minusDays(2)
                    .format(DateTimeFormatter.ofPattern("yyyy-MM-dd"))
            )
            .addDate("requestedAt", Date())
            .toJobParameters()

        // when
        val result = jobLauncherTestUtils.launchJob(jobParameters) // job실행

        // then
        assertThat(result.status).isEqualTo(BatchStatus.COMPLETED) // job 성공 여부
        val members = memberRepository.findAll()
        assertThat(members).filteredOn { it.status === MemberStatus.DORMANT }.hasSize(2)
        assertThat(members).filteredOn { it.status !== MemberStatus.DORMANT }.hasSize(1)
    }
}
```

총 3개의 Member 를 미리 준비하고 저장합니다.  
1일, 3일, 5일 전에 마지막으로 수정된 날짜 이력이 있군요.  
<br>
해당 Job Batch 🖊 를 돌리게 되면, 총 1명의 Member만 active 상태가 되겠군요.  
<br>
추가적으로 고민할 요소는 있습니다. 대량의 데이터의 경우에는 어떻게 미리 데이터를 준비해야 할까요?  
저도 아직까지 해결방안을 잘 모르겠습니다.. 😢  

좋은 의견이 있으면 남겨주시면 좋을 것 같아요.!!    
<br>

추가 꿀팁.

```yaml
logging.level:
  org.hibernate:
    type: trace
    SQL: debug
```

꼭 설정합시다..꼭!!  
`p6spy` 도 있긴 한데, 너무 장황해져서 저는 안쓰기도 합니다 ㅎㅎ..

## 마무리

전체적으로 구조에 대한 `Deep Dive` 보다는 어떤식으로 테스트 코드를 짜고,   
어떤식으로 설정해야 되는지에 대한 전체적인 설명이었습니다.  
<br>
피드백이나 의견은 언제나 환영이에요.!!