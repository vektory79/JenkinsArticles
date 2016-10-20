# Установка

В качестве заготовки для конфигурации установки использовал следующий репозитарий: https://github.com/runcom/my-openshift-origin

Но оказалось, что и для этого варианта установки нужно проделать ряд подготовительных действий.

После проведения всех подготовительных мероприятий можно запустить инсталляцию так:

```bash
ansible-playbook playbooks/byo/config.yml --inventory /root/work/my-openshift-origin/hosts
```

Теперь подробнее о подготовке и разных особенностях настройки:

## Подготовка OS

Прежде всего, нужно убедиться, что в основное сетевое устройство имеет имя eth0. Как выяснилось в CentOS изменили стратегию именования сетевых устройств и теперь могут формироваться имена типа eno12312. Что с этим делать можно почитать тут: http://mylinuxdiary.com/index.php/2015/07/04/changing-network-interface-name-from-eno-to-eth0-in-centos-7/

У всех узлов **желательно** иметь доменные имена. Иначе придётся много всякого прописывать в **/etc/hosts** на узле с точкой входа в кластер.

Все IP адреса должны быть статическими, т.к. etcd собирает кластер именно по IP, а не по доменным именам.

Далее необходимо правильно настроить файловую подсистему для образов/контейнеров Docker. Иначе будут дикие тормоза в самых непредсказуемых местах. Подробнее тут: https://docs.docker.com/engine/userguide/storagedriver/device-mapper-driver/

Ну и в целом необходимо проконтролировать пререквизиты: https://docs.openshift.org/latest/install_config/install/prerequisites.html

## NFS

OpenShift может сразу при инсталляции развернуть внутренний репозитарий Docker образов и сервис сбора статистики. Для этого ему необходима настроенная NFS шара на каком-либо внешнем ресурсе (на сервере, не входящем с кластер). Но если такого сервера нету и надо расшарить ресурс прямо с одного из узлов, то происходит "опаньки", т.к. в процессе инсталляции полностью перенастраивается фаервол и доступ к NFS оказывается закрытым.

Для нормальной работы инсталятора еобходимо применить патч а Ansible скрипту:

```diff
diff --git a/playbooks/common/openshift-nfs/config.yml b/playbooks/common/openshift-nfs/config.yml
index 000e46e..68ec15b 100644
--- a/playbooks/common/openshift-nfs/config.yml
+++ b/playbooks/common/openshift-nfs/config.yml
@@ -2,5 +2,15 @@
 - name: Configure nfs
   hosts: oo_nfs_to_config
   roles:
+  - role: os_firewall
+    os_firewall_allow:
+    - service: rpcbind tcp
+      port: 111/tcp
+    - service: rpcbind udp
+      port: 111/udp
+    - service: NFS tcp
+      port: 2049/tcp
+    - service: NFS udp
+      port: 2049/udp
   - role: openshift_facts
   - role: openshift_storage_nfs
```

Ещё раз: NFS шара должна быть создана до запуска инсталлятора и указана в environments у Ansible.

В NFS шаре нужно предварительно создать два каталога с правами на запись: **metrics** и **registry**.

## LDAP

OpenShift поддерживает различные методы аутентификации. В частности LDAP. Для его подключения нужно у Ansuble прописать следующий параметр (тут и далее пароли изменены):

```ini
openshift_master_identity_providers=[{'name': 'krista_ldap_provider', 'challenge': 'true', 'login': 'true', 'kind': 'LDAPPasswordIdentityProvider', 'attributes': {'id': ['dn'], 'email': ['mail'], 'name': ['cn'], 'preferredUsername': ['uid']}, 'bindDN': 'user@domain.org', 'bindPassword': 'password', 'ca': '', 'insecure': 'false', 'url': 'ldap://dc8a.domain.org:389/dc=domain,dc=org?sAMAccountName'}]
```

Нужно обратить внимание, что указан конкретный сервер LDAP. В большинстве других систем можно указывать просто domain.org для обеспечения автоматического обнаружения и балансировки серверов. Но в случае с OpenShift этот номер не проходит, т.к. SSL сертификаты выданы на конкретные сервера, а OpenShift пытается проверять их для домена domain.org и ругается, что сертификат "левый".

**Примечание:** т.к. тут используется JSON структура, то возможно лучше задавать две настройки для каждого из имеющихся сейчас серверов. Но это надо ещё тестировать, как OpenShift на это будет реагировать.

## Proxy

Прокси, прежде всего, нужно задавать в настройках Ansible:

```ini
openshift_http_proxy=http://user:password@proxy6.domain.org:8080
openshift_https_proxy=https://user:password@proxy6.domain.org:8080
openshift_no_proxy=domain.org,localhost
```

В принципе этого достаточно. В процессе инсталляции, Ansible пропишет эти значения во все места где нужно. Включая OS, Docker и Git. Но, если хочется более стабильного результата установки (см. ниже), то придётся самостоятельно прописать прокси для Docker на каждом узле. Для этого в файл **/etc/sysconfig/docker** нужно добавить строки:

```ini
HTTP_PROXY=http://user:password@proxy6.domain.org:8080
HTTPS_PROXY=http://user:password@proxy6.domain.org:8080
NO_PROXY=localhost,127.0.0.1,domain.org,10.0.0.0/8
```

и перезапустить Docker:

```bash
systemctl restart docker
```

## прогрев инсталятора

Ansible, в процессе инсталляции, разворачивает ряд Dcoker образов из внешних репозитариев. Но, связь с внешними репозитариями часто бывает плохой. В результате загрузка образов затягивается и Ansible вылетает по таймауту. Во избежание этой ситуации можно зарание загрузить на каждую ноду все необходимые образы:

```bash
docker pull cockpit/kubernetes:latest
docker pull openshift/origin-docker-registry:v1.3.0
docker pull openshift/origin-haproxy-router:v1.3.0
docker pull openshift/origin-deployer:v1.3.0
docker pull openshift/origin-pod:v1.3.0
docker pull openshift/origin-sti-builder:v1.3.0
docker pull openshift/origin-metrics-deployer:latest
docker pull openshift/origin-metrics-cassandra:latest
docker pull openshift/origin-metrics-heapster:latest
docker pull openshift/origin-metrics-hawkular-metrics:latest
```

Только для этого придётся настраивать прокси в Dcoker (см. ранее).

## Доступ к Cockpit

Т.к. инсталятор перетирает все правила в файерволе при установке, то приходится заново открывать доступ в Cockpit.

Для этого необходимо в файле **/etc/sysconfig/iptables** добавить строчку

```
-A INPUT -p tcp -m tcp --dport 9090 -j ACCEPT
```

перед

```
-A INPUT -j REJECT --reject-with icmp-host-prohibited
```

И перезапустить iptables:

```bash
systemctl restart iptables
```

## Ansible environment

Для начала установки с помощью Ansible, ему необходимо предоставить environment файл, в котором указать что и как и куда устанавливать. Ниже приведён файл, который использовался при развёртывании стенда OpenShift:

```ini
[OSEv3:children]
masters
nodes
etcd
nfs

[OSEv3:vars]
ansible_ssh_user=root
deployment_type=origin
openshift_master_identity_providers=[{'name': 'krista_ldap_provider', 'challenge': 'true', 'login': 'true', 'kind': 'LDAPPasswordIdentityProvider', 'attributes': {'id': ['dn'], 'email': ['mail'], 'name': ['cn'], 'preferredUsername': ['uid']}, 'bindDN': 'user@domain.org', 'bindPassword': 'password', 'ca': '', 'insecure': 'false', 'url': 'ldap://dc8a.domain.org:389/dc=domain,dc=org?sAMAccountName'}]

osn_storage_plugin_deps=['ceph','glusterfs']

openshift_hosted_registry_storage_kind=nfs
openshift_hosted_registry_storage_host=test-os-master.domain.org
openshift_hosted_registry_storage_nfs_options='*(rw,root_squash)'
openshift_hosted_registry_storage_nfs_directory=/storage/nfs
openshift_hosted_registry_storage_volume_name=registry
openshift_hosted_registry_storage_access_modes=['ReadWriteMany']

openshift_hosted_metrics_deploy=true
openshift_hosted_metrics_storage_kind=nfs
openshift_hosted_metrics_storage_host=test-os-master.domain.org
openshift_hosted_metrics_storage_nfs_directory=/storage/nfs
openshift_hosted_metrics_storage_volume_name=metrics
openshift_hosted_metrics_storage_volume_size=10Gi
penshift_hosted_metrics_storage_access_modes=['ReadWriteOnce']
 
openshift_install_examples=true
openshift_master_default_subdomain=test-os-master.domain.org

osm_use_cockpit=true
osm_cockpit_plugins=['cockpit-kubernetes']

openshift_http_proxy=http://user:password@proxy6.domain.org:8080
openshift_https_proxy=https://user:password@proxy6.domain.org:8080
openshift_no_proxy=domain.org,localhost

[masters]
test-os-master.domain.org openshift_schedulable=true

[nodes]
test-os-master.domain.org openshift_node_labels="{'region': 'infra', 'role': 'master', 'name': 'test-os-master'}"
test-os-node1.domain.org openshift_node_labels="{'region': 'app', 'role': 'node', 'name': 'test-os-node1'}"
test-os-node2.domain.org openshift_node_labels="{'region': 'app', 'role': 'node', 'name': 'test-os-node2'}"

[etcd]
test-os-master.domain.org

[nfs]
test-os-master.domain.org
```

# Использование

## Подключение к стороннему Docker репозитарию

Основная информация тут: https://docs.openshift.org/latest/dev_guide/managing_images.html#allowing-pods-to-reference-images-from-other-secured-registries

Пример для конкретного случая:

```bash
oc secrets new-dockercfg docker-registry.domain.org --docker-server=docker-registry.domain.org:8021 --docker-username=openshift --docker-password=password  --docker-email=coreteam@domain.org
oc secrets link default docker-registry.domain.org --for=pull
```

## Выдача прав суперпользователся

Для того, чтобы выдать права суперпользователя необходимо выполнить следующую команду:

```bash
oadm policy add-cluster-role-to-user cluster-admin <user>
```

Например для пользователя из LDAP:

```bash
oadm policy add-cluster-role-to-user cluster-admin "CN=Пупкин Иван Иванович,OU=USERS,OU=Локальная сеть,DC=domain,DC=org"
```

## Повышение прав для контейнеров

OpenShift старается обеспечить максимальную безопасность. В результате по умолчанию проектам запрещено деплоить образы, которые стартуют под root. Ну и старается подпихнуть целый ряд ограничений на уровне cgroups и SELinux:

```yaml
      securityContext:
        capabilities:
          drop:
            - KILL
            - MKNOD
            - SETGID
            - SETUID
            - SYS_CHROOT
        privileged: false
        seLinuxOptions:
          level: 's0:c8,c2'
        runAsUser: 1000060000
```

Если в какой-то доверенный проект необходимо задеплоить образ, требующий расширенных прав, то тогда нужно выдать разрешение на этот проект:

```bash
oc project <project name>
oadm policy add-scc-to-user privileged -z default
```

Так же при деплое необходимо явно указать набор необходимых разрешений:

```yaml
 securityContext:
   runAsUser: 0
```

Поный Deployment Config может выглядеть так:

```yaml
apiVersion: v1
kind: DeploymentConfig
metadata:
  name: kallithea
  namespace: test
  selfLink: /oapi/v1/namespaces/test/deploymentconfigs/kallithea
  uid: 207c561c-8fd2-11e6-ba53-1eaef84320e2
  resourceVersion: '136117'
  generation: 4
  creationTimestamp: '2016-10-11T16:45:33Z'
  labels:
    app: kallithea
  annotations:
    openshift.io/generated-by: OpenShiftWebConsole
spec:
  strategy:
    type: Rolling
    rollingParams:
      updatePeriodSeconds: 1
      intervalSeconds: 1
      timeoutSeconds: 600
      maxUnavailable: 25%
      maxSurge: 25%
    resources:
  triggers:
    -
      type: ConfigChange
    -
      type: ImageChange
      imageChangeParams:
        automatic: true
        containerNames:
          - kallithea
        from:
          kind: ImageStreamTag
          namespace: test
          name: 'kallithea:latest'
        lastTriggeredImage: 'docker-registry.krista.ru:8021/krista/kallithea@sha256:43d98110f7164b1d84682f32ab003abbf7efe4715fa6566d517e71579cf46de4'
  replicas: 1
  test: false
  selector:
    app: kallithea
    deploymentconfig: kallithea
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: kallithea
        deploymentconfig: kallithea
      annotations:
        openshift.io/generated-by: OpenShiftWebConsole
    spec:
      volumes:
        -
          name: kallithea-1
          emptyDir:
      containers:
        -
          name: kallithea
          image: 'docker-registry.krista.ru:8021/krista/kallithea@sha256:43d98110f7164b1d84682f32ab003abbf7efe4715fa6566d517e71579cf46de4'
          ports:
            -
              containerPort: 5000
              protocol: TCP
          env:
            -
              name: PUID
              value: '1000'
            -
              name: PGID
              value: '1000'
            -
              name: KALLITHEA_LANG
              value: ru
          resources:
          volumeMounts:
            -
              name: kallithea-1
              mountPath: /config/kallithea
          terminationMessagePath: /dev/termination-log
          imagePullPolicy: Always
      restartPolicy: Always
      terminationGracePeriodSeconds: 30
      dnsPolicy: ClusterFirst
      securityContext:
        runAsUser: 0
status:
  latestVersion: 3
  observedGeneration: 4
  replicas: 1
  updatedReplicas: 1
  availableReplicas: 1
  details:
    message: 'caused by a config change'
    causes:
      -
        type: ConfigChange
```

# Прочее

* [Хорошая обзорная презентация](http://docs.huihoo.com/openshift/OpenShift-3-Technical-Architecture.pdf)
* [OpenShift xPaaS version 3. «Hello, world»](https://habrahabr.ru/post/312348/)
* [OpenShift v3. Часть II. Продолжение знакомства. ROR4](https://habrahabr.ru/post/312492/)
* [OpenShift v 3 III. OpenShift Origin 1.3](https://habrahabr.ru/post/312778/)
* [OpenShift + Jenkins + Bitbucket, непрерывная интеграция и публикация из коробки](https://habrahabr.ru/post/313146/)
