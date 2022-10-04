# ReplicaSet

- ⚡ 목표️
    1. ReplicaSet ?
    2. ReplicaSet -> Pod을 어떻게 관리?

- 등장배경 
    1. pod만 쓰면 팟에 어떤 문제가 생겼을때 자동으로 복구 되지 않음. 
    2. 즉, 정해진 수 만큼 복제하고 관리하는 것 -> replicaSet

- 목차 
    1. ReplicaSet 만들기
    2. 스케일 아웃
    3. 마무리 
    4. 참고
    5. 문제

## ReplicaSet 만들기
echo-rs.yml
```yml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: echo-rs
spec:
  replicas: 1
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
Pod보다 설정이 조금 복잡하지만, 각 항목을 이해하면 아주 어렵지 않습니다. 일단 생성하고 상세 내용을 봅시다.

```bash
# ReplicaSet 생성
kubectl apply -f echo-rs.yml

# 리소스 확인
kubectl get po,rs

```

실행 결과

```bash
NAME                READY   STATUS    RESTARTS   AGE
pod/echo-rs-tcdwj   1/1     Running   0          61s

NAME                      DESIRED   CURRENT   READY   AGE
replicaset.apps/echo-rs   1         1         1       61s
```

ReplicaSet과 Pod이 같이 생성된 것을 볼 수 있습니다.

![replicaset & pod][https://subicura.com/k8s/build/imgs/guide/replicaset/rs.webp]

|정의|	설명|
|---|---|
|spec.selector	|label 체크 조건|
|spec.replicas	|원하는 Pod의 개수|
|spec.template	|생성할 Pod의 명세|

template을 자세히 보니 이전에 본 Pod 설정 파일과 완전히 동일한 것을 알 수 있습니다.

```yaml
metadata:
  labels:
    app: echo
    tier: app
spec:
  containers:
    - name: echo
      image: ghcr.io/subicura/echo:v1
```

생성된 Pod의 label을 확인해봅니다.

```bash
kubectl get pod --show-labels
```

실행 결과 
```bash
NAME            READY   STATUS    RESTARTS   AGE   LABELS
echo-rs-tcdwj   1/1     Running   0          3m    app=echo,tier=app
```

설정한 대로 app=echo,tier=app label이 보입니다. 그럼 임의로 label을 제거하면 어떻게 될까요?
```bash
# app- 를 지정하면 app label을 제거
kubectl label pod/echo-rs-tcdwj app-

# 다시 Pod 확인
kubectl get pod --show-labels
```

실행결과 
```bash
NAME            READY   STATUS    RESTARTS   AGE   LABELS
echo-rs-tcdwj   1/1     Running   0          3m    tier=app
echo-rs-kv4mh   1/1     Running   0          5s    app=echo,tier=app
```

기존에 생성된 Pod의 app label이 사라지면서 selector에 정의한 app=echo,tier=app 조건을 만족하는 Pod의 개수가 0이 되어 새로운 Pod이 만들어졌습니다.

다시 app label을 추가해봅니다.

```bash
# app- 를 지정하면 app label을 제거
kubectl label pod/echo-rs-tcdwj app=echo

# 다시 Pod 확인
kubectl get pod --show-labels
```

실행 결과 

```bash
NAME            READY   STATUS        RESTARTS   AGE     LABELS
echo-rs-h4q86   1/1     Running       0          4m      app=echo,tier=app
echo-rs-kv4mh   0/1     Terminating   0          2m19s   app=echo,tier=app
```

replicas에 정의한 대로 Pod의 개수를 1로 유지하기 위해 기존 Pod을 제거합니다.

ReplicaSet이 어떻게 동작하는지 살펴봅니다.

![ReplicaSet동작][https://user-images.githubusercontent.com/57933835/193765902-3ebdcd19-842e-491e-a057-2511619a28fd.png]

1. ReplicaSet Controller는 ReplicaSet조건을 감시하면서 현재 상태와 원하는 상태가 다른 것을 체크
2. ReplicaSet Controller가 원하는 상태가 되도록 Pod을 생성하거나 제거
3. Scheduler는 API서버를 감시하면서 할당되지 않은unassigned Pod이 있는지 체크
4. Scheduler는 할당되지 않은 새로운 Pod을 감지하고 적절한 노드node에 배치
5. 이후 노드는 기존대로 동작

ReplicaSet은 ReplicaSet Controller가 관리하고 Pod의 할당은 여전히 Scheduler가 관리합니다. 각자 맡은 역할을 충실히 수행하는 모습이 보기 좋습니다.

## 스케일 아웃
ReplicaSet을 이용하면 손쉽게 Pod을 여러개로 복제할 수 있습니다.

echo-rs-scaled.yml
```yml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: echo-rs
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
이전에 작성한 것과 차이점은 replicas: 4 입니다. 바로 실행해봅니다.

```bash
kubectl apply -f echo-rs-scaled.yml

#Pod 확인
kubectl get pod,rs
```

실행 결과

```bash
NAME                READY   STATUS    RESTARTS   AGE
pod/echo-health     1/1     Running   0          99m
pod/echo-rp         0/1     Running   0          5h28m
pod/echo-rs-4slzq   1/1     Running   0          13m
pod/echo-rs-4zhst   1/1     Running   0          13m
pod/echo-rs-7d5r2   1/1     Running   0          22m
pod/echo-rs-cpqwm   1/1     Running   0          13m
pod/mariadb         1/1     Running   0          31m
pod/mongodb         1/1     Running   0          76m

NAME                      DESIRED   CURRENT   READY   AGE
replicaset.apps/echo-rs   4         4         4       22m
```

기존에 생성된 Pod외에 3개가 추가되었습니다

## 마무리 
- 레플리카셋은 원하는 개수의 팟을 유지하는 역할 담당
- 라벨을 이용 팟을 체크하기 때문에 라벨이 겹치지 않게 신경써서 정의해야함 
- 실전에서는 레플리카셋을 단독으로 쓰는 경우는 거의 없음 
  - 디플로이먼트를 주로 이용함 