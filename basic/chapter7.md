---
marp: true
---

# Configmap과 Secret

---

어플리케이션에 설정값이 필요

도커 이미지 파일에 설정값을 하드 코딩 할 수 있음.

이 방식은 설정 옵션을 유연하게 변경할 수 없다는 단점이 있음.

고로 Configmap과 secret을 활용하여 설정값과 비밀값에 대해 이 문제를 해결할 수 있음.

---

## Configmap

네임스페이스 별로 Configmap이 존재함을 명심할 것

Configmap을 생성하는 다양한 방법이 존재

- Create ConfigMaps from literal values : —from-literal
- Create a ConfigMap from generator : configMapGenerator with kustomize

---

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: game-demo
data:
  # 속성과 비슷한 키; 각 키는 간단한 값으로 매핑됨
  player_initial_lives: 3
  ui_properties_file_name: "user-interface.properties"
  #
  # 파일과 비슷한 키
  game.properties: |
    enemy.types=aliens,monsters
    player.maximum-lives=5
  user-interface.properties: |
    color.good=purple
    color.bad=yellow
    allow.textmode=true 
```

---

## 참조

[https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/)

[https://kubernetes.io/ko/docs/concepts/configuration/configmap/](https://kubernetes.io/ko/docs/concepts/configuration/configmap/)

[https://kubernetes.io/ko/docs/tasks/manage-kubernetes-objects/kustomization/](https://kubernetes.io/ko/docs/tasks/manage-kubernetes-objects/kustomization/)

---