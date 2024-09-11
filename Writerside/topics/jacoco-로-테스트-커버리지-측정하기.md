# jacoco 로 테스트 커버리지 측정하기

테스트 코드 커버리지 측정을 위해 jacoco  라이브러리를 사용해봅시다.

#### gradle task 구성
`./gradlew test`  시 일반적인 테스트만 실행되도록
`./gradlew jacocoTestReport`  로 실행 시 기존의 테스트 실행 + jacoco 테스트 레포트를 발행하도록
구현할 예정입니다.

## jacocoTestReport 태스크 작성

```Gradle
tasks.jacocoTestReport {

    dependsOn(tasks.test)

    reports {
        html.required.set(true)
        xml.required.set(true)
        csv.required.set(false)

        html.outputLocation = file(project.layout.buildDirectory.dir("jacoco/index.html"))
        xml.outputLocation = file(project.layout.buildDirectory.dir("jacoco/index.xml"))
    }

    val qDomains = emptyList<String>().toMutableList()

    for (c in 'A'..'Z') {
        val qPattern = "**/*Q$c*"
        qDomains.add("$qPattern*")
    }

    val excludes =
        listOf(
            "**/service/*",
            "**/dto/*",
            "**/controller/*",
            "**/external/*",
            "**/repository/*",
            "**/common/*",
            "**/*PhoteApplication*",
            "**/entity/*RefreshToken*",
        ) + qDomains

    classDirectories.setFrom(
        sourceSets["main"].output.asFileTree.matching {
            exclude(excludes)
        },
    )
    finalizedBy("jacocoTestCoverageVerification")
}
```

- jacocoTestReport 를 생성하는 태스크입니다. 
- 이때  dependsOn(tasks.test)  로 test  태스크를 선제적으로 실행할 것을 명시해줘야 테스트 레포트에 결과가 제대로 표시됩니다.
레포트 형식은 html , xml , csv  가 있고 아래와 같이 필요한 것만 true  로 세팅할 수 있습니다.
  - ```
    html.required.set(true)
      xml.required.set(true)
      csv.required.set(false)
    ```

- 레포트가 생성될 위치를 지정해줄 수 있습니다.
  - ```
    html.outputLocation = file(project.layout.buildDirectory.dir("jacoco/index.html"))
    xml.outputLocation = file(project.layout.buildDirectory.dir("jacoco/index.xml"))
    }
    ```
- 나머지 하단은 레포트에 포함하지 않을 패키지나 클래스를 제외하는 코드입니다. 
  - queryDSL 을 사용하여 Q~ 도메인이 자동 생성되므로 이를 제외해주고
    또 모든 비즈니스 로직을 엔티티 클래스 내부에 뒀으므로 엔티티 외의 클래스들은 모두 제외하도록 했습니다.


> **⚠️ 근데... 자바의 lombok 에서의 setter, getter 은 명시적으로 제외해줄 수 있던데 코틀린의  프로퍼티 같은 경우는 어떻게 커버리지 측정에서 제외해야하는지 모르겠습니다...... help me...**


- 마지막으로 `jacocoTestReport`  태스크가 끝난 뒤에는 `jacocoTestCoverageVerification` 태스크로 마무리되도록 해서 테스트 커버리지 룰을 체크합니다.


## jacocoTestCoverageVerification  태스크 작성

```Gradle
tasks.jacocoTestCoverageVerification {

    violationRules {
        rule {
            element = "CLASS"

            limit {
                counter = "BRANCH"
                value = "COVEREDRATIO"
                minimum = 0.50.toBigDecimal()
            }

            limit {
                counter = "METHOD"
                value = "COVEREDRATIO"
                minimum = 0.40.toBigDecimal()
            }

            val qDomains = emptyList<String>().toMutableList()

            for (c in 'A'..'Z') {
                val qPattern = "**.*Q$c*"
                qDomains.add("$qPattern*")
            }

            excludes =
                listOf(
                    "**.service.*",
                    "**.dto.*",
                    "**.controller.*",
                    "**.external.*",
                    "**.repository.*",
                    "**.common.*",
                    "**.*PhoteApplication*",
                    "**.entity.*RefreshToken*",
                ) + qDomains
        }
    }
}
```

- 테스트 커버리지를 얼마나 달성시켜야할지 말 그대로 **“룰"** 을 체크하는 태스크입니다.
- `counter = “BRANCH” , “METHOD”` 등으로 커버지리 별 최소 퍼센티지를 지정할 수 있습니다.
  - 이때 지정한 퍼센티지에 도달하지 못 할 경우 BUILD FAILED  로 처리됩니다. 
- 레포트를 생성할 때와 마찬가지로 커버리지 체크에서 제외시킬 패키지나 클래스 등을 명시할 수 있습니다.


>✅ 이때 중요한 점!!!!!
>
>레포트 생성 시와는 다르게 여기에서 exclude 항목을 지정할 때는 경로 형식이 아니라 패키지 형식처럼 @@@.@@@.@@  과 같이 점으로 구분합니다!!!!
이 부분을 몰라서 헛짓을 많이 했구요....



## 결과

`./gradlew jacocoTestReport`  로 태스크를 실행하여 jacoco 레포트를 생성합니다.


![스크린샷 2024-09-11 오후 6.47.04.png](스크린샷_2024-09-11_오후_6.47.04.png)


element 에서 한 부분을 클릭해보면 아래와 같이 메서드 별 커버리지를 확인할 수 있습니다.


![스크린샷 2024-09-11 오후 6.47.14.png](스크린샷_2024-09-11_오후_6.47.14.png)