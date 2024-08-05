# docker 에 jenkins 띄우기

## Docker 에 Jenkins 컨테이너 띄우기


```Bash
sudo service docker start
```


```Bash
docker pull jenkins/jenkins:lts
docker run -p -d 9090:8080 -p 50000:50000 jenkins/jenkins:lts
```

- 포테 서버 ec2에서는 8080 포트가 이미 사용중이므로 9090:8080 사용합니다.


> `-p 9090:8080`   
호스트의 9090포트를 컨테이너의 8080포트로 매핑한다는 의미


```kotlin
2024-07-28 12:23:57.047+0000 [id=31]	INFO	jenkins.InitReactorRunner$1#onAttained: Completed initialization
2024-07-28 12:23:57.077+0000 [id=24]	INFO	hudson.lifecycle.Lifecycle#onReady: Jenkins is fully up and running
2024-07-28 12:23:57.932+0000 [id=47]	INFO	h.m.DownloadService$Downloadable#load: Obtained the updated data file for hudson.tasks.Maven.MavenInstaller
2024-07-28 12:23:57.934+0000 [id=47]	INFO	hudson.util.Retrier#start: Performed the action check updates server successfully at the attempt #1
```


초반에 도커에서 젠킨스 실행 중 위와 같은 부분에서 계속 멈춰있는 오류가 있었는데 아래 커맨드로 포트를 열어주고 나니 정상 작동했습니다.

```kotlin
sudo ufw allow  9090
```


![image](https://res.craft.do/user/full/ef03b946-2e9d-718e-9911-bdafdd78ffa6/doc/CACC90F5-7529-4D90-8246-5B3C88436CB1/5DA425C2-F348-4F5A-B8BB-407D9C3DE491_2/BI68mfTHhnPygaBw8tlgS05PBOa1wiqrnmq8WR8yfygz/%202024-07-28%20%209.28.43.png)

`ec2 api 주소 : 9090`  으로 접속하면 (처음 접속할 경우) 위와 같은 페이지가 뜹니다.
----

### ⚠️ 디스크 공간이 부족하다는 알림이 뜬다면



```bash
불필요한 컨테이너 정리: docker container prune
사용하지 않는 이미지 정리: docker image prune -a
사용하지 않는 볼륨 정리: docker volume prune
사용하지 않는 네트워크 정리: docker network prune
위의 모든 것을 한 번에 정리: docker system prune -a
```


출처: [https://yamea-guide.tistory.com/entry/JENKINS-DOCKER-no-space-left-on-device](https://yamea-guide.tistory.com/entry/JENKINS-DOCKER-no-space-left-on-device) [기타치는 개발자의 야매 가이드:티스토리]

----


### ⚠️ 젠킨스 접속 속도가 느려졌다면



ec2 인스턴스의 주소가 재실행 등의 이유로 바뀌었다면 젠킨스 내에서 초기에 설정한 주소와 달라지므로 접속 속도가 느려진다고 합니다. 따라서 jenkins 환경 설정에서 바뀐 주소대로 재설정해줍니다.

![image](https://res.craft.do/user/full/ef03b946-2e9d-718e-9911-bdafdd78ffa6/doc/CACC90F5-7529-4D90-8246-5B3C88436CB1/715D956F-1C8E-4D80-BD75-211BBEE89FBF_2/jzFjB0pOpOTCBnvc2BjxOa6hfNoZuJyMlPVzIPud3fIz/%202024-08-01%20%204.28.27.png)

[https://jiholine10.tistory.com/414](https://jiholine10.tistory.com/414)


