# Coroutine 으로 비동기 처리하기

### 의존성 추가하기


```kotlin
dependencies {
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core:1.9.0-RC")
}
```


**kotlinx-coroutines-core** 라이브러리는 최신 코틀린 버전을 사용해야한다고 합니다

```kotlin
plugins {
    // For build.gradle.kts (Kotlin DSL)
    kotlin("jvm") version "2.0.0"
    
    // For build.gradle (Groovy DSL)
    id "org.jetbrains.kotlin.jvm" version "2.0.0"
}
```

> ✅ 우리 프로젝트에서도 코틀린 버전을 업그레이드했는데 KtLint 버전과 매치가 안되서 일단 이전 버전으로 그대로 뒀습니다. 정상 작동되긴 하네요..?!


기존의 채점 방식은 다음과 같습니다.

```kotlin
data class GradeExamRequest(
    val time: Int,
    val answers: List<SubmittedAnswerRequest>,
)

data class SubmittedAnswerRequest(
    val questionId: UUID,
    val submittedAnswer: String?,
)
```


제출된 답안 콜렉션에서 question이 객관식 형태라면 단순 == 비교, 주관식 형태라면 ChatGPT 를 호출해 유사도를 측정하여 정오답 판별을 합니다.

## CoroutineScope {id="coroutinescope_1"}


ChatGPT를 호출하는 부분을 비동기적으로 처리하여 속도를 높이고 싶기 때문에 아래와 같이 해당 부분을 `CoroutineScope` 로 감싸서 코루틴 범위로 만듭니다.

```kotlin
CoroutineScope(Dispatchers.IO).launch {
    if (savingAnswer.submittedAnswer == null) {
        savingAnswer.isCorrect = false
    } else {
        when (question.category) {
            Category.MULTIPLE -> savingAnswer.checkMultipleAnswer()
            Category.ESSAY ->
                savingAnswer.isCorrect =
                    async { gradeByChatGpt(savingAnswer) }.await()
        }
    }
    if (savingAnswer.isCorrect) {
        totalCorrect += 1
    }
}
```


### CoroutineScope 이란?

- 코루틴이 실행되는 범위를 정의하는 개념

**역할**


- **코루틴 관리:** 여러 개의 코루틴을 효율적으로 관리하고, 필요에 따라 취소하거나 일시 중지할 수 있습니다.
- **자원 해제:** 스코프가 종료될 때 연관된 자원들을 자동으로 해제하여 메모리 누수를 방지합니다.
- **비동기 작업 관리:** 비동기 작업의 시작과 종료를 명확하게 구분하고, 작업의 결과를 처리할 수 있도록 합니다.

**기능**


- **코루틴 생성:** `launch`, `async` 등의 코루틴 빌더를 사용하여 스코프 내에서 코루틴을 생성합니다.
- **코루틴 취소:** `cancel()` 함수를 호출하여 스코프 내의 모든 코루틴을 취소할 수 있습니다.
- **코루틴 기다리기:** `join()` 함수를 호출하여 스코프 내의 모든 코루틴이 종료될 때까지 기다릴 수 있습니다.

위 코드에서는 코루틴을 생성하기 위해 `launch` 를 사용했습니다.

### CoroutineScope vs GlobalScope

1. **GlobalScope**

   •	특징:

   •	GlobalScope는 코루틴을 전역적으로 실행할 때 사용하는 범위입니다.

   •	앱의 전체 수명 동안 살아있기 때문에, GlobalScope에서 실행된 코루틴은 명시적으로 취소되지 않는 한 계속 실행됩니다.

   •	주로 백그라운드에서 무기한으로 실행되는 작업을 위해 사용됩니다.

   •	주의점:

   •	메모리 누수의 위험이 있습니다. 코루틴을 명시적으로 취소하지 않으면 앱이 종료되기 전까지 계속해서 메모리를 점유할 수 있습니다.

   •	구조화된 동시성(Structured Concurrency)을 위반합니다. 즉, 부모 코루틴의 생명주기에 따라 자동으로 자식 코루틴이 취소되지 않으므로, 관리가 어려워질 수 있습니다.


2. **CoroutineScope**

   •	특징:

   •	CoroutineScope는 특정 범위 내에서 코루틴을 실행하기 위한 범위입니다.

   •	구조화된 동시성을 제공하여, 코루틴이 속한 스코프가 종료될 때 모든 자식 코루틴도 자동으로 취소됩니다.

   •	일반적으로 CoroutineScope는 lifecycleScope, viewModelScope 등 특정 라이프사이클에 바인딩되어 사용됩니다.

   •	장점:

   •	메모리 누수 없이 코루틴을 안전하게 관리할 수 있습니다.

   •	명시적으로 취소할 필요 없이, 스코프의 종료 시점에 자동으로 취소되므로 관리가 용이합니다.

처음에는 무지성으로 **GlobalScope** 로 설정하고 실행했는데 정상작동하긴 했습니다. 하지만 어플리케이션 수명동안 코루틴이 계속 실행된다는 점을 알고 **CoroutineScope**로 전환했습니다. **GlobalScope**는 보통 절대로 취소되지 않는 백그라운드 작업이 필요할 때 사용한다고 합니다.

### Dispatchers.IO 는 뭔가요?


코루틴이 실행될 **스레드**를 지정하는 역할을 합니다. 즉, 코루틴이 어떤 환경에서 실행될지를 결정하는 요소입니다.

여기에선 ChatGPT api와 통신하는 부분을 코루틴으로 실행할 것이므로 `Dispatchers.IO`  디스패쳐를 사용했습니다.

| **종류**                                | **설명**                                                                 |
| ------------------------------------- | ---------------------------------------------------------------------- |
| Dispatchers.Main                      | 안드로이드 앱의 메인 스레드(UI 스레드)에서 실행됨 UI 업데이트와 같은 작업에 사용됨                      |
| Dispatchers.IO | 비동기 I/O 작업(네트워크 통신, 파일 입출력 등)에 최적화된 스레드 풀에서 실행됨 CPU를 많이 사용하지 않는 작업에 적합 |
| Dispatchers.Default                   | 일반적인 CPU 집약적인 작업에 사용되는 스레드 풀에서 실행됨                                     |
| Dispatchers.Unconfined                | 특별한 경우에 사용되며, 코루틴이 시작된 스레드에서 실행됨                                       |

```kotlin
savingAnswer.isCorrect = async { gradeByChatGpt(savingAnswer) }.await()
```


비동기로 처리될 부분은 `async{}` 로 실행합니다. `async` 는 새로운 코루틴을 시작하는 코루틴 빌더로 비동기 연산을  시작한다는 의미를 담고 있습니다. 이 함수는 실행 후에 `Deferred` 인스턴스를 반환하는데 이는 비동기 연산이 완료 된 후 나중에 결과를 주겠다는 약속을 의미합니다. 이를 기다린 후 완성된 결과를 받기 위해서 .await()로 반환받습니다.

저는  `async { gradeByChatGpt(savingAnswer) }`  에서 Deferred<Answer> 이 반환되면, `.await()` 로 언팩하여 Answer 엔티티를 받는 느낌으로 받아들였습니다.

## suspend fun 을 이용하여 비동기 처리 완성하기


외부 API를 호출하는 작업은 일반적으로 시간이 오래 걸리기 때문에 동기적으로 처리한다면, 그 동안 다른 작업이 멈춰버리는 현상이 발생할 수 있습니다.

그래서 외부 API 를 호출하는 부분을 위에서 코루틴 스코프 내로 추가했습니다. 이제 이 호출 부분이 실행 중에 다른 코루틴에게 제어권을 양보할 수 있도록 `suspend fun`  으로 변경해줍니다.

```kotlin
private suspend fun gradeByChatGpt(savingAnswer: Answer): Boolean {
        val chatGptRequest =
            ChatGPTRequest(model, savingAnswer.submittedAnswer!!, savingAnswer.question.answer)

        val chatGPTResponse =
            withContext(Dispatchers.IO) {
                template.postForObject(url, chatGptRequest, ChatGPTResponse::class.java)
            }

        return when (chatGPTResponse!!.choices[0].message.content) {
            "true" -> true
            else -> false
        }
    }
```


#### suspend fun {id="suspend-fun_1"}

- 일시 중단 함수
- 함수 내에 일시 중단 지점을 포함할 수 있는 특별한 함수

→ coroutine은 스레드를 비동기적으로 실행해야하므로 코루틴이 비동기적인 작업을 중단(suspend)하고, 나중에 다시 재개(resume)할 수 있도록 해주기 위해 `suspend`  가 필요합니다.

#### withContext의 용도는?


처음 withContext로 template.postForObject() 를 감싸기 전에는 아래 경고가 보였습니다.

`Possibly blocking call in non-blocking context could lead to thread starvation`

RestTemplate 의 postForObject() 메서드는 blocking call이므로`CoroutineScope(Dispatcher.IO)` 의 스코프에서 실행되면 스레드 starvation이 일어날 수 있다는 것입니다.

일단 인텔리제이 자동 제안에 따라 withContext 로 감싸줬고 경고가 해제되었습니다. 찾아보니 보통은 어떤 코루틴 스코프 내에서 실행 중에, 특정 부분은 **코루틴 스코프에서 지정한 Dispatcher와 다른 context로 실행하고 싶을 때** 새로 context를 지정해주는 용도로 사용한다고 합니다.  ( 이 부분은 더 알아가봐야겠습니다..!)

## 결과


#### 측정 기준

- postman api 요청 소요 시간
- gradeByChatGPT() 메서드 시작 시간, 종료 시간
- 주관식 문제 3개를 채점 요청했습니다. 따라서 ChatGPT에 채점 요청을 3번하게 됩니다.

### 기존: 동기 처리했을 때


![image](https://res.craft.do/user/full/ef03b946-2e9d-718e-9911-bdafdd78ffa6/doc/310965D7-AD48-4E31-B929-B1E94C9250D0/7B399E98-B528-47B6-8D71-BC4629B3DAAF_2/IooKnCmjjtHUvrEQ3iGlIBYD3Lzf1xqNcJ4AzJiGw90z/%202024-08-09%20%2011.13.05.png)

![image](https://res.craft.do/user/full/ef03b946-2e9d-718e-9911-bdafdd78ffa6/doc/310965D7-AD48-4E31-B929-B1E94C9250D0/2A0D71F2-A701-41B0-96EA-373E791AE4D8_2/ayf7S6VLi2ybNZLFvATMyNj4sY20fXW5D6Hr9IFdR34z/%202024-08-09%20%2011.54.38.png)


- 요청 처리 시간은 **4.58 s** 이고 ChatGPT 채점 요청은 차례대로 3번을 진행합니다.

### 변경: 비동기 처리했을 때


![image](https://res.craft.do/user/full/ef03b946-2e9d-718e-9911-bdafdd78ffa6/doc/310965D7-AD48-4E31-B929-B1E94C9250D0/CC208D2F-9900-4703-AC20-68511FD7B13B_2/zizYnFZSZNzx2fSU9wlcWMs86r8iIWVE85xYLQU7Z7Mz/%202024-08-09%20%2011.58.46.png)

![image](https://res.craft.do/user/full/ef03b946-2e9d-718e-9911-bdafdd78ffa6/doc/310965D7-AD48-4E31-B929-B1E94C9250D0/ECF73130-4CF7-4A33-9C3D-44BA37D6CB13_2/s1CZxWDQPEAjP5HDXqeM2W5fmVNYyxVnrNfPOoBNHPgz/%202024-08-09%20%2011.59.00.png)


- 요청 처리 시간은 **700 ms** 이고 ChatGPT 채점 요청은 개별적으로 시작해 비동기적으로 처리됨을 알 수 있습니다.

api 요청 처리가 눈에 띄게 빨라졌음을 확인할 수 있습니다.
