# k8s 와 함께하는 부하테스트

## 앱을 만들었으니 부하테스트를 해보자
https://github.com/hanhunh89/run_miniboard_with_k8s.git 에서 작은 게시판을 만들었다.<br>
부하테스트를 해보자. 

# 부하가 없을때의 반응.
게시판에 간단한 이미지를 업로드하고 리퀘스트를 보내보자. 
```
curl -o /dev/null -s -w "HTTP status : %{http_code}  response time: %{time_total}\n" http://35.229.58.204/miniboard/post/3
```
이 명령어를 사용하면 http 응답코드와 응답시간을 알 수 있다.<br>
```
HTTP status : 200  response time: 2.592030
HTTP status : 200  response time: 2.782813
HTTP status : 200  response time: 0.155669
HTTP status : 200  response time: 0.162197
HTTP status : 200  response time: 2.706040
HTTP status : 200  response time: 0.179227
HTTP status : 200  response time: 0.143350
HTTP status : 200  response time: 0.157710
HTTP status : 200  response time: 0.14128
```
pod에서 tomcat이 구동하고 첫번째 패킷을 응답하는데는 2.7초 정도가 소요된다. <br>
나는 pod를 세개 만들었으니, 2.7초가 소요된 응답이 세번 보인다.<br>
이후 다른 웹브라우저로 접속해도 0.2초 미만의 응답시간을 보여준다.<br>
이것은 cache에 영향을 받기 때문이다. <br>
그렇기 때문에 pod를 구동하고 나서 실제 서비스를 하기 전에 적당한 부하를 가해서 cache를 만들어주어야 한다.<br>

# auto scaling 설정
부하에 따라 pod가 수평으로 확장되도록 hpa를 설정하자. <br>
tomcat deploy를 생성한다. 
```

# tomcat-deploy.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tomcat-deploy
spec:
  replicas: 3
  selector:
    matchLabels:
      app: tomcat
  template:
    metadata:
      labels:
        app: tomcat
    spec:
      initContainers:
       - name: git-clone-init-container
         image: alpine/git:latest
         command: ['git', 'clone', 'https://github.com/hanhunh89/run_miniboard_with_k8s.git', '/git-repo']
         volumeMounts:
         - name: git-repo-volume
           mountPath: /git-repo
       - name: get-key-init-container  # if you have key.json in git, you don't need it.
         image: alpine/git:latest
         command: ['sh', '-c', 'git clone https://{github token}@github.com/hanhunh89/keyfile.git /git-repo2'] #cloud storage key file pulling
         volumeMounts:
         - name: git-repo-volume-two
           mountPath: /git-repo2
      containers:
      - name: tomcat-container
        image: tomcat:9
        resources:
          requests:
            memory: "100Mi"
            cpu: "50m"
        volumeMounts:
        - name: git-repo-volume
          mountPath: /usr/local/tomcat/webapps
        - name: git-repo-volume-two
          mountPath: /usr/local/tomcat/keys
      volumes:
      - name: git-repo-volume
        emptyDir: {}
      - name: git-repo-volume-two
        emptyDir: {}
```
이때 containers.resources를 설정하지 않으면 FailedGetResourceMetric에러가 발생한다. <br>
metric server가 cpu 사용율을 불러오지 못하기 때문이다. <br>
<br>
다음으로 hpa object를 만들자
```
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: tomcat-hpa  # HPA의 이름
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: tomcat-deploy  # HPA가 조절할 대상 배포의 이름
  minReplicas: 1  # 최소 복제본 수
  maxReplicas: 10  # 최대 복제본 수
  metrics:
  - type: Resource  # 사용할 메트릭 유형
    resource:
      name: cpu  # CPU 사용률을 기반으로 스케일링
      target:
        type: Utilization  # 이용률로 설정
        averageUtilization: 50  # 평균 CPU 사용률을 50%로 유지하도록 조절
```
```
C:\Users\hanhu\my>kubectl get hpa
NAME         REFERENCE                  TARGETS         MINPODS   MAXPODS   REPLICAS   AGE
tomcat-hpa   Deployment/tomcat-deploy   <unknown>/50%   1         10        0          10s
```
hpa 정보에서 현재 target의 cpu 사용율을 볼 수 있다.<br>
처음에 unknown으로 나오다가 곧 사용율이 표시된다.<br>
```
C:\Users\hanhu\my>kubectl get hpa
NAME         REFERENCE                  TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
tomcat-hpa   Deployment/tomcat-deploy   7%/50%    1         10        3          74s
```

## 패킷이 빠지는데요...?
hpa를 설정한 후에 다른 서버에서 부하를 만들어보자
```
#!/bin/bash
# test.sh
while true; do
   curl -o /dev/null -s -w "HTTP status : %{http_code}  response time: %{time_total}\n" http://35.229.58.204/miniboard/post/3
   sleep 0.01
done
```
```
./test.sh
```
test.sh 파일을 실행하면 부하를 일으킨다. 
```
status : 200  response time: 0.363456
HTTP status : 200  response time: 0.311612
HTTP status : 200  response time: 0.277864
HTTP status : 200  response time: 0.285045
HTTP status : 200  response time: 0.287120
HTTP status : 200  response time: 0.186463
HTTP status : 200  response time: 0.264364
HTTP status : 200  response time: 0.299233
HTTP status : 200  response time: 0.274094
HTTP status : 200  response time: 0.448884
HTTP status : 200  response time: 12.713994  # 12초의 시간이 걸린다. 
HTTP status : 200  response time: 0.407350
HTTP status : 200  response time: 0.221831
HTTP status : 200  response time: 12.060001  # 12초의 시간이 걸린다. 
HTTP status : 200  response time: 0.188232
HTTP status : 200  response time: 4.872967
HTTP status : 200  response time: 2.593327
HTTP status : 200  response time: 0.200095
HTTP status : 200  response time: 12.279367 # 12초의 시간이 걸린다. 
HTTP status : 200  response time: 0.168284
HTTP status : 200  response time: 0.154366
```
그런데 문제가 발생했다. <br>
중간중간 12초의 응답시간을 보이는 패킷이 생긴다.<br>
pod에서 tomcat이 다 올리오지 않았는데 loadbalaner가 해당 pod로 패킷을 보내기 때문이다.<br>
12초면 패킷이 누락되었다고 판단해야 한다. 단 하나의 패킷도 놓치지 않기 위해 추가적인 설정이 필요하다. 

## probe를 이용한 pod 상태 확인
probe에는 startup, live, readiness가 있다. <br>
* 참고 : https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/ <br><br>
pod가 준비된 상태에서 서비스를 시행하기 위하여, readiness probe를 사용해보자<br>
readiness probe를 사용하면, probe가 준비된 상태를 정의할 수 있다. <br>
우리가 원하는 상태를 충족해야지만, loadbalancer에서 pod에 패킷을 보낸다<br>
probe를 적용하기 위해 tomcat-deploy.yaml을 수정하자
```
# tomcat-deploy.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tomcat-deploy
spec:
  replicas: 3
  selector:
    matchLabels:
      app: tomcat
  template:
    metadata:
      labels:
        app: tomcat
    spec:
      initContainers:
       - name: git-clone-init-container
         image: alpine/git:latest
         command: ['git', 'clone', 'https://github.com/hanhunh89/run_miniboard_with_k8s.git', '/git-repo']
         volumeMounts:
         - name: git-repo-volume
           mountPath: /git-repo
       - name: get-key-init-container  # if you have key.json in git, you don't need it.
         image: alpine/git:latest
         command: ['sh', '-c', 'git clone https://{github token}@github.com/hanhunh89/keyfile.git /git-repo2'] #cloud storage key file pulling
         volumeMounts:
         - name: git-repo-volume-two
           mountPath: /git-repo2
      containers:
      - name: tomcat-container
        image: tomcat:9
        resources:
          requests:
            memory: "100Mi"
            cpu: "50m"
        readinessProbe:   # redinessProbe를 추가했다.
          httpGet:        # http request를 통해 ready 상태를 확인한다. 
            path: miniboard/post/4  # 특정 페이지에 http request를 보낸다.
            port: 8080
          initialDelaySeconds: 10   # container가 준비되고 10초 뒤에 요청을 보낸다.
          periodSeconds: 5       # 매 5초마다 요청을 보낸다. 
        volumeMounts:
        - name: git-repo-volume
          mountPath: /usr/local/tomcat/webapps
        - name: git-repo-volume-two
          mountPath: /usr/local/tomcat/keys
      volumes:
      - name: git-repo-volume
        emptyDir: {}
      - name: git-repo-volume-two
        emptyDir: {}
```
conaters.readinessProbe를 추가했다.
이제 deployment를 적용해보자.
```
kubectl apply -f tomcat-deploy.yaml
```

이제 다른 서버에서 부하를 발생시키자. 
```
#!/bin/bash
# test.sh
while true; do
   curl -o /dev/null -s -w "HTTP status : %{http_code}  response time: %{time_total}\n" http://35.229.58.204/miniboard/post/3
   sleep 0.01
done
```
결과는 생각보다 놀라운데, 최대 응답시간이 1.2초 정도로 나왔다. <br>
물론 누락되는 패킷은 없다.<br>
readiness를 위한 패킷을 발생시키면서, 캐시를 준비하는 역할도 담당했다.


## pod가 줄어들면서 패킷이 빠지는데요...?
pod가 증가할 때 패킷이 빠지는 현상을 처리했다.<br>
그러면 pod가 줄어들 때도 패킷이 빠질까...?<br>
정답은 yes.<br>
rolling update를 시행하면 old version의 pod가 순차적으로 종료된다.<br>
이때 pod가 서비스를 종료하는 과정에서 loadbalancer가 패킷을 해당 pod에 분배하면 패킷이 빠지게 된다.<br>
```
while true; do
   curl  -s -w "HTTP status : %{http_code}  response time: %{time_total}\n" -L http://35.229.58.204/a
   sleep 0.01
done
```
a 디렉토리에는 index.html파일이 있고 해당 파일은 'a'를 return한다.<br>
rolling update를 시행하며 a를 b로 바꾸었다. <br>
```
HTTP status : 200  response time: 0.172856
a
HTTP status : 200  response time: 0.174321
a
HTTP status : 200  response time: 0.173549
b
HTTP status : 200  response time: 0.171092
b
HTTP status : 200  response time: 0.174109
b
HTTP status : 200  response time: 0.175673
HTTP status : 302  response time: 0.228071
b
HTTP status : 200  response time: 0.173880

HTTP status : 000  response time: 130.550242 # http status 000을 나타낸다. 패킷이 빠진다. 
a
HTTP status : 200  response time: 0.175524
a
HTTP status : 200  response time: 0.172615
a
HTTP status : 200  response time: 0.173165
a
HTTP status : 200  response time: 0.173343
a
HTTP status : 200  response time: 0.173216
a
```
a에서 b로 바뀌는 와중에 패킷이 하나 빠지는 것을 볼 수 있다.<br>
중간에 302를 return하면서 제대로 index.html을 불러오지 못하는 것도 있다. <br>

단 하나의 패킷도 누락되지 않게 하려면 어떻게 할까? <br> 

이를 위해서는 pod가 종료되는 절차를 알아야 한다. <br>
```
1. Pod 삭제 또는 스케일 다운 등 종료 요청이 발생
2. gracefulShutdownPeriod 시작
3. kubelet이 TERM 신호를 보내 graceful shutdown 시도
4. Pod의 상태가 "Terminating"으로 변경
5. endpoints에서 pod가 제거되면서 loadbalancer에서 pod에게 패킷 전송 중단
6. preStop 핸들러 실행
7. gracefulShutdownPeriod가 지난 후 Pod이 완전히 종료됩니다
```
loadbalancer에서 패킷 전송이 중단된 이후에 시간을 두어서<br>
pod가 패킷을 모두 처리하기 위해서는 prestop을 사용해야 한다. <br>
prestop은 아래와 같이 설정한다. 
```
#tomcat2-deploy.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tomcat-deploy
spec:
  replicas: 3
  selector:
    matchLabels:
      app: tomcat
  template:
    metadata:
      labels:
        app: tomcat
    spec:
      terminationGracePeriodSeconds: 120
      initContainers:
       - name: git-clone-init-container
         image: alpine/git:latest
         command: ['git', 'clone', 'https://github.com/hanhunh89/index1.git', '/git-repo/a']
         volumeMounts:
         - name: git-repo-volume
           mountPath: /git-repo
      containers:
      - name: tomcat-container
        image: tomcat:9
        resources:
          requests:
            memory: "100Mi"
            cpu: "50m"
        readinessProbe:   
          httpGet:        
            path: a 
            port: 8080
          initialDelaySeconds: 10 
          periodSeconds: 5       
        volumeMounts:
        - name: git-repo-volume
          mountPath: /usr/local/tomcat/webapps
        lifecycle: # lifecycle 정의
          preStop:   #prestop 정의
            exec:
              command: ["/bin/sh", "-c", "sleep 60"]  # 60초 동안 대기
      volumes:
      - name: git-repo-volume
        emptyDir: {}
```
pod가 terminating 상태로 전환되어 loadbalancer에서 제거되고도<br>
60초 동안 더 동작한다. 이 시간동안 남은 트래픽을 처리할 수 있다. <br>

다시 트래픽을 발생시키고 rolling update를 해보자.
```
while true; do
   curl  -s -w "HTTP status : %{http_code}  response time: %{time_total}\n" -L http://35.229.58.204/a
   sleep 0.01
done
```
이제는 누락되는 패킷이 생기지 않고 rolling update가 정상적으로 실행될 것이다.<br>
hpa을 적용하면 부하가 늘어날때 pod가 줄었다가, 다시 부하가 줄어들면 pod도 줄어든다.<br>
이제 hpa로 인해 pod가 줄어들 때도 패킷이 빠지지 않을 수 있다.<br>

# 이제 트래픽의 로드가 바뀌거나 수시로 발생하는 업데이트에도 시스템 관리자는 발뻣고 잘 수 있다. 

# 끗. 
