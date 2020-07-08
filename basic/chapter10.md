---
marp: true
---

# ServiceAccount and RBAC

---

여러 개발자와 어플리케이션이 쿠버네티스를 동시에 사용하는 상황이 빈번함.

- 고려할 대상 중 하나는 보안
  - 리눅스 시스템을 생각해보면 됨. 모든 사용자가 루트 권한을 갖는다? 온갖 난장판이 펼쳐질 것.

---

쿠버네티스에서 제공하는 보안 기능

RBAC(Role Based Access Control)

RBAC 기반의 Service Account 기능을 사용

---

RBAC 기반의 Service Account 기능을 사용

- RBAC 기능을 통해 Service Account에게 명령 실행 권한을 지정
- Service Account는 사용자 또는 App에 해당

---

우리가 알아갈 내용

- Service Account 및 RBAC를 사용하기 위한 Role
- 클러스터 Role
- 사용자를 추상화한 User 및 Group
- Open ID Connect(OIDC)
  - OAuth

---

kubectl apply -f 명령시 일어나는 일들

> 사용자(kubectl) -> kube-apiserver -> etcd

---

kube-apiserver에서 일어나는 일들

> HTTP handler -> Authentication -> Authorization -> Mutating Admission Controller -> Validating Admission Controller

클라이언트가 쿠버네티스 사용자가 맞는지(인증), 해당 기능을 실행할 권한이 있는지(인가) 체크하기 위해 다음과 같은 방법을 제공

- 인증서
- OAuth
- Service Account

---

기본적으로 처음 설치 시에 kubectl이 관리자 권한을 가지도록 설정됨.

`~/.kube/config` 위치에 클러스터 및 유저 관련 설정이 있음.

이 처럼 인증서로 인증하는 방법은 관리가 힘들기에 자주 사용되는 방법이 아니라고 함.

---

# Service Account

---

체계적으로 권한을 관리하기 위한 `쿠버네티스 오브젝트`.

서비스 어카운트는 한 명의 사용자나 어플리케이션에 해당한다고 생각하면 쉬움.

---

```
# list
kubectl get serviceaccount
kubectl get sa

# create
kubectl create sa <service-account-name>

# usage - impersonate
kubectl get services --as system:serviceaccount:default:<service-account-name>
```

ps) impersonate는 특정 사용자를 가장해(권한으로) API를 사용하는 행위를 말한다.

ps) system:serviceaccount는 인증을 위해 서비스 어카운트를 사용한다는 이야기

ps) default는 account의 네임스페이스

---

아직 위 명령어는 서비스 어카운트를 생성했을 뿐 권한을 설정하지 않아 에러 발생

---

# 서비스 어카운트에 권한 설정하기

---

Role과 Cluster Role을 통해 권한을 설정할 수 있다.

각각은 마찬가지로 쿠버네티스 오브젝트다.

> An RBAC Role or ClusterRole contains rules that represent a set of permissions. Permissions are purely additive (there are no "deny" rules).

고로 whitelist 형태로 권한을 설정할 뿐, blacklist는 없다.

---

Role은 네임스페이스에 속하는 오브젝트다.

고로 네임스페이스에 속하는 오브젝트들에 대한 권한을 정의할 때 사용합니다.

> A Role always sets permissions within a particular namespace ; when you create a Role, you have to specify the namespace it belongs in.

즉 Role이 만들어진 namespace에 속하지 않는 오브젝트는 논외인것

---

```
kubectl get role

kubectl get clusterrole
```

---

```
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: test-role
rules:
  - apiGroups: [""] # 대상이 될 오브젝트 API 그룹
    resources: ["services"] # 대상이 될 오브젝트 이름
    verbs: ["get", "list"] # 어떤 동작을 허용할 건지 명시
```

apiGroups: 대상이 될 오브젝트 API 그룹. ""는 코어 API 그룹

resources: 대상이 될 오브젝트 이름. kubectl api-resources에 출력되는 대상들을 사용가능.

verbs: 어떤 동작을 허용할 건지 명시. 위에서는 kubectl get services 혹은 kubectl list services가 가능

<!--
7/1 스터디 결과

-->

---

## Role 생성 결과

```
➡ daily()$ kubectl apply -f role.yaml
role.rbac.authorization.k8s.io/test-role created

➡ daily()$ kubectl get roles
NAME        AGE
test-role   7s
```

---

이 상태는 롤(권한)만 생성했을 뿐, 실제 어떤 대상에게 권한을 할당한 상태는 아니다.
Service Account에게 롤을 부여하려면 RoleBinding이라는 오브젝트를 만들어서 할 수 있다.

---

## RoleBinding 예시

```
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: test-role-binding
  namespace: default
subjects:
  - kind: ServiceAccount # 권한 부여 대상
    name: test-account # test-account라는 이름의 ServiceAccount에 해당
    namespace: default
roleRef:
  kind: Role
  name: test-role
  apiGroup: rbac.authorization.k8s.io
```

즉 `subjects`에게 `roleRef`에 해당하는 Role을 할당하는 작업

---

## RoleBinding 생성 결과

```
➡ daily()$ kubectl get services --as system:serviceaccount:default:test-account
NAME         TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE
hello-node   LoadBalancer   10.96.251.38   <pending>     8080:32199/TCP   150d
kubernetes   ClusterIP      10.96.0.1      <none>        443/TCP          150d

➡ daily()$ kubectl get deployment --as system:serviceaccount:default:test-account
Error from server (Forbidden):
deployments.apps is forbidden:
User "system:serviceaccount:default:test-account" cannot list
resource "deployments" in API group "apps" in the namespace "default"
```

service에 대해서만 권한을 주었기에 deployments는 동작하지 않는 모습이다.

---

## 오늘의 소소한 깨달음 - kubernetes object names

path segment name: path 단위 이름임. `..`, `.`, `/`, `%` 은 포함되선 안됨.

dns sub-domain name 제약

    contain no more than 253 characters
    contain only lowercase alphanumeric characters, '-' or '.'
    start with an alphanumeric character
    end with an alphanumeric character

DNS Label Names 제약

    contain at most 63 characters
    contain only lowercase alphanumeric characters or '-'
    start with an alphanumeric character
    end with an alphanumeric character

https://kubernetes.io/docs/concepts/overview/working-with-objects/names/

---

## 다음주 월요일

- configmap 내부 동작 원리 파악
