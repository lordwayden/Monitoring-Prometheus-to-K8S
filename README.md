Поднятый и работающий кластер Kubernetes.
Установленный и работающий Helm.
helm list.

Установка

1. Создайте пространство имён и склонируйте Git-репозиторий prometheus-operator:

$ kubectl create ns monitoring
$ git clone https://github.com/coreos/prometheus-operator.git
$ cd prometheus-operator

2. Установите prometheus-operator deployment:

$ helm install --name prometheus-operator \ 
  --set rbacEnable=true --namespace=monitoring helm/prometheus-operator

3. Установите Prometheus и Alertmanager specs, а также Grafana deployment:

$ helm install --name prometheus --set serviceMonitorsSelector.app=prometheus \ 
  --set ruleSelector.app=prometheus --namespace=monitoring helm/prometheus
$ helm install --name alertmanager --namespace=monitoring helm/alertmanager
$ helm install --name grafana --namespace=monitoring helm/grafana

4. Установите kube-prometheus, чтобы загрузить предопределённые k8s exporters и serviceMonitors:

$ helm install --name kube-prometheus --namespace=monitoring helm/kube-prometheus

Если всё прошло успешно, можно запустить эту команду для вывода списка приложений:

$ kubectl get pods -n monitoring
NAME                                                      READY     STATUS    RESTARTS   AGE
alertmanager-alertmanager-0                               2/2       Running   0          3m
grafana-grafana-3066287131-brj8n                          2/2       Running   0          4m
kube-prometheus-exporter-kube-state-2696859725-s8m56      2/2       Running   0          3m
kube-prometheus-exporter-node-029w0                       1/1       Running   0          3m
kube-prometheus-exporter-node-n3txz                       1/1       Running   0          3m
kube-prometheus-exporter-node-q2rk3                       1/1       Running   0          3m
prometheus-operator-prometheus-operator-514889780-qm3fp   1/1       Running   0          4m
prometheus-prometheus-0                                   2/2       Running   0          3m

Prometheus

Пробросьте сервер Prometheus на свой компьютер, чтобы получить доступ к панели через http://localhost:9090:

$ kubectl port-forward -n monitoring prometheus-prometheus-0 9090



В панели Prometheus можно делать запросы к метрикам, просматривать предопределённые уведомления и целевые объекты Prometheus.

Обратите внимание: Если какие-то цели возвращают ошибку недоступности, проверьте группы безопасности и правила firewall. Если у вас нет целей, представленных на скриншоте выше, проверьте лейблы подов K8s, т.к. иногда утилиты, используемые для деплоя кластера, не ставят их.

Обратите внимание (№2): В проекте prometheus-operator работают над упаковкой стандартных уведомлений для K8s в Helm chart. Однако сейчас для их загрузки требуется выполнить последовательность из команд ниже (в будущем эта необходимость пропадёт):

$ sed -ie 's/role: prometheus-rulefiles/app: prometheus/g' contrib/kube-prometheus/manifests/prometheus/prometheus-k8s-rules.yaml
$ sed -ie 's/prometheus: k8s/prometheus: prometheus/g' contrib/kube-prometheus/manifests/prometheus/prometheus-k8s-rules.yaml
$ sed -ie 's/job=\"kube-controller-manager/job=\"kube-prometheus-exporter-kube-controller-manager/g' contrib/kube-prometheus/manifests/prometheus/prometheus-k8s-rules.yaml
$ sed -ie 's/job=\"apiserver/job=\"kube-prometheus-exporter-kube-api/g' contrib/kube-prometheus/manifests/prometheus/prometheus-k8s-rules.yaml
$ sed -ie 's/job=\"kube-scheduler/job=\"kube-prometheus-exporter-kube-scheduler/g' contrib/kube-prometheus/manifests/prometheus/prometheus-k8s-rules.yaml
$ sed -ie 's/job=\"node-exporter/job=\"kube-prometheus-exporter-node/g' contrib/kube-prometheus/manifests/prometheus/prometheus-k8s-rules.yaml
$ kubectl apply -n monitoring -f contrib/kube-prometheus/manifests/prometheus/prometheus-k8s-rules.yaml

Grafana

Для отладочных целей у Prometheus есть expression browser. Чтобы получить красивую панель, воспользуйтесь Grafana со встроенной возможностью выполнять запросы в Prometheus.

Обратите внимание: В проекте prometheus-operator работают над созданием простого deployment для Grafana, вероятно, с использованием нового CRD. На данный момент для её настройки требуется выполнить следующие команды (в будущем эта необходимость пропадёт):

$ sed -ie 's/grafana-dashboards-0/grafana-grafana/g' https://raw.githubusercontent.com/coreos/prometheus-operator/master/contrib/kube-prometheus/manifests/grafana/grafana-dashboards.yaml
$ sed -ie 's/prometheus-k8s.monitoring/prometheus-prometheus.monitoring/g' https://raw.githubusercontent.com/coreos/prometheus-operator/master/contrib/kube-prometheus/manifests/grafana/grafana-dashboards.yaml
$ kubectl apply -n monitoring -f https://raw.githubusercontent.com/coreos/prometheus-operator/master/contrib/kube-prometheus/manifests/grafana/grafana-dashboards.yaml
$ kubectl port-forward -n monitoring $(kubectl get pods --selector=app=grafana-grafana -n monitoring --output=jsonpath={.items..metadata.name})  3000

Подождите несколько секунд, пока Grafana загрузит данные, откройте http://localhost:3000 в браузере и изучайте замечательные графики!


Grafana: доступные dashboards для  Kubernetes


Grafana: графики для планирования занятости/производительности  Kubernetes

Alertmanager

Alertmanager обслуживает уведомления, отправляемые клиентскими приложениями вроде сервера Prometheus. Он обеспечивает устранение дублей, группировку, отправку в правильный сервис-получатель вроде электронной почты, PagerDuty или OpsGenie. Также он отвечает за заглушение (silence) и подавление (inhibit) уведомлений.

Мы уже установили Alertmanager командами выше, и осталось пробросить порт сервиса на ваш компьютер, после чего можно будет открывать http://localhost:9093 в веб-браузере:

$ kubectl port-forward -n monitoring alertmanager-alertmanager-0 9093
