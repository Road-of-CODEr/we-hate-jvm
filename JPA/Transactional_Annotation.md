# JPA에서 @Transactinoal에 대한 궁금증

JPA는 트랜잭션을 기반으로 사용됩니다. 그래서 JPA를 사용하면 `@Transactional`을 잘 사용해야 합니다.

제가 생각하는 국룰은 Service 레이어에서 저장, 수정, 삭제하는 로직에는 `@Transactional` 을 붙이고, 조회하는 로직에서는 `@Transactional(readOnly = true)` 를 붙입니다.

그런데 왜 그렇게 붙여야할까요?

## em.persist()와 em.find() 그리고 트랜잭션

간단한 문제를 내볼게요. 아래의 테스트 코드는 통과가 될까요? (Post는 엔티티로 content 필드만 가지고 있습니다.)

```java
@SpringBootTest
class PostServiceTest {

    @PersistenceContext
    private EntityManager em;

    @Test
    public void persistTest() {
        Post post = Post.builder()
            .content("글 입니다")
            .build();
        em.persist(post);
    }
}
```

위 코드는 돌려보면 단순히 트랜잭션이 없기 때문에 영속화되지 않습니다. 그래서 아래와 같은 `TransactionRequiredException` 예외가 나옵니다.

```java
No EntityManager with actual transaction available for current thread - cannot reliably process 'persist' call
javax.persistence.TransactionRequiredException: No EntityManager with actual transaction available for current thread - cannot reliably process 'persist' call
```

그럼 아래와 같은 테스트는 어떨까요?

```java
@SpringBootTest
class PostServiceTest {

    @PersistenceContext
    private EntityManager em;

    @Autowired
    private PostRepository postRepository;

    private Post savedPost;

    @BeforeEach
    void setup() {
        Post post = Post.builder()
            .content("글 입니다")
            .build();
        savedPost = postRepository.save(post);
    }

    @Test
    void findTest() {
        em.find(Post.class, savedPost.getId());
    }
}
```

통과합니다. 왜일까요?

기본 키로 가져오는 `em.find()`는 단순히 영속성 컨텍스트에 있는 값을 꺼내오는 메서드로 트랜잭션을 필요로 하지 않기 때문입니다. (`em.find()`에 락을 건다면 그때는 트랜잭션이 필요합니다)

그러면 또 궁금증이 듭니다. `em.find()` 를 했을 때는 트랜잭션을 안 거는 것이 좋을까? 아닙니다. 트랜잭션을 아예 걸지 않으면 모든 SELECT 쿼리마다 commit을 하기 때문에 성능이 떨어집니다. 트랜잭션을 건다면 마지막에 한 번만 commit을 하기 때문에 좀 더 효율적이겠죠? 이외에도 작업을 트랜잭션 단위로 나눈다는 트랜잭션 자체의 장점이 있습니다.

그런데 여기서 의심되는 부분이 있습니다. 트랜잭션을 따로 주지 않았는데 `PostRepository.save()`를 실행했을 때는 왜 `TransactionRequiredException` 가 나오지 않았을까요?

`SimpleJpaRepository` 에 가보면 클래스에 `@Transactional(readOnly = true)` 가 있고 `save` 메서드에는 `@Transactional`이 있습니다.

```java
@Transactional
@Override
public <S extends T> S save(S entity) {

	if (entityInformation.isNew(entity)) {
		em.persist(entity);
		return entity;
	} else {
		return em.merge(entity);
	}
}
```

`JpaRepository`를 상속한 `PostRepository`에서 `save()`를 사용하면 `@Transactional`이  이미 있기때문에 사용가능했던 겁니다!

## @Transactional(readOnly = true)의 동작 원리

보통 Service 레이어에서 조회메서드가 있다면 readOnly 옵션을 true로 설정하게 됩니다. readOnly 옵션을 true로 설정하면 뭐가 좋을까요?

`@Transactional`의 readOnly 옵션을 true로 설정하더라도 실제 데이터베이스의 트랜잭션을 read-only 트랜잭션으로 생성할 지 여부는 데이터 계층 프레임워크의 구현 여부에 따라 달라질 수 있습니다.

JPA의 구현체인 Hibernate의 경우, 조회만 한다고 했을 때 DB에 반영할 것이 없다고 판단합니다. 트랜잭션의 commit 시점에 실행하는 flush의 FlushMode를 Manual로 변경해서 dirty checking을 하지 않게 됩니다.

실제로 dirty checking을 하기 위해서 내부적으로 객체를 2개(스냅샷)를 만들어서 비교하는데 readOnly 옵션을 true로 주면 dirty checking을 할 필요가 없기 때문에 스냅샷을 따로 만들지 않습니다.

이외에도 JDBC Connection에도 최적화가 됩니다.

## 테스트코드에 @Transactional 사용 시 주의할 점

보통 테스트를 작성할 때 `@SpringBootTest`를 사용합니다. `@SpringBootTest`는 애플리케이션에 대한 모든 빈, 설정을 가져와서 테스트합니다.

jpa 패키지에 `@DataJpaTest`라는 게 있습니다. `@DataJpaTest`는 JPA 관련 설정만 로드하기에 JPA에 관련된 부분만 주입해서 사용할 수 있는 특징을 가지고 있습니다.

그런데 `@DataJpaTest`를 보시면 `@Transactional` 이 있습니다! 이게 왜 문제일까요?

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@BootstrapWith(DataJpaTestContextBootstrapper.class)
@ExtendWith(SpringExtension.class)
@OverrideAutoConfiguration(enabled = false)
@TypeExcludeFilters(DataJpaTypeExcludeFilter.class)
@Transactional // 이 친구!
@AutoConfigureCache
@AutoConfigureDataJpa
@AutoConfigureTestDatabase
@AutoConfigureTestEntityManager
@ImportAutoConfiguration
public @interface DataJpaTest {
    //...
}
```

`@Target(ElementType.TYPE)` 이므로 클래스 전체에 적용됩니다. `@Transactional`이 있기 때문에 모든 테스트 메서드에 `@Transactional`이 기본으로 들어가게 됩니다. 

그래서 프로덕션 코드(Service 레이어)에서 `@Transactional`이 없어도 테스트코드는 통과하는 일이 일어납니다.

```java
@Service
@RequiredArgsConstructor
public class CustomPostService {

    private final PostRepository postRepository;

    // @Transactional
    public void accessPost(Long id) {
        Post post = postRepository.findById(id).orElseThrow(IllegalArgumentException::new);
        post.getComments().get(0).getContent(); // 지연로딩을 위한 코드!
    }
}
```

```java
@DataJpaTest
@Import(CustomPostService.class)
class PostServiceTest {

    @Autowired
    private CustomPostService postService;

    @Autowired
    private PostRepository postRepository;

    private Post savedPost;

    @BeforeEach
    void setup() {
        Post post = Post.builder()
            .content("글 입니다")
            .build();
        Comment comment = Comment.builder()
            .content("댓글입니다")
            .build();
        post.addComment(comment);

        savedPost = postRepository.save(post);
    }

    @DisplayName("프로덕션에 @Transactional을 붙이지 않았음에도 테스트가 통과된다!")
    @Test
    void accessCommentTest() {
        assertThatCode(() -> postService.accessComment(savedPost.getId()))
            .doesNotThrowAnyException();
    }
}
```

즉, 프로덕션 코드는 에러가 터지는데 테스트는 통과되는 아이러니한 상황이 발생하게 됩니다.

이런 상황을 방지하기 위해 테스트 코드에 `@Transactional` 이 있는지 살펴보고 있다면 없애는 것이 더 나은 테스트코드 작성법이라고 생각합니다.

## 요약

> SimpleJpaRepository에는 @Transactional(readOnly = true) 가 있고 save 메서드에 @Transactional이 있다  

> @Transactional(readOnly = true)를 사용하면 dirty checking을 하지 않는다  

> 테스트코드에 @Transactional이 적용됐는지 살펴보고 가능하면 없애자

## 출처

- 김영한님의 JPA 강의들
- [https://docs.spring.io/spring-framework/docs/5.0.x/spring-framework-reference/data-access.html](https://docs.spring.io/spring-framework/docs/5.0.x/spring-framework-reference/data-access.html)
- [https://www.inflearn.com/questions/7185](https://www.inflearn.com/questions/7185)
- [https://kwonnam.pe.kr/wiki/springframework/transaction](https://kwonnam.pe.kr/wiki/springframework/transaction)
