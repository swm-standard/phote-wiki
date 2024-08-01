# GitAction으로 테스트 자동화

GitAction 으로 CI 세팅을 해봅시다.

>GitAction 으로 진행하는 이유 😵‍💫🥹
> 1. jenkins에서 PR Status Check 로 결과를 넘겨주는 방법을 도저히 못찾았습니다. (단순 빌드까지는 가능)
> 2. 어차피 단순 테스트 용도라면 실제 빌드 및 배포와는 관계가 없을듯해서 임시방편으로 GitAction을 택했습니다.
> 3. 추후라도 jenkins로 구현이 가능하다면 꼭 시도해보겠습니다.

### test-automation.yml 분석

```yaml
name: test-automation

on:
    push:
        branches:
            - 'develop'
    pull_request:
        branches:
            - 'develop'

permissions: write-all
```

- `develop` 브랜치에 한해 push 이벤트, PR 이벤트가 발생하는 경우에 해당 gitaction이 실행되도록 합니다.

<procedure>
<step>
✅ permission 설정을 해줘야 아래의 유닛 테스트 결과 코멘트 액션이 잘 실행됩니다. (해당 커맨드가 없을 경우 <format color="Red">403 error </format> 가 발생함)
</step>
</procedure>

```yaml
jobs:
    code-coverage:
        runs-on: ubuntu-latest

        steps:
            - uses: actions/checkout@v3
            - name: Set up JDK 17
              uses: actions/setup-java@v3
              with:
                  java-version: '17'
                  distribution: 'temurin'

            - name: Set up properties
              run: |
                cd src/main
                mkdir resources
                cd resources
                touch application.properties
                echo "${{ secrets.APPLICATION }}" > application.properties
                cat application.properties

              shell: bash

            - name: Grant execute permission for gradlew
              run: chmod +x gradlew
```

<procedure>
<step>
✅ Phote service는 java 17 버전을 사용하므로 꼭 <code>java-version : '17'</code> 로 명시합니다.
</step>
<step>
해당 레포지토리에는 <code>main</code> 디렉토리 아래의 <code>resources</code> 디렉토리부터는 아예 올라가있지 않으므로 디렉토리부터 생성 후에 프로퍼티 작성을 해야합니다.
</step>
<step>
<code>${{ secrets.APPLICATION }}</code>  은 Github Repository secret 변수로 설정돼있습니다.
</step>
<step>
gradlew 에 권한을 부여합니다. (<b>실행권한 (x) 추가 </b>)
</step>
</procedure>


```yaml
            - name: Test with Gradle
              run: ./gradlew clean build test jacocoTestReport

            - name: Publish Unit Test Results
              uses: EnricoMi/publish-unit-test-result-action@v1
              if: ${{ always() }}
              with:
                  files: build/test-results/**/*.xml
```
<procedure>
<step>
빌드 명령어로 빌드를 실행합니다.

추후 테스트 커버리지까지 코멘트로 보고받을 수 있도록 <code>test jacocoTestReport</code> 옵션을 더합니다.
</step>
<step>
빌드 시 <code>build/test-results/**/*.xml</code> 에 저장되는 테스트 결과를 이용하여 유닛테스트 결과를 코멘트로 보고할 수 있도록 합니다. 
</step>
</procedure>