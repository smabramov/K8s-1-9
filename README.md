# «Управление доступом»-Абрамов Сергей

### Цель задания

В тестовой среде Kubernetes нужно предоставить ограниченный доступ пользователю.

------

### Чеклист готовности к домашнему заданию

1. Установлено k8s-решение, например MicroK8S.
2. Установленный локальный kubectl.
3. Редактор YAML-файлов с подключённым github-репозиторием.

------

### Инструменты / дополнительные материалы, которые пригодятся для выполнения задания

1. [Описание](https://kubernetes.io/docs/reference/access-authn-authz/rbac/) RBAC.
2. [Пользователи и авторизация RBAC в Kubernetes](https://habr.com/ru/company/flant/blog/470503/).
3. [RBAC with Kubernetes in Minikube](https://medium.com/@HoussemDellai/rbac-with-kubernetes-in-minikube-4deed658ea7b).

------

### Задание 1. Создайте конфигурацию для подключения пользователя

1. Создайте и подпишите SSL-сертификат для подключения к кластеру.
2. Настройте конфигурационный файл kubectl для подключения.
3. Создайте роли и все необходимые настройки для пользователя.
4. Предусмотрите права пользователя. Пользователь может просматривать логи подов и их конфигурацию (`kubectl logs pod <pod_id>`, `kubectl describe pod <pod_id>`).
5. Предоставьте манифесты и скриншоты и/или вывод необходимых команд.

------

### Решение

Создадим сертификаты:

```
serg@k8sclaster:~$ mkdir cert && cd cert
serg@k8sclaster:~/cert$ openssl genrsa -out netology.key 2048
serg@k8sclaster:~/cert$ openssl req -new -key netology.key -out netology.csr -subj "/CN=netology/O=group2"
serg@k8sclaster:~/cert$ openssl x509 -req -in netology.csr -CA /var/snap/microk8s/7394/certs/ca.crt -CAkey /var/snap/microk8s/7394/certs/ca.key -CAcreateserial -ou
t netology.crt -days 500
Certificate request self-signature ok
subject=CN = netology, O = group2

```
Настроим конфигурационный фаил:

```

serg@k8snode:~$ cd .kube/
serg@k8snode:~/.kube$ kubectl config set-credentials netology --client-certificate=/home/serg/.kube/cert/netology.crt --client-key=/home/serg/.kube/cert/netology.key
User "netology" set.
serg@k8snode:~/.kube$ cd
serg@k8snode:~$ kubectl config set-context netology-context --cluster=microk8s-cluster --user=netology
Context "netology-context" modified.
serg@k8snode:~$ kubectl config view
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: DATA+OMITTED
    server: https://192.168.0.249:16443
  name: microk8s-cluster
contexts:
- context:
    cluster: microk8s-cluster
    user: admin
  name: microk8s
- context:
    cluster: microk8s-cluster
    user: netology
  name: netology-context
current-context: microk8s
kind: Config
preferences: {}
users:
- name: admin
  user:
    client-certificate-data: DATA+OMITTED
    client-key-data: DATA+OMITTED
- name: netology
  user:
    client-certificate: cert/netology.crt
    client-key: cert/netology.key

    ```

Создал роли и все необходимые настройки для пользователя:

[role.yaml]()

[role-binding.yaml]()

```
serg@k8snode:~/git/K8s-1-9/code$ kubectl apply -f role.yaml
role.rbac.authorization.k8s.io/pod-logs-describe created
serg@k8snode:~/git/K8s-1-9/code$ kubectl apply -f role-binding.yaml
rolebinding.rbac.authorization.k8s.io/pod-reader created

```
Предусмотрел права пользователя. Пользователь может просматривать логи подов и их конфигурацию (kubectl logs pod <pod_id>, kubectl describe pod <pod_id>)

```
serg@k8snode:~/git/K8s-1-9/code$ kubectl get role pod-logs-describe -o yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"rbac.authorization.k8s.io/v1","kind":"Role","metadata":{"annotations":{},"name":"pod-logs-describe","namespace":"default"},"rules":[{"apiGroups":[""],"resources":["pods","pods/log"],"verbs":["watch","list"]}]}
  creationTimestamp: "2024-11-25T17:09:29Z"
  name: pod-logs-describe
  namespace: default
  resourceVersion: "182104"
  uid: 48d8f28c-9041-4fb1-8507-662dbd7040a2
rules:
- apiGroups:
  - ""
  resources:
  - pods
  - pods/log
  verbs:
  - watch
  - list

  ```

  Запустим на кластере pod.

```
serg@k8snode:~/git/K8s-1-9/code$ kubectl apply -f test_multitool.yaml 
pod/test-multitool created
serg@k8snode:~/git/K8s-1-9/code$ kubectl get pods -o wide
NAME             READY   STATUS    RESTARTS   AGE   IP             NODE         NOMINATED NODE   READINESS GATES
test-multitool   1/1     Running   0          5s    10.1.218.171   k8sclaster   <none>           <none>

```

Переключимся на пользователя и проверим pod:

```
serg@k8snode:~/git/K8s-1-9/code$ kubectl config get-contexts
CURRENT   NAME               CLUSTER            AUTHINFO   NAMESPACE
*         microk8s           microk8s-cluster   admin      
          netology-context   microk8s-cluster   netology   
serg@k8snode:~/git/K8s-1-9/code$ kubectl config use-context netology-context
Switched to context "netology-context".
serg@k8snode:~/git/K8s-1-9/code$ kubectl config get-contexts
CURRENT   NAME               CLUSTER            AUTHINFO   NAMESPACE
          microk8s           microk8s-cluster   admin      
*         netology-context   microk8s-cluster   netology 
serg@k8snode:~/git/K8s-1-9/code$ kubectl logs pod test-multitool
error: error from server (NotFound): pods "pod" not found in namespace "default"

```

Для выполнения команд logs и describe пользователю требуется добавление в роль verbs: ["get"]:

```
serg@k8snode:~/git/K8s-1-9/code$ kubectl get role pod pod-logs-describe -o yaml
apiVersion: v1
items:
- apiVersion: rbac.authorization.k8s.io/v1
  kind: Role
  metadata:
    annotations:
      kubectl.kubernetes.io/last-applied-configuration: |
        {"apiVersion":"rbac.authorization.k8s.io/v1","kind":"Role","metadata":{"annotations":{},"name":"pod-logs-describe","namespace":"default"},"rules":[{"apiGroups":[""],"resources":["pods","pods/log"],"verbs":["watch","list","get"]}]}
    creationTimestamp: "2024-11-25T17:09:29Z"
    name: pod-logs-describe
    namespace: default
    resourceVersion: "185254"
    uid: 48d8f28c-9041-4fb1-8507-662dbd7040a2
  rules:
  - apiGroups:
    - ""
    resources:
    - pods
    - pods/log
    verbs:
    - watch
    - list
    - get
    
```
Проверяем повторно:

```
serg@k8snode:~/git/K8s-1-9/code$ kubectl config use-context netology-context
Switched to context "netology-context".
serg@k8snode:~/git/K8s-1-9/code$ kubectl describe pod test-multitool
Name:             test-multitool
Namespace:        default
Priority:         0
Service Account:  default
Node:             k8sclaster/192.168.0.249
Start Time:       Mon, 25 Nov 2024 20:17:12 +0300
Labels:           <none>
Annotations:      cni.projectcalico.org/containerID: fe29558d8b0514214780f2f0fc43ec8e527ef5ba175e1117a14cca13a8b03de8
                  cni.projectcalico.org/podIP: 10.1.218.171/32
                  cni.projectcalico.org/podIPs: 10.1.218.171/32
Status:           Running
IP:               10.1.218.171
IPs:
  IP:  10.1.218.171
Containers:
  test-multitool:
    Container ID:   containerd://817b5045b38e6fc6f90dbac0d1913ab1a4e31d65dad0690de6011f36db4ba628
    Image:          wbitt/network-multitool
    Image ID:       docker.io/wbitt/network-multitool@sha256:d1137e87af76ee15cd0b3d4c7e2fcd111ffbd510ccd0af076fc98dddfc50a735
    Port:           <none>
    Host Port:      <none>
    State:          Running
      Started:      Mon, 25 Nov 2024 20:17:15 +0300
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-4g2vf (ro)
Conditions:
  Type                        Status
  PodReadyToStartContainers   True 
  Initialized                 True 
  Ready                       True 
  ContainersReady             True 
  PodScheduled                True 
Volumes:
  kube-api-access-4g2vf:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   BestEffort
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:                      <none>
serg@k8snode:~/git/K8s-1-9/code$ kubectl logs test-multitool
The directory /usr/share/nginx/html is not mounted.
Therefore, over-writing the default index.html file with some useful information:
WBITT Network MultiTool (with NGINX) - test-multitool - 10.1.218.171 - HTTP: 80 , HTTPS: 443 . (Formerly praqma/network-multitool)

```
Попробуем удалить pod:

```
serg@k8snode:~/git/K8s-1-9/code$ kubectl delete -f role.yaml 
Error from server (Forbidden): error when deleting "role.yaml": roles.rbac.authorization.k8s.io "pod-logs-describe" is forbidden: User "netology" cannot delete resource "roles" in API group "rbac.authorization.k8s.io" in the namespace "default"

```




### Правила приёма работы

1. Домашняя работа оформляется в своём Git-репозитории в файле README.md. Выполненное домашнее задание пришлите ссылкой на .md-файл в вашем репозитории.
2. Файл README.md должен содержать скриншоты вывода необходимых команд `kubectl`, скриншоты результатов.
3. Репозиторий должен содержать тексты манифестов или ссылки на них в файле README.md.

------

