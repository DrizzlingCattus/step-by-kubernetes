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

# 롤 vs 클러스터 롤

롤 - 네임스페이스에 한정되는 오브젝트 권한 정의
ex) pod, service, deployment

클러스터 롤 - 클러스터 단위의 리소르 권한 정의
ex) node, pv

---

# 클러스터 롤 생성하기

```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole # Role 파일이랑 다른점은 kind 정도...
metadata:
  namespace: default
  name: nodes-reader
rules:
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["get", "list"]
```

---

# 클러스터 롤 바인딩

롤 바인당과 마찬가지로 클러스터 롤 바인딩도 존재

```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: test-clusterrole-binding
  namespace: default
subjects:
- kind: ServiceAccount
  name: test-user
  namespace: default
roleRef:
  kind: ClusterRole
  name: nodes-reader
  apiGroup: rbac.authorization.k8s.io
```

---


```
kubectl get nodes --as system:serviceaccount:default:test-user
```

---

### 만약 네임스페이스에 종속되는 오브젝트에 대해 클러스터 롤을 생성할 경우, 모든 네임스페이스의 리소스 권한이 부여

---

# 쿠버네티스 API를 사용하기

쿠버네티스 API 를 시용하기 위해서는 인증이 필요함.

curl로 직접 쿠버네티스 API 서버에 질의해보자

---

## master node 주소는 어떻게 알지?

---

```
$ kubectl config view
apiVersion: v1
clusters:
- cluster:
    certificate-authority: /Users/user/.minikube/ca.crt
    server: https://192.168.64.2:8443
  name: minikube
contexts:
- context:
    cluster: minikube
    user: minikube
  name: minikube
current-context: minikube
kind: Config
preferences: {}
users:
- name: minikube
  user:
    client-certificate: /Users/user/.minikube/profiles/minikube/client.crt
    client-key: /Users/user/.minikube/profiles/minikube/client.key
```

---

## master 노드에 질의를 하면

```
# -k 옵션은 알려지지않은 인증서를 사용하겠다.
$ curl https://192.168.64.2:8443 -k
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {

  },
  "status": "Failure",
  "message": "forbidden: User \"system:anonymous\" cannot get path \"/\"",
  "reason": "Forbidden",
  "details": {

  },
  "code": 403
}
```

---

> "message": "forbidden: User \"system:anonymous\" cannot get path \"/\"",

system:anonymous는 인증에 실패한 유저(User)를 의미

실패한 이유는 API를 호출할 권한이 없기 때문.

해당 API 질의시 `내가 어떤 ServiceAccount 다`라는 정보를 전송하는 방법이 필요.

---

모든 ServiceAccount가 만들어질때 대응되는 secret도 만들어짐.

```
$ kubectl get secrets
NAME                   TYPE                                  DATA   AGE
default-token-kx6s5    kubernetes.io/service-account-token   3      28m
stenrine-token-xt9dc   kubernetes.io/service-account-token   3      5s
```

이 secret으로 ServiceAccount임을 증명할 수 있다.

---

이렇게 ServiceAccount에 secret이 할당되있는 모습을 볼 수 있다.

```
$ kubectl describe sa stenrine
Name:                stenrine
Namespace:           default
Labels:              <none>
Annotations:         <none>
Image pull secrets:  <none>
Mountable secrets:   stenrine-token-xt9dc
Tokens:              stenrine-token-xt9dc
Events:              <none>
```

---

ca.crt는 공개 인증서
namespace는 해당 ServiceAccount가 존재하는 네임스페이스
이중 token을 가지고 JWT 인증을 수행하면 된다.

```
$ kubectl describe secret stenrine-token-xt9dc
Name:         stenrine-token-xt9dc
Namespace:    default
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: stenrine
              kubernetes.io/service-account.uid: cfffb475-5a26-4af6-97b3-a0f8a11384f5

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1066 bytes
namespace:  7 bytes
token:      eyJhbGciOiJSUzI1NiIsIm ...(생략)...
```

---

# JWT 인증 도식도

```
secret <--------> ServiceAccount <--(RoleBinding)--> Role 
(stenrine token)    (stenrine)
  |
  |
   ----> (사용자 혹은 App) --(JWT 인증)--> kubernetes API server
```

---

![bg 100%](https://d33wubrfki0l68.cloudfront.net/d65bee40cabcf886c89d1015334555540d38f12e/c6a46/images/docs/admin/k8s_oidc_login.svg)


---

사용자는 curl 명령에 Authorization header에 base64로 디코딩한 토큰을 함께 넣어 보내면 됩니다.

```
curl https://192.168.64.2:8443 --header "Authorization: Bearer $decoded_token" -k
```

base64로 디코딩하는 이유는 secret 데이터는 base64로 인코딩되있기 때문

---

로컬 호스트 요청만 처리한다면 kubectl proxy 기능을 사용할 수 있습니다. (테스트 용도)

---


DEX와 OAuth를 이용한 쿠버 사용자 인증

https://m.blog.naver.com/PostView.nhn?blogId=alice_k106&logNo=221598325656&proxyReferer=https:%2F%2Fwww.google.com%2F

---

네트워크 로드벨런서 정리 잘해둔곳

https://medium.com/@pakss328/%EB%A1%9C%EB%93%9C%EB%B0%B8%EB%9F%B0%EC%84%9C%EB%9E%80-l4-l7-501fd904cf05


---

## 클러스터 내부에서 kubernetes API 사용하기

---

쿠버네티스 클러스터 내부에서 실행되는 애플리케이션은 어떻게 API 서버에 접근하고 인증을 수행할 수 있을끼?

---

크게 2가지 방법이 있음
1. built-in kubernetes service 이용
2. kubernetes SDK 이용

SDK는 다른 언어로 구현된 라이브러리 같은 녀석이니 패스하고 첫번째만 살펴볼것

---

## 첫번째 방법

쿠버네티스는 클러스터 내부에서 API를 접근할 수 있도록 미리 만들어둔 서비스가 있음

```
➡ ~()$ kubectl get svc
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   4d
```

kubernetes란 이름의 서비스를 통해 API 서버에 접근 가능

---

```

pod ----(
  kubernetes.default.svc라는 dns로 접근 가능
)--> kubernetes란 이름의 service ---
--> kubernetes API server

```

이제 인증을 하려면 pod에서 secret 값을 보내야하는데,

그렇다면 secret 값을 pod에 어떻게 가져오나?

---

쿠버네티스는 포드를 생성할때 자동으로 서비스 어카운트의 시크릿을 포드 내부에 마운트함.

서비스 어카운트는 pod yaml에서 따로 설정이 안되있으면 default 서비스 어카운트를 사용.

```
$ kubectl describe pod nginx-deployment-6b474476c4-cbcq2
(...)
Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-kx6s5 (ro)
(...)
```

---

마운트 경로에 가보면 다음과 같이 token 파일이 있는것을 확인할 수 있음

```
$ kubectl exec nginx-deployment-6b474476c4-cbcq2 ls /var/run/secrets/kubernetes.io/serviceaccount
ca.crt
namespace
token
```

---

## kubeconfig

kubeconfig 파일에는 클러스터에 접근할 수 있는 다영한 인증정보들이 포함되어 있습니다.

kubectl config 명령을 통해 확인, 수정이 가능합니다.

---

## kubeconfig 파일 구조

- clusters : 쿠버네티스 클러스터(API 서버)의 접속 정보 목록
- users : 사용자 인증 정보. 사용자 명, 서비스 어카운트 토큰, 인증서 정보가 포함.
- context : cluster와 user를 조합한 정보. kubectl은 해당 context중 하나를 선택해야한다.

---

## Example

```
$ cat ~/.kube/config
apiVersion: v1
clusters:
- cluster: # 아래로 클러스터 정보들이 있음
    certificate-authority: /Users/user/.minikube/ca.crt
    server: https://192.168.64.2:8443
  name: minikube # 클러스터 이름
contexts:
- context:
    cluster: minikube # cluster에서 가져온부분
    user: minikube # user에서 가져온 부분
  name: minikube # 컨텍스트 이름
current-context: minikube # kubectl이 현재 사용중인 context
kind: Config
preferences: {}
users:
- name: minikube # 유저 이름
  user: # 인증서로 user 인증 정보 구성
    client-certificate: /Users/user/.minikube/profiles/minikube/client.crt
    client-key: /Users/user/.minikube/profiles/minikube/client.key
```

---

아렇게 컨텍스트를 생성할 수 있다.

```
➡ kuberntes()$ kubectl config get-contexts
CURRENT   NAME       CLUSTER    AUTHINFO   NAMESPACE
*         minikube   minikube   minikube

➡ kuberntes()$ kubectl config set-context my-new-context --cluster=minikube --user=minikube
Context "my-new-context" created.

➡ kuberntes()$ kubectl config get-contexts
CURRENT   NAME             CLUSTER    AUTHINFO   NAMESPACE
*         minikube         minikube   minikube
          my-new-context   minikube   minikube
```

---

또한 사용할 컨텍스트를 변경할 수 있다.

```
$ kubectl config use-context my-new-context
```

---

# User와 Group

---

서비스 어카운트와 User 및 Group 개념과 차이점은? 왜 존재하는 개념?

---

User는 실제 쿠버네티스 사용자를 의미함.

서비스 어카운트는 이런 유저의 한 종류를 말함.

---

요 아래 내용을 해석할 수 있으면 이해를 빨리 할 수 있음.

```
$ kubectl get services --as system:serviceaccount:default:stenrine

Error from server (Forbidden): services is forbidden: 
User "system:serviceaccount:default:stenrine" 
cannot list resource "services" in API group "" 
in the namespace "default"
```

system:serviceaccount:default:stenrine 가 유저이름.

system:serviceaccount:<네임스페이스>:<서비스 어카운트 이름>

---

Group은 User 집합을 나타냄.

위에서 언급된

`system:serviceaccount`

가 서비스 어카운트 집합이라고 함.

이 집합은 모든 네임스페이스에 속하는 모든 서비스 어카운트가 속한 집합임.

---