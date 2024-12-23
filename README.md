# Настройка VM 
Информация по паролям/явкам тут.
В качестве mgm машины была выбрана 172.16.252.87.
На ней поставила докер, пип. Создала файл /etc/environment, в который через переменную KUBECONFIG задала кастомный путь к конфигу кубера.
Закинуть ssh ключ на все ноды будущего кластера.
Для этого подготовила файлик servers и скрипт по закидыванию на указанные серверы.
Ключ генерировала под пользователя gitlb-runner.

## Установка Gitlab на mgm сервере
Гитлаб ставила в докере, для этого был создан docker-compose.yaml.

![img_2.png](img_2.png)

Из интересного, это пропертя external_url. Установила ее во внешний адрес 5.8.17.160, но тогда происходила ошибка при последующем выполнение пайплайна на раннере. Так как раннер стучался по внешнему адресу, а не по адресу внутри сети. Так что было принято решение установить пропертю в значение внутреннего ип и регистрировать гитлаб раннер через токен. Благо он еще доступен до 18 версии гитлаба.
Создала два проекта: DevOps и Apps.

В DevOps лежит репозиторий для развертывания кубера с помощью kuberspay. Выкачала kuberspray версии 2.24, так до этого уже использовала данную версию. Так же там размещен репозиторий под чарты.

В Apps лежит модифицированный online boutique от гугла. Убрала оттуда чарты, манифесты кубера и остальные ненужные элементы.


![img_7.png](img_7.png)

Регистрация раннера через токен в репе:

![img.png](img.png)

Данный токен будет использоваться после установки раннера.


## Установка Gitlab Runner на mgm сервере
Устанавливала согласно [инструкции](https://docs.gitlab.com/runner/install/linux-repository.html).
Регистрация раннера:
```
gitlab-runner register \
--non-interactive \
--url "http://172.16.252.87" \
--token "$RUNNER_TOKEN" \
--executor "shell" 
```
RUNNER_TOKEN - из пункта выше.

![img_3.png](img_3.png)

## Установка k9s на mgm сервере

k9s скачала из [репозитория](https://github.com/derailed/k9s/releases/tag/v0.32.7) через wget, установила dpkg -i.


![img_4.png](img_4.png)

## Установка Harbor на mgm сервере
В качестве хранилища артифактов был выбран харбор, так как на момент настройки, на тачке не было выхода в интернет. Ставила через офлайн установщик, после чего сгенерировала сертификат для харбора.
Так же изменила следующие проперти в harbor.yml в соответствии с конфигурацией машин:
```
hostname: 172.16.252.87

http:
  port: 8080

https:
  port: 8443
  certificate: /opt/certs/harbor.crt
  private_key: /opt/certs/harbor.key
```

Сам харбор крутится в докере

![img_1.png](img_1.png)

### Установка helm на mgm сервере

Helm устанавливала в соответсвии с инструкцией на их сайте:
1. Скачала нужную вам версию
2. tar -zxvf helm-v3.0.0-linux-amd64.tar.gz
3. mv linux-amd64/helm /usr/local/bin/helm



![img_5.png](img_5.png)


![img_6.png](img_6.png)


## Пайлайны проектов в GitLab

Все пайплайны построены на правиле: он что-то делает только если это пуш в главную ветку, а это возможно только через мр. Все через мр, так как 1) +- во всех компаниях все через мр 2) мр проверяют старшие и умные 3) не всегда то, что ты пушишь в ветку дев хочется куда-то заливать (знаю по себе). 
Сама логика работы с проектом у меня такая:
1. Есть проект K8s otus, он в целом не зависит от того, что на него будет потом ставиться. То есть этот проект заточен только на развертывание кубера и его настройку.
2. Проект Charts содержит в себе наборы чартов, так же не зависит от того, куда мы потом его будет деплоить. Все чарты пушатся в Харбор.
3. Online boutique зависит как от того, где лежит Харбор, потому что берет оттуда образы и зависит от кубера.

# Online boutique
После рефактора оригинальной версии бутика, в репе находится только код сервисов, докерфайлы для них и в корне проекта лежит values.yaml под изменненую версию значения чарта. Алгоритм работы чарта такой, собираются образы сервисов, после чего они пушаться в Харбор. Затем в кубере разворачивается новый релиз чарта, с обновленными образами. 

```
stages:
  - clone-repo
  - build-push-image
  - helm-install-release

workflow:
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH

clone-repo-job:
  stage: clone-repo
  script:
    - env
    - cd ~
    - rm -rf online-boutique
    - git clone http://172.16.252.87/apps/online-boutique.git
    - echo "Finish cloning online-boutique"

build-push-image-job:
  stage: build-push-image
  script:
    - cd ~/online-boutique/src
    - tag="latest"
    - registry="172.16.252.87:8443/library"
    - |+
      for d in */; do
          image_name=${d::-1}
          cd $d
          docker build -f Dockerfile -t $image_name:$tag . > /dev/null
          docker tag $image_name:$tag $registry/$image_name:$tag
          docker push $registry/$image_name:$tag
          cd ..
      done

helm-install-release-job:
  stage: helm-install-release
  script:
    - cd ~/online-boutique
    - helm upgrade --install minio-operator operator --repo https://operator.min.io --version 6.0.3 --set tenants=null --create-namespace -n minio-operator
    - helm repo add prometheus https://prometheus-community.github.io/helm-charts
    - helm repo update
    - helm upgrade --install  prometheus prometheus/kube-prometheus-stack --version 67.4.0  -f helm-chart-values.yaml --create-namespace -n monitoring
    - helm upgrade --install online-boutique oci://172.16.252.87:8443/charts/onlineboutique -f helm-chart-values.yaml --create-namespace -n online-boutique
    - echo "Installed new helm release!"
```

# Charts
Пайплайн проходит по репозиторию, собирает чарты и пушит в харбор.
```
stages:
  - clone-repo
  - helm-package-push

workflow:
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH

clone-repo-job:
  stage: clone-repo
  script:
    - echo "staring cloning repo!"
    - cd ~
    - rm -rf charts
    - git clone http://172.16.252.87/devops/charts.git
    - echo "Finish cloning charts repo!"

helm-package-job:
  stage: helm-package-push
  script:
    - cd ~/charts/charts
    - registry="oci://172.16.252.87:8443/charts"
    - |+
      for d in */; do
        cd $d
        echo "Dir name: $d"
        helm dependency update
        helm package .
        package_name=$(ls | grep ".tgz")
        helm push $package_name $registry --insecure-skip-tls-verify
        cd ..
      done
```

# K8s otus
Данный пайплайн просто выкачивает репу и запускает несколько плейбуков для развертывания кластера.
```
stages:
  - clone-repo
  - run-kuberspray
  - copy-kubeconfig
  - change-kubeconfig
  - add-node-labels
  - update-certs
  - set-vip

workflow:
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH

clone-repo-job:
  stage: clone-repo
  script:
    - echo "staring cloning repo!"
    - cd ~
    - rm -rf otus-k8s
    - git clone http://172.16.252.87/devops/otus-k8s.git
    - echo "Finish cloning otus-k8s repo!"

run-kuberspray-job:
 stage: run-kuberspray
 script:
   - cd ~/otus-k8s
   - ansible-playbook -i inventory/otus/inventory.ini --become --become-user=root cluster.yml
   - echo "Finish k8s setup!"

copy-kubeconfig-job:
  stage: copy-kubeconfig
  script:
    - scp root@172.16.252.86:/tmp/config /tmp/config
    - echo "Finish copying kubeconf!"

change-kubeconfig-job:
  stage: change-kubeconfig
  script:
    - sed -i 's/127.0.0.1/172.16.252.86/' /tmp/config
    - echo "Finish fixing kubecong!"

label-nodes-job:
  stage: add-node-labels
  script:
    - ssh root@172.16.252.86 'kubectl label no crp-infra-1 crp-infra-2 role=infra; kubectl label no crp-monitoring-1 role=monitoring; kubectl label no crp-node-1 crp-node-2 crp-node-3 role=service'
    - echo "Finish labeling nodes!"

update-certs-job:
  stage: update-certs
  script:
    - cd ~/otus-k8s
    - ansible-playbook -i inventory/otus/inventory.ini --become --become-user=root update_certs.yml
    - echo "Finish k8s cert set up!"


set-vip-job:
  stage: set-vip
  script:
    - cd ~/otus-k8s
    - ansible-playbook -i inventory/otus/inventory.ini --become --become-user=root set_vip.yml
    - echo "Finish k8s vip set up!"
```

## Online boutique репозиторий
В данный момент поместила чарты бутика и минио. Минио выбрала для того, чтобы показать, что я умею пользоваться функционалом helm, ну и чтобы бутику не было одиноко в харборе. Немного поправила чарт бутика, добавила ингрес контроллер, а так же было интересно воспользоваться функционалом зависимостей в чарте. Добавила в качестве зависимостей минио и локи.

После пуша в дефолтную ветку и окончания работы пайпайна можно перейти на страницу crp-k8s.digdes.com и увидеть красоту.

![img_8.png](img_8.png)

В данном же репозитории лежит файл helm-chart-values.yaml, который применяется при установки релиза чарта. В нем можно менять настройки минио, локи, мониторинга в целом, отдельных сервисов самого приложения.

## Мониторинг

Пайплайн репозитория бутика содержит в себе установку операторов minio и kube-prometheus-stack, через зависимости чарта бутика ставится минио, локи и промтейл. Что очень удобно и позволяет посредством названий чартов использовать один файл в репозитрии бутика для их управления.

Конфигурация графаны
```
grafana:
  enabled: true
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: role
                operator: In
                values:
                  - monitoring
  adminPassword: prom-operator
  ingress:
    enabled: true
    hosts:
      - *default-crp-otus-project-host
    path: /grafana
  additionalDataSources:
    - name: Loki
      editable: true
      type: loki
      url: http://online-boutique-loki-gateway.online-boutique.svc.cluster.local:80
  grafana.ini:
    server:
      domain: *default-crp-otus-project-host
      root_url: "https://%(domain)s/grafana"
      serve_from_sub_path: "true"
```
Конфигурация локи
```
loki:
  create: true
  global:
    dnsService: "coredns"
  loki:
    server:
      grpc_server_max_recv_msg_size: 20971520
      grpc_server_max_send_msg_size: 20971520
    auth_enabled: false
    structuredConfig:
      memberlist:
        cluster_label: "loki"
      schema_config:
        configs:
          - from: "2020-09-07"
            store: tsdb
            object_store: "s3"
            schema: "v13"
            index:
              period: "24h"
              prefix: "loki_index_"
    compactor:
      compaction_interval: 10m
      retention_enabled: true
      retention_delete_delay: 2h
      retention_delete_worker_count: 150
      delete_request_store: s3
    limits_config:
      retention_period: 24h
      ingestion_rate_mb: 4
      ingestion_burst_size_mb: 6
    storage:
      bucketNames:
        chunks: loki
        ruler: loki
        admin: loki
      s3:
        s3: "http://minio:minio_secret_pass@minio.online-boutique.svc.cluster.local:9000"
        endpoint: http://minio-hl.online-boutique.svc.cluster.local:9000
        s3ForcePathStyle: true
        access_key_id: "minio"
        secret_access_key: "minio_secret_pass"
        insecure: true
        region: null
        sse_encryption: false
        http_config:
          idle_conn_timeout: 90s
          response_header_timeout: 0s
          insecure_skip_verify: true
    ingester:
      autoforget_unhealthy: true
    storage_config:
    memcached:
      chunk_cache:
        enabled: false
      results_cache:
        enabled: false
    commonConfig:
      replication_factor: 1

  resultsCache:
    enabled: false
  chunksCache:
    enabled: false

  write:
    replicas: 1
    autoscaling:
      enabled: false
    extraArgs:
      - "-log.level=info"
    maxUnavailable: 0
    persistence:
      volumeClaimsEnabled: false
      dataVolumeParameters:
        emptyDir: {}

  read:
    replicas: 1
    autoscaling:
      enabled: false
    extraArgs:
      - "-log.level=info"
    maxUnavailable: 0
    persistence:
      volumeClaimsEnabled: false
      dataVolumeParameters:
        emptyDir: {}

  backend:
    replicas: 1
    autoscaling:
      enabled: false
    extraArgs:
      - "-log.level=info"
    maxUnavailable: 0
    persistence:
      volumeClaimsEnabled: false
      dataVolumeParameters:
        emptyDir: {}

  lokiCanary:
    enabled: false

  sidecar:
    rules:
      enabled: false

  serviceMonitor:
    enabled: false

  monitoring:
    selfMonitoring:
      enabled: false

  test:
    enabled: false

  enterprise:
    enabled: false

  gateway:
    enabled: true
```
Конфигурация промтейл
```
promtail:
  create: true
  config:
    clients:
      - url: http://online-boutique-loki-gateway.online-boutique.svc.cluster.local/loki/api/v1/push
```


Для просмотра графы можно перейти по следующему адресу crp-k8s.digdes.com/grafana

Для просмотра минио - по адресу crp-k8s-minio.digdes.com

В целом вся эта информация предоставляется при деплое чарта в кубер.

![img_9.png](img_9.png)

Примеры мониторинга 


![img_10.png](img_10.png)


![img_11.png](img_11.png)

