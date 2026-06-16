# Дипломная работа по профессии «Системный администратор» - `Пономарев Денис`

# ЗАДАНИЕ:
Ключевая задача — разработать отказоустойчивую инфраструктуру для сайта, включающую мониторинг, сбор логов и резервное копирование основных данных. Инфраструктура должна размещаться в Yandex Cloud и отвечать минимальным стандартам безопасности.

<b>Само задание доступно по ссылке:</b>
https://github.com/netology-code/sys-diplom/tree/diplom-zabbix

# РЕШЕНИЕ

## Сайт

* Мною было развёрнуто 6 виртуальных машин.
* Серверы Web и Elasticsearch находятся в приватных подсетях.
* Серверы Zabbix, Kibana и Application Load Balancer размещены в публичной подсети.
* Web-серверы (web-1 и web-2) находятся в разных зонах доступности для обеспечения отказоустойчивости.

![ВМ](materials/vm/01.png)


<b>Установлен Nginx:</b>

![web-server-1](materials/nginx/01.png)

![web-server-2](materials/nginx/02.png)

<b>Настройка балансировщика:</b>

1. Создайте Target Group, включите в неё две созданных ВМ:

![tg-group](materials/balancer/01.png)

2. Создайте Backend Group, настройте backends на target group, ранее созданную. Настройте healthcheck на корень (/) и порт 80, протокол HTTP.

![web-backend](materials/balancer/02.png)

3. Создайте HTTP router. Путь укажите — /, backend group — созданную ранее.

![web-router](materials/balancer/03.png)

4. Создайте Application load balancer для распределения трафика на веб-сервера, созданные ранее. Укажите HTTP router, созданный ранее, задайте listener тип auto, порт 80.

![web-router](materials/balancer/04.png)

<b>Протестируйте сайт curl -v <публичный IP балансера>:80</b>

![web-router](materials/balancer/05.png)


## Мониторинг

1. Создайте ВМ, разверните на ней Zabbix. На каждую ВМ установите Zabbix Agent, настройте агенты на отправление метрик в Zabbix.

![zabbix](materials/zabbix/01.png)

<b>Доступ:</b><br>
Сайт: http://89.169.156.132:8080/<br>
Логин: Admin<br>
Пароль: zabbix

2. Настройте дешборды с отображением метрик, минимальный набор — по принципу USE (Utilization, Saturation, Errors) для CPU, RAM, диски, сеть, http запросов к веб-серверам. Добавьте необходимые tresholds на соответствующие графики.

![zabbix](materials/zabbix/02.png)

## Логи

1. Cоздайте ВМ, разверните на ней Elasticsearch.

Из-за недоступности продуктов HashiCorp, elasticsearch был развёрнут через докер-образ:

![elasticsearch](materials/elasticsearch/01.png)

2. Установите filebeat в ВМ к веб-серверам, настройте на отправку access.log, error.log nginx в Elasticsearch:

```
filebeat.inputs:
- type: filestream
  id: nginx-logs
  paths:
    - /var/log/nginx/*.log
  fields:
    log_source: nginx-docker-host
  tags: ["nginx-logs", "nginx"]
```

* Блок filebeat.inputs настраивает сбор логов.
* type: filestream отвечает за чтение логов, как поток файлов.
* paths: ["/var/log/nginx/*.log"]: Данная строка указывает Filebeat, какие файлы читать. В данном случае это все файлы с расширением .log, которые находятся в директории /var/log/nginx/.
* tags: ["nginx-logs", "nginx"]: Данные теги будут добавлены к каждому собранному событию.

```
output.elasticsearch:
  hosts: ["http://{{ hostvars['elasticsearch-vm']['ansible_host'] }}:9200"]
```

* output.elasticsearch: Этот блок отвечает за отправку логов в Elasticsearch.
* hosts: ["http://{{ hostvars['elasticsearch-vm']['ansible_host'] }}:9200"]: Данная команда указывает, куда необходимо отправлять логи.

3. Создайте ВМ, разверните на ней Kibana.

Из-за недоступности продуктов HashiCorp, kibana был развёрнут через докер-образ:

![kibana](materials/kibana/01.png)

4. Сконфигурируйте соединение с Elasticsearch.

```
ELASTICSEARCH_HOSTS: "http://{{ hostvars[groups['elasticsearch'][0]]['ansible_host'] }}:9200"
```
Здесь была использована переменная окружения, которую Kibana использует для определения адреса, по которому находится Elasticsearch.

<b>Доступ:</b><br>
Сайт: http://51.250.88.31:5601/

![сайт](materials/kibana/02.png)


## Сеть

1. Разверните один VPC. Сервера web, Elasticsearch поместите в приватные подсети. Сервера Zabbix, Kibana, application load balancer определите в публичную подсеть.

![сеть](materials/network/01.png)

2. Настройте Security Groups соответствующих сервисов на входящий трафик только к нужным портам. Настройте ВМ с публичным адресом, в которой будет открыт только один порт — ssh.

![bastion](materials/bastion/01.png)

## Резервное копирование

![bastion](materials/snapshots/01.png)
