### Домашнее задание к занятию «Управление доступом» Баранов Сергей

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


```
root@baranov:/home/baranovsa/kube2.4# openssl genrsa -out test.key 2048
Generating RSA private key, 2048 bit long modulus (2 primes)
...............................................................................................+++++
....+++++
e is 65537 (0x010001)
root@baranov:/home/baranovsa/kube2.4# openssl req -new -key test.key -out test.csr -subj "/CN=test/O=o>
root@baranov:/home/baranovsa/kube2.4# openssl x509 -req -in test.csr -CA ca.crt -CAkey ca.key -CAcreat>
Signature ok
subject=CN = test, O = ops
Getting CA Private Key
root@baranov:/home/baranovsa/kube2.4# ls
ca.crt  ca.key  ca.srl  role_binding.yaml  role.yaml  test.crt  test.csr  test.key
root@baranov:/home/baranovsa/kube2.4#
```

2. Настройте конфигурационный файл kubectl для подключения.

```
root@baranov:/home/baranovsa/kube2.4# kubectl config set-credentials test --client-certificate=cert/te>
User "test" set.
root@baranov:/home/baranovsa/kube2.4#
root@baranov:/home/baranovsa/kube2.4# kubectl config set-context test-context --cluster=microk8s-clust>
Context "test-context" created.
root@baranov:/home/baranovsa/kube2.4#
root@baranov:/home/baranovsa/kube2.4# kubectl config view
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: DATA+OMITTED
    server: https://10.0.2.15:16443
  name: microk8s-cluster
- cluster:
    certificate-authority: /root/.minikube/ca.crt
    extensions:
    - extension:
        last-update: Sat, 13 Apr 2024 22:30:01 +07
        provider: minikube.sigs.k8s.io
        version: v1.32.0
      name: cluster_info
    server: https://192.168.49.2:8443
  name: minikube
contexts:
- context:
    cluster: microk8s-cluster
    user: admin
  name: microk8s
- context:
    cluster: minikube
    extensions:
    - extension:
        last-update: Sat, 13 Apr 2024 22:30:01 +07
        provider: minikube.sigs.k8s.io
        version: v1.32.0
      name: context_info
    namespace: default
    user: minikube
  name: minikube
- context:
    cluster: microk8s-cluster
    user: sergey
  name: sergey-context
- context:
    cluster: docker-desktop
    user: test
  name: test
- context:
    cluster: microk8s-cluster
    user: test
  name: test-context
- context:
    cluster: microk8s-cluster
    user: user
  name: user-context
current-context: test
kind: Config
preferences: {}
users:
- name: admin
  user:
    client-certificate-data: DATA+OMITTED
    client-key-data: DATA+OMITTED
- name: minikube
  user:
    client-certificate: /root/.minikube/profiles/minikube/client.crt
    client-key: /root/.minikube/profiles/minikube/client.key
- name: sergey
  user:
    client-certificate: /home/baranovsa/cert2/cert/sergey.crt
    client-key: /home/baranovsa/cert2/cert/sergey.key
- name: test
  user:
    client-certificate: /home/baranovsa/kube2.4/cert/test.crt
    client-key: /home/baranovsa/kube2.4/cert/test.key
- name: user
  user:
    client-certificate-data: DATA+OMITTED
    client-key-data: DATA+OMITTED
root@baranov:/home/baranovsa/kube2.4#
root@baranov:/home/baranovsa/kube2.4# kubectl config get-contexts
CURRENT   NAME             CLUSTER            AUTHINFO   NAMESPACE
          microk8s         microk8s-cluster   admin
*         minikube         minikube           minikube   default
          sergey-context   microk8s-cluster   sergey
          test             docker-desktop     test
          test-context     microk8s-cluster   test
          user-context     microk8s-cluster   user
root@baranov:/home/baranovsa/kube2.4#
```

3. Создайте роли и все необходимые настройки для пользователя.

[role.yaml]()

```
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-desc-logs
  namespace: default
rules:
- apiGroups: [""]
  resources: ["pods", "pods/log"]
  verbs: ["watch", "list", "get"]
```

[role_binding.yaml]()

```
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: pod-reader
  namespace: default
subjects:
- kind: User
  name: test
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-desc-logs
  apiGroup: rbac.authorization.k8s.io

```

```
root@baranov:/home/baranovsa/kube2.4#kubectl apply -f role.yaml
role.rbac.authorization.k8s.io/pod-desc-logs created
root@baranov:/home/baranovsa/kube2.4#
root@baranov:/home/baranovsa/kube2.4#kubectl apply -f role_binding.yaml 
rolebinding.rbac.authorization.k8s.io/pod-reader created
root@baranov:/home/baranovsa/kube2.4#
```


4. Предусмотрите права пользователя. Пользователь может просматривать логи подов и их конфигурацию (`kubectl logs pod <pod_id>`, `kubectl describe pod <pod_id>`).

```
root@baranov:/home/baranovsa/kube2.4# kubectl config use-context test-context
Switched to context "test-context".
root@baranov:/home/baranovsa/kube2.4#
root@baranov:/home/baranovsa/kube2.4# kubectl get node
Error from server (Forbidden): nodes is forbidden: User "test" cannot list resource "nodes" in API group "" at the cluster scope
root@baranov:/home/baranovsa/kube2.4# kubectl get pod
NAME                                  READY   STATUS    RESTARTS        AGE
deployment-01-5c9cdc8d6d-4wwzs        2/2     Running   7 (3h12m ago)   3d20h
deployment-01-5c9cdc8d6d-rn27k        2/2     Running   7 (3h12m ago)   3d20h
root@baranov:/home/baranovsa/kube2.4#
root@baranov:/home/baranovsa/kube2.4# kubectl get deployment
Error from server (Forbidden): deployments.apps is forbidden: User "test" cannot list resource "deployments" in API group "apps" in the namespace "default"
root@baranov:/home/baranovsa/kube2.4#
root@baranov:/home/baranovsa/kube2.4# kubectl get ns
Error from server (Forbidden): namespaces is forbidden: User "test" cannot list resource "namespaces" in API group "" at the cluster scope
root@baranov:/home/baranovsa/kube2.4# kubectl get ns
Error from server (Forbidden): namespaces is forbidden: User "test" cannot list resource "namespaces" in API group "" at the cluster scope
root@baranov:/home/baranovsa/kube2.4#
root@baranov:/home/baranovsa/kube2.4# kubectl get sa
Error from server (Forbidden): serviceaccounts is forbidden: User "test" cannot list resource "serviceaccounts" in API group "" in the namespace "default"
root@baranov:/home/baranovsa/kube2.4#
root@baranov:/home/baranovsa/kube2.4# kubectl logs nfs-server-nfs-server-provisioner-0
I0414 11:28:20.130978       1 main.go:64] Provisioner cluster.local/nfs-server-nfs-server-provisioner specified
I0414 11:28:20.162135       1 main.go:88] Setting up NFS server!
I0414 11:28:29.062773       1 server.go:149] starting RLIMIT_NOFILE rlimit.Cur 65536, rlimit.Max 65536
I0414 11:28:29.089024       1 server.go:160] ending RLIMIT_NOFILE rlimit.Cur 1048576, rlimit.Max 1048576
I0414 11:28:29.254766       1 server.go:134] Running NFS server!
root@baranov:/home/baranovsa/kube2.4#
root@baranov:/home/baranovsa/kube2.4# kubectl logs deployment-01-5c9cdc8d6d-4wwzs
Defaulted container "nginx" out of: nginx, multitool
/docker-entrypoint.sh: /docker-entrypoint.d/ is not empty, will attempt to perform configuration
/docker-entrypoint.sh: Looking for shell scripts in /docker-entrypoint.d/
/docker-entrypoint.sh: Launching /docker-entrypoint.d/10-listen-on-ipv6-by-default.sh
10-listen-on-ipv6-by-default.sh: info: Getting the checksum of /etc/nginx/conf.d/default.conf
10-listen-on-ipv6-by-default.sh: info: Enabled listen on IPv6 in /etc/nginx/conf.d/default.conf
/docker-entrypoint.sh: Launching /docker-entrypoint.d/20-envsubst-on-templates.sh
/docker-entrypoint.sh: Launching /docker-entrypoint.d/30-tune-worker-processes.sh
/docker-entrypoint.sh: Configuration complete; ready for start up
2024/04/14 09:15:02 [notice] 1#1: using the "epoll" event method
2024/04/14 09:15:02 [notice] 1#1: nginx/1.20.2
2024/04/14 09:15:02 [notice] 1#1: built by gcc 10.2.1 20210110 (Debian 10.2.1-6)
2024/04/14 09:15:02 [notice] 1#1: OS: Linux 5.10.0-28-amd64
2024/04/14 09:15:02 [notice] 1#1: getrlimit(RLIMIT_NOFILE): 65536:65536
2024/04/14 09:15:02 [notice] 1#1: start worker processes
2024/04/14 09:15:02 [notice] 1#1: start worker process 31
2024/04/14 09:15:02 [notice] 1#1: start worker process 32
root@baranov:/home/baranovsa/kube2.4#
```

5. Предоставьте манифесты и скриншоты и/или вывод необходимых команд.

[role.yaml]()

[role_binding.yaml]()



------

### Правила приёма работы

1. Домашняя работа оформляется в своём Git-репозитории в файле README.md. Выполненное домашнее задание пришлите ссылкой на .md-файл в вашем репозитории.

2. Файл README.md должен содержать скриншоты вывода необходимых команд `kubectl`, скриншоты результатов.

3. Репозиторий должен содержать тексты манифестов или ссылки на них в файле README.md.

------

