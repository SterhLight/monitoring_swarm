# МОНИТОРИНГ НА DOCKER-SWARM
## ПОШАГОВОЕ КОНФИГУРИРОВАНИЕ ЧЕРЕЗ CLI РУЧКАМИ
## ПОДГОТОВКА
### Создание директорий для хранения файлов конфигупации и volums
/opt/monitoring/grafana  
/opt/monitoring/prometheus  
/opt/monitoring/alertmanager  
/opt/monitoring/cadvisor  
/opt/monitoring/node_exporter  
```
mkdir -p /opt/monitoring/grafana && mkdir -p /opt/monitoring/prometheus \
&& mkdir -p /opt/monitoring/alertmanager && mkdir -p /opt/monitoring/cadviso \
&& mkdir -p /opt/monitoring/node_exporter
```
***user=ansible_user***  
***group=ansible_user***  
***mode=0775***  
```
chown -R $ansible_user:$ansible_user /opt/monitoring/
chmod -R 0775 /opt/monitoring/
```
### Создание конфигов
#### Копирование конфигов в созданные дирректории
***bash***  
/opt/monitoring/prometheus/prometheus.yml  
/opt/monitoring/grafana/grafana.ini  
/opt/monitoring/alertmanager/alertmanager.yml
#### Создание служб конфигов
***bash***
```
docker config create prometheus /opt/monitoring/prometheus/prometheus.yml
docker config create grafana /opt/monitoring/grafana/grafana.ini
docker config create alertmanager.yml /opt/monitoring/alertmanager/alertmanager.yml
```
### Создание volumse
***bash***
```
docker volume create prometheus_data
docker volume create grafana_lib
docker volume create grafana_datasources
docker volume create grafana_dashboards
docker volume create grafana_plugins
```
### Создание сети
***bash***
```
docker network create -d overlay monitoring_network
```
### Пометить сервер (создание labels)
***bash***
```
docker node update --label-add key1=monitoring sv1
```
## ПОДНЯТИЕ NODE-EXPORTER
```
docker service create \
  --mount type=bind,source=/,destination=/rootfs:ro \
  --mount type=bind,source=/proc,destination=/host/proc:ro \
  --mount type=bind,source=/sys,destination=/host/sys:ro \
  --network monitoring_network -p 9100:9100/tcp \
  --replicas=5 \
  --name=node-exporter prom/node-exporter:v1.6.0
```
## ПОДНЯТИЕ cADVISOR
```
docker service create \
  --mount type=bind,source=/,destination=/rootfs:ro \
  --mount type=bind,source=/var/run,destination=/var/run:ro \
  --mount type=bind,source=/sys,destination=/sys:ro \
  --mount type=bind,source=/var/lib/docker/,destination=/var/lib/docker:ro \
  --mount type=bind,source=/dev/disk/,destination=/dev/disk:ro \
  --network=monitoring_network \
  -p=8080:8080 \
  --replicas=5 \
  --name=cadvisor gcr.io/cadvisor/cadvisor:v0.47.2
```
## ПОДНЯТИЕ PROMETHEUS
```
docker service create \
--network monitoring_network -p 9090:9090 \
--mount type=volume,src=prometheus_data,dst=/prometheus \
--config source=prometheus,target=/etc/prometheus/prometheus.yml \
--replicas=1 \
--constraint node.labels.key1==monitoring \
--name=prometheus prom/prometheus:v2.44.0
```
## ПОДНЯТИЕ GRAFANA
```
docker service create \
  --network monitoring_network -p 3000:3000 \
  --mount type=volume,source=grafana_lib,destination=/var/lib/grafana \
  --mount type=volume,source=grafana_datasources,destination=/etc/grafana/datasources \
  --mount type=volume,source=grafana_dashboards,destination=/etc/grafana/dashboards \
  --mount type=volume,source=grafana_plugins,destination=/var/lib/grafana/plugins \
  --replicas=1 \
  --config source=grafana,target=/etc/grafana/grafana.ini \
  --constraint node.labels.key1==monitoring \
  --name grafana grafana/grafana:10.0.0-ubuntu
```
## ПОДНЯТИЕ ALERTMANAGER
```
docker service create \
  --network monitoring_network -p 9093:9093 \
  --replicas=1 \
  --config source=alertmanager.yml,target=/etc/alertmanager/config.yml \
  --constraint node.labels.key1==monitoring \
  --name alertmanager prom/alertmanager:v0.25.0
```
