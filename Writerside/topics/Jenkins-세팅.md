# Jenkins 세팅

```Bash
sudo service docker start
```


```Bash
docker pull jenkins/jenkins:lts
docker run -p 8080:8080 -p 50000:50000 jenkins/jenkins:lts
```

- 우리 서비스에서는 8080 포트가 이미 사용중이라 8081:8081 사용

![image](https://res.craft.do/user/full/ef03b946-2e9d-718e-9911-bdafdd78ffa6/doc/CACC90F5-7529-4D90-8246-5B3C88436CB1/CDF43CF6-F1D2-4641-A5B4-98D00302E518_2/ikiOxy2w6UiwImbf6IO26hH2Xyh3gyOw3pR9tVLpty8z/%202024-07-28%20%205.51.29.png)


- 실행 후 나오는 초기 패스워드를 복사해둔다.

```kotlin
2024-07-28 12:23:57.047+0000 [id=31]	INFO	jenkins.InitReactorRunner$1#onAttained: Completed initialization
2024-07-28 12:23:57.077+0000 [id=24]	INFO	hudson.lifecycle.Lifecycle#onReady: Jenkins is fully up and running
2024-07-28 12:23:57.932+0000 [id=47]	INFO	h.m.DownloadService$Downloadable#load: Obtained the updated data file for hudson.tasks.Maven.MavenInstaller
2024-07-28 12:23:57.934+0000 [id=47]	INFO	hudson.util.Retrier#start: Performed the action check updates server successfully at the attempt #1
```


여기서 계속 멈춰있는 오류가 있었는데 아래 커맨드로 포트를 열어주고 나니 정상 작동했다.

```kotlin
sudo ufw allow  9090
```

