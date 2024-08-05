# Redis로 RefreshToken 구현하기

<show-structure for="chapter,procedure" depth="2"/>

## Redis 설치 및 사용


### Redis docker에 설치하기

1. `docker pull redis`
2. `docker run --name=redis -d -p 6739:6739 redis:latest`


> ✅ redis 기본 포트는 6739번입니다.


### docker-compose 로 Redis 실행하기


처음에는 위와 같이 docker 컨테이너를 직접 생성하였습니다. 그런데 로컬에서 실행하는 서버와 ec2 인스턴스 내에서 docker로 실행되는 redis와 연결하는 것이 불가능하다는 것을 알게 됐습니다. 따라서 로컬에서 개발 및 테스트 용으로 사용할 수 있도록 개발용 `docker-compose-dev.yml`  파일을 생성했습니다.

```yaml
services:
    redis:
        image: redis
        container_name: redis
        volumes:
             - /var/lib/docker/volumes/redis-volume/_data:/data
        ports:
            - "6379:6379"
        hostname: redis
```

>**✅ 알아두면 좋은점**
> 
> 1. docker 컨테이너는 종료하면 내부 데이터가 모두 삭제됩니다. 따라서 redis 를 포함하여 데이터 보존이 필요한 DB 와 같은 용도의 컨테이너에는 볼륨을 설정해주는 것이 좋습니다.
> - docker 컨테이너는 stateless(무상태) 애플리케이션이기 때문입니다. 반대로 DB 는 stateful (유상태) 애플리케이션이므로 볼륨 설정이 필요한 것입니다.
    > → 여기에서는 `redis-volumn` 이라는 이름을 가진 볼륨을 생성하고 컨테이너와 연결했습니다.
> - **Docker Volume** : 컨테이너의 일부로 동작하지만, 컨테이너와 별개의 생애주기를 갖는 독립적인 객체
> 
{style="note"}

### redis 패스워드 설정하기 {id="redis_1"}


❎ 아래는 실제로 구현하지 않았습니다. ❎

이유는 로컬에서 `redis.conf`  를 작성하고 이를 ***실행 시에 컨테이너로 마운트 해줘야하는데*** 개발자 각자 일일이 세팅하는 것이 번거로울 것 같다고 판단했기 때문입니다. 또한 redis 탈취 위험이 있어 배포 시에는 패스워드 설정이 필요할 것 같지만 현재는 로컬에서 개발 및 테스트 용도로만 사용되기 때문에 과감히! 패스워드 세팅은 제외했습니다.

```yaml
        volumes:
             - ./redis.conf:/usr/local/etc/redis/redis.conf
             - 
        ports:
            - "6379:6379"
        command: redis-server /usr/local/etc/redis/redis.conf
```


#### 🙋application.properties 에서 redis.host는 어떻게 표기하면 되나요?


```yaml
#redis
spring.data.redis.host=localhost
spring.data.redis.port=6379
spring.data.redis.password=${{password}}
```

- 위에서 docker-compose로 실행하는 redis 컨테이너도 결국에는 로컬에서 실행되는 것이기 때문에 `localhost` 라고 표시해주면 됩니다.


> 🥸 의문인 점
**  처음  설계할 때는 springboot 실행도 docker-compose에 함께 작성하였어서 host name을 r`edis `로 설정했습니다. 이렇게 실행하면 정상작동돼야할텐데 안됐습니다.. 이유는 ... 모르므로 나중에 찾아보겠습니다.

- ec2 로 배포를 할 때도 기존 docker-compose 파일에 redis 설정을 그대로 복붙하면 될 것 같습니다. (프로퍼티 파일에서 `spring.data.redis.host`  값을 달라질 것)


----

## RefreshToken 로직 설계 과정


처음에는 auth 를 제외한 모든 api 요청 시에 accessToken과 refreshToken 를 모두 받아서 accessToken이 만료됐을 경우 서버 측에서 바로 함께 받은 refreshToken을 이용하여 새로운 accessToken 을 생성할까 했습니다.

그런데 이와 같이 구현할 경우 accessToken 갱신 여부에 따라 responseBody 형식이 달라질 수 있으므로 accessToken 만료 시 갱신 요청을 할 수 있는 api 를 따로 분리하여 구현했습니다.

→ 서비스 이용자 입장에서는 달라질게 없습니다! 단지 클라이언트 측에서 에러를 반환받을 경우 accessToken 갱신 api 를 내부적으로 요청하기만 하면 됩니다

> ⚠️ 단점
> 실제로 잘못된 사용자가 요청해서 403 에러가 반환됐는지, accessToken이 만료됐기 때문인지 구분할 수가 없습니다.
> (근데 이 둘을 구분할 수 있는 구현 방법이 있는지도 잘 모르겠습니다...! ㅎㅎ)


### refreshToken 관련 sequence diagram


<code-block lang="plantuml">
    @startuml
    Actor Client
    Participant Server
    Database MySql
    Database Redis
    
    == 로그인/회원가입 시 ==
    activate Client
    Client -> Server: 로그인/회원가입 요청
    activate Server
    Server -> MySql: 회원 정보 요청
    activate MySql
    MySql --> Server: 회원 정보
    deactivate MySql
    Server -> Redis: refreshToken 저장
    Redis --> Server: refreshToken
    Server --> Client: 회원 정보/accessToken/refreshToken 반환
    deactivate Server
    
    == api 요청 시 accessToken 이 만료된 경우==
    Client -> Server: api 요청
    activate Server
    Server -> Server: accessToken 유효성 체크
    Server --> Client: 403 에러 반환
    deactivate Server
    
    == refreshToken으로 accessToken 갱신==
    Client -> Server:  accessToken 갱신 요청(refreshToken)
    activate Server
    Server -> Redis: refreshToken 조회
    activate Redis
    alt refreshToken이 존재할 경우
    Redis --> Server: refreshToken entity
    Server -> Server: accessToken 갱신
    Server --> Client: accessToken
    else refreshToken이 만료된 경우
    Redis --> Server!!:
    Server --> Client: 401 에러
    end
    
    deactivate Redis
    
    @enduml
</code-block>


----

## CreateToken  메서드 구현 변경


위 다이어그램에서 볼 수 있듯이, **refreshToken**을 발급받는 방법은


1. **로그인/회원가입하기**

뿐입니다.

반면에 **accessToken**을 발급받는 방법은


1. **로그인/회원가입하기**
2. **refreshToken으로 갱신하기**

두가지가 있습니다.

위 두가지 경우에서 생성 메서드를 동일하게 사용하기 위해서 `JwtTokenProvider` 의 `createToken` 을 일부 수정하였습니다.

### 기존


```kotlin
fun createToken(userInfoResponseDto: UserInfoResponse, memberId: UUID): String {
        val now = Date()
        val accessExpiration = Date(now.time + ACCESS_TOKEN_EXPIRATION)

        val accessToken =
            Jwts
                .builder()
                .setSubject(userInfoResponseDto.email)
                .setSubject(memberId.toString())
                .claim("memberId", memberId)
                .setIssuedAt(now)
                .setExpiration(accessExpiration)
                .signWith(key, SignatureAlgorithm.HS256)
                .compact()
        return "Bearer $accessToken"
    }
```


기존에는 8번 라인 memberId와 dto를 모두 인자로 받아서  `.setSubject(userInfoResponseDto.email)`  와 같이 subject를 사용자 email 로 설정했습니다. 하지만 **refreshToken으로 갱신하기** 방법으로 accessToken을 생성할 경우 `userInfoResponsedto` (로그인/회원가입 시에 필요한 dto임) 를 사용하지 않으므로 subject를 memberId 를 사용하는 것으로 변경했습니다.

### 변경 이후


```kotlin
    fun createToken(memberId: UUID): String {
        val now = Date()
        val accessExpiration = Date(now.time + ACCESS_TOKEN_EXPIRATION)

        val accessToken =
            Jwts
                .builder()
                .setSubject(memberId.toString())
                .claim("memberId", memberId)
                .setIssuedAt(now)
                .setExpiration(accessExpiration)
                .signWith(key, SignatureAlgorithm.HS256)
                .compact()

        return "Bearer $accessToken"
    }
```

----
