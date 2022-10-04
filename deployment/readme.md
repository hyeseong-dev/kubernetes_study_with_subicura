# Deployment

- ⚡️ 목표 
   Deployment를 이용하여 Pod의 새로운 버전으로 업데이트하고 롤백하는 방법을 알아 봅시다. 
   - 버전 업데이트 
   - 롤백 

디플로이먼트는 쿠버에서 가장 널리 사용되는 오브젝트임. 
pod을 이용하여 update하고 이력을 관리하여 rollback하거나 특정 revision으로 돌아갈 수 있음. 

## 목차 
## Deployment 만들기 
## 버전관리 
## 배포 전략 설정
## 마무리 
## 참고 
## 문제 

## Deployment 만들기 

```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: echo-deploy
spec:
  replicas: 4
  selector:
    matchLabels:
      app: echo
      tier: app
  template:
    metadata:
      labels:
        app: echo
        tier: app
    spec:
      containers:
        - name: echo
          image: ghcr.io/subicura/echo:v1
```

실행 결과 

```bash
❯ k apply -f echo-deployment.yml
deployment.apps/echo-deploy created

❯ k get po,rs,deploy
NAME                               READY   STATUS    RESTARTS   AGE
pod/echo-deploy-77dbb94b69-5xbpj   1/1     Running   0          18s
pod/echo-deploy-77dbb94b69-f58hj   1/1     Running   0          18s
pod/echo-deploy-77dbb94b69-kbtc6   1/1     Running   0          18s
pod/echo-deploy-77dbb94b69-sftjx   1/1     Running   0          18s

NAME                                     DESIRED   CURRENT   READY   AGE
replicaset.apps/echo-deploy-77dbb94b69   4         4         4       18s

NAME                          READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/echo-deploy   4/4     4            4           18s
```


결과 또한 ReplicaSet과 비슷해 보이지만 Deployment의 진가는 Pod을 새로운 이미지로 업데이트할 때 발휘됩니다.

기존 설정에서 이미지 태그만 변경하고 다시 적용합니다.

```yml
❯ k apply -f echo-deployment-v2.yml
deployment.apps/echo-deploy configured
❯ k get po,rs,deploy
NAME                               READY   STATUS    RESTARTS   AGE
pod/echo-deploy-665857f7dd-9wp8m   1/1     Running   0          7s
pod/echo-deploy-665857f7dd-gbxjq   1/1     Running   0          11s
pod/echo-deploy-665857f7dd-sfpbf   1/1     Running   0          11s
pod/echo-deploy-665857f7dd-xkqnk   1/1     Running   0          6s

NAME                                     DESIRED   CURRENT   READY   AGE
replicaset.apps/echo-deploy-665857f7dd   4         4         4       11s
replicaset.apps/echo-deploy-77dbb94b69   0         0         0       17m

NAME                          READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/echo-deploy   4/4     4            4           17m
```

Pod이 모두 새로운 버전으로 업데이트되었습니다.

| Pod 업데이트
엄밀히 말하면 "Pod을 새로운 버전으로 업데이트한다"는 건 잘못된 표현이고, "새로운 버전의 Pod을 생성하고 기존 Pod을 제거한다"가 정확한 표현입니다.


Deployment는 새로운 이미지로 업데이트하기 위해 ReplicaSet을 이용합니다. 버전을 업데이트하면 새로운 ReplicaSet을 생성하고 해당 ReplicaSet이 새로운 버전의 Pod을 생성합니다.

![DeploymentAtReplicaset1][https://user-images.githubusercontent.com/57933835/193773587-ba0da025-ee63-45ca-8019-077fdc5eded0.png]

새로운 ReplicaSet을 0 -> 1개로 조정하고 정상적으로 Pod이 동작하면 기존 ReplicaSet을 4 -> 3개로 조정합니다.

![DeploymentAtReplicaset2][https://user-images.githubusercontent.com/57933835/193773602-c2589d3f-06b8-48ac-807c-c300d366283a.png]

새로운 ReplicaSet을 1 -> 2개로 조정하고 정상적으로 Pod이 동작하면 기존 ReplicaSet을 3 -> 2개로 조정합니다.

![DeploymentAtReplicaset3][https://user-images.githubusercontent.com/57933835/193773625-a9422862-58d3-4024-bab1-97c50dbabdc9.png]

새로운 ReplicaSet을 2 -> 3개로 조정하고 정상적으로 Pod이 동작하면 기존 ReplicaSet을 2 -> 1개로 조정합니다.

![DeploymentAtReplicaset4][https://user-images.githubusercontent.com/57933835/193773667-a61f3115-1c37-4949-a10a-f2b44f25c4cf.png]

최종적으로 새로운 ReplicaSet을 4개로 조정하고 정상적으로 Pod이 동작하면 기존 ReplicaSet을 0개로 조정합니다. 🎉 업데이트 완료!



![DeploymentAtReplicaset5][https://user-images.githubusercontent.com/57933835/193773686-e70a2c4e-8f07-432b-902e-22f8b8a3a5e1.png]

생성한 Deployment의 상세 상태를 보면 더 자세히 알 수 있습니다.

```bash
kubectl describe deploy/echo-deploy
```

실행결과 

```bash
❯ k describe deploy/echo-deploy
Name:                   echo-deploy
Namespace:              default
CreationTimestamp:      Tue, 04 Oct 2022 17:14:27 +0900
Labels:                 <none>
Annotations:            deployment.kubernetes.io/revision: 2
Selector:               app=echo,tier=app
Replicas:               4 desired | 4 updated | 4 total | 4 available | 0 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  25% max unavailable, 25% max surge
Pod Template:
  Labels:  app=echo
           tier=app
  Containers:
   echo:
    Image:        ghcr.io/subicura/echo:v2
    Port:         <none>
    Host Port:    <none>
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      True    MinimumReplicasAvailable
  Progressing    True    NewReplicaSetAvailable
OldReplicaSets:  <none>
NewReplicaSet:   echo-deploy-665857f7dd (4/4 replicas created)
Events:
  Type    Reason             Age    From                   Message
  ----    ------             ----   ----                   -------
  Normal  ScalingReplicaSet  25m    deployment-controller  Scaled up replica set echo-deploy-77dbb94b69 to 4
  Normal  ScalingReplicaSet  8m27s  deployment-controller  Scaled up replica set echo-deploy-665857f7dd to 1
  Normal  ScalingReplicaSet  8m27s  deployment-controller  Scaled down replica set echo-deploy-77dbb94b69 to 3
  Normal  ScalingReplicaSet  8m27s  deployment-controller  Scaled up replica set echo-deploy-665857f7dd to 2
  Normal  ScalingReplicaSet  8m23s  deployment-controller  Scaled down replica set echo-deploy-77dbb94b69 to 2
  Normal  ScalingReplicaSet  8m23s  deployment-controller  Scaled up replica set echo-deploy-665857f7dd to 3
  Normal  ScalingReplicaSet  8m22s  deployment-controller  Scaled down replica set echo-deploy-77dbb94b69 to 1
  Normal  ScalingReplicaSet  8m22s  deployment-controller  Scaled up replica set echo-deploy-665857f7dd to 4
  Normal  ScalingReplicaSet  8m22s  deployment-controller  Scaled down replica set echo-deploy-77dbb94b69 to 0
```

![communication among components][https://user-images.githubusercontent.com/57933835/193774884-a5cee93e-bccb-4d94-8d51-a56a7f76863d.png]

1. Deployment Controller는 Deployment조건을 감시하면서 현재 상태와 원하는 상태가 다른 것을 체크
2. Deployment Controller가 원하는 상태가 되도록 ReplicaSet 설정
3. ReplicaSet Controller는 ReplicaSet조건을 감시하면서 현재 상태와 원하는 상태가 다른 것을 체크
4. ReplicaSet Controller가 원하는 상태가 되도록 Pod을 생성하거나 제거
5. Scheduler는 API서버를 감시하면서 할당되지 않은unassigned Pod이 있는지 체크
6. Scheduler는 할당되지 않은 새로운 Pod을 감지하고 적절한 노드node에 배치
7. 이후 노드는 기존대로 동작

## 버전관리 
Deployment는 변경된 상태를 기록합니다.

```bash 
# 히스토리 확인
kubectl rollout history deploy/echo-deploy

# revision 1 히스토리 상세 확인
kubectl rollout history deploy/echo-deploy --revision=1

# 바로 전으로 롤백
kubectl rollout undo deploy/echo-deploy

# 특정 버전으로 롤백
kubectl rollout undo deploy/echo-deploy --to-revision=2
```

## 배포 전략 설정 

Deployment 다양한 방식의 배포 전략이 있습니다. 여기선 롤링업데이트RollingUpdate 방식을 사용할 때 동시에 업데이트하는 Pod의 개수를 변경해보겠습니다.

- echo-strategy.yml 

```yml 
apiVersion: apps/v1
kind: Deployment
metadata:
  name: echo-deploy-st
spec:
  replicas: 4
  selector:
    matchLabels:
      app: echo
      tier: app
  minReadySeconds: 5
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 3
      maxUnavailable: 3
  template:
    metadata:
      labels:
        app: echo
        tier: app
    spec:
      containers:
        - name: echo
          image: ghcr.io/subicura/echo:v1
          livenessProbe:
            httpGet:
              path: /
              port: 3000
```

Deployment를 생성하고 결과를 확인해봅니다.

```bash 
kubectl apply -f echo-strategy.yml
kubectl get po,rs,deploy

# 이미지 변경 (명령어로)
kubectl set image deploy/echo-deploy-st echo=ghcr.io/subicura/echo:v2

# 이벤트 확인
kubectl describe deploy/echo-deploy-st
```

execution result

```bash
❯ k set image deploy/echo-deploy-st echo=ghcr.io/subicura/echo:v2
deployment.apps/echo-deploy-st image updated
❯ k describe deploy/echo-deploy-st
Name:                   echo-deploy-st
Namespace:              default
CreationTimestamp:      Tue, 04 Oct 2022 17:51:32 +0900
Labels:                 <none>
Annotations:            deployment.kubernetes.io/revision: 2
Selector:               app=echo,tier=app
Replicas:               4 desired | 4 updated | 4 total | 4 available | 0 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        5
RollingUpdateStrategy:  3 max unavailable, 3 max surge
Pod Template:
  Labels:  app=echo
           tier=app
  Containers:
   echo:
    Image:        ghcr.io/subicura/echo:v2
    Port:         <none>
    Host Port:    <none>
    Liveness:     http-get http://:3000/ delay=0s timeout=1s period=10s #success=1 #failure=3
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      True    MinimumReplicasAvailable
  Progressing    True    NewReplicaSetAvailable
OldReplicaSets:  <none>
NewReplicaSet:   echo-deploy-st-696dd684f9 (4/4 replicas created)
Events:
  Type    Reason             Age   From                   Message
  ----    ------             ----  ----                   -------
  Normal  ScalingReplicaSet  94s   deployment-controller  Scaled up replica set echo-deploy-st-c47b6c6b4 to 4
  Normal  ScalingReplicaSet  13s   deployment-controller  Scaled up replica set echo-deploy-st-696dd684f9 to 3
  Normal  ScalingReplicaSet  13s   deployment-controller  Scaled down replica set echo-deploy-st-c47b6c6b4 to 1
  Normal  ScalingReplicaSet  13s   deployment-controller  Scaled up replica set echo-deploy-st-696dd684f9 to 4
  Normal  ScalingReplicaSet  3s    deployment-controller  Scaled down replica set echo-deploy-st-c47b6c6b4 to 0

```

Pod을 하나씩 생성하지 않고 한번에 3개가 생성된 것을 확인할 수 있습니다.

| 배포 전략
| maxSurge와 maxUnavailable의 기본값은 25%입니다. 대부분의 상황에서 적당하지만 상황에 따라 적절하게 조정이 필요합니다.


## 마무리
Deployment는 가장 흔하게 사용하는 배포방식입니다. 이외에 StatefulSet, DaemonSet, CronJob, Job등이 있지만 사용법은 크게 다르지 않습니다.

쿠버네티스에서 컨테이너 생성 방법은 여기까지 알아보고 Pod을 외부로 노출하는 방법을 알아보겠습니다.

