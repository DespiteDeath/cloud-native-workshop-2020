# Управление Трафиком


В предыдущей части, мы рассмотрели как работают основные строительные блоки Service Mesh -  Istio Networking API, Virtual Services, Destination Rules.

В этой части, мы поработаем с более продвинутыми техниками маршрутизации трафика на базе Istio, нежели те которые доступны из коробки в Kubernetes.


## Обновленный coffee-shop

Для того чтобы продемострировать изменения в проекте, мы добавим вторую версию нашего coffee-shop.
До сих пор мы использовали только изначальную версию  (`version: v1`).

Далее мы добавим вторую версию нашего микросервисного примера, который будет себя вести немного по другому.

Однако эта версия не просто новая версия того же приложения, которую можно обновлять и развертывать с минимальным временем простоя (т.е. переход `v1` ->` v2`).

Скорее, мы специально хотим, чтобы обе версии были доступны в течение некоторого времени, например, для проведения A/B-тестирования или Канареечных релизов.

Ресурс orders сейчас возвращает тип кофе заглавными буквами, поскольку это перечислимое значение `CoffeeType`.
Мы поменяем в коде класс `OrdersResource` для вывода типов кофе строчными буквами.
После того как мы пересоберем проект на Maven, мы упакуем наше приложение в Docker образ с новым именем `coffee-shop:2`.

Мы создаем новый Deployment Definition для того чтобы развернуть наше новое приложение coffee-shop, при этом не трогая предыдущую версию:
```yaml
kind: Deployment
apiVersion: apps/v1
metadata:
  name: coffee-shop-v2
spec:
  selector:
    matchLabels:
      app: coffee-shop
      version: v2
  replicas: 1
  template:
    metadata:
      labels:
        app: coffee-shop
        version: v2
    spec:
      containers:
      - name: coffee-shop
        image: de.icr.io/cee-<your-name>-workshop/coffee-shop:2
        imagePullPolicy: Always
        ports:
        - containerPort: 9080
        livenessProbe:
          exec:
            command: ["sh", "-c", "curl -f http://localhost:9080/"]
          initialDelaySeconds: 20
        readinessProbe:
          exec:
            command: ["sh", "-c", "curl -s http://localhost:9080/health | grep -q coffee-shop"]
          initialDelaySeconds: 40
```

Pod-ы из этого Deployment-а будут промаркированы label-ами `app: coffee-shop` и `version: v2`.
Сервис coffee-shop просто перенаправит запросы на этот Pod. Однако сейчас этот трафик пойдет на предыдущую версию, поскольку Istio proxy настроены на нее.

Давайте убедимся, что новый Deployment и Pod-ы созданы:

```
kubectl get deployments
kubectl get pods
```


## A/B тестирование с Istio

A/B тестирование это метод при котором тестируются две версии одного сервиса для того чтобы понять какой из них лучше отвечает метрикам определенным в тесте.
Чтобы добавить вторую версию нашего сервиса мы изменим Destination Rule дабавив в него новый subset (`name: v2`):
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: coffee-shop
spec:
  host: coffee-shop
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
```

Далее применяем обновленный Destination Rule к кластеру.
Как вы заметили, ничего не изменилось с работой приложения.
Если вы посмотрите на текущий Virtual Service - весь входящий трафик настроен на subset `v1`:

```
kubectl get virtualservice coffee-shop --output yaml
```

Давайте поменяем правило маршрутизации указав также subset `v2`, но только для трафика который содержит HTTP заголовок от браузера Firefox:

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: coffee-shop
spec:
  hosts:
  - "*"
  gateways:
  - coffee-shop-gateway
  http:
  - match:
    - headers:
        user-agent:
          regex: '.*Firefox.*'
    route:
    - destination:
        host: coffee-shop
        port:
          number: 9080
        subset: v2
  - route:
    - destination:
        host: coffee-shop
        port:
          number: 9080
        subset: v1
```

Новое правило будет маршрутизировать трафик от браузеров Firefox на все эксземпляры которые находятся в subset `v2`, а остальной трафик игнорировать, который в свою очередь будет обрабатываться правилом по умолчанию маршрутизирущим трафик на subset `v1`.
В Istio для каждого сервиса может быть определен только один `VirtualService`, и соответственно, когда вы определяете несколько блоков в [HTTPRoute](https://istio.io/docs/reference/config/networking/virtual-service/#HTTPRoute), их порядок в котором они определены - важен.

Давайте применим эти изменения на кластере. 

Теперь мы видем другое поведение нашего сервиса, если отправляем запросы из Firefox.
Тоже самое можно сэмулировать из командной строки `curl`, если указать соответствующий загаловок в запросе:

```
curl <ip-address>:<node-port>/coffee-shop/resources/orders -i -XPOST \
  -H 'User-agent: Mozilla/5.0 (X11; Linux x86_64; rv:62.0) Gecko/20100101 Firefox/62.0' \
  -H 'Content-Type: application/json' \
  -d '{"type":"Espresso"}'

curl <ip-address>:<node-port>/coffee-shop/resources/orders \
  -H 'User-agent: Mozilla/5.0 (X11; Linux x86_64; rv:62.0) Gecko/20100101 Firefox/62.0'
```


## Канареечные Deployment-ы

Чтобы минимизировать риск от возможных ошибок в новых версиях сервисов используются канареечные deployment-ы. Новые версии сервисов постепенно становятся доступны т.е. малыми порциями обрабатывают траффик от пользователей.

Чтобы постепенно начать направлять трафик на новую версию coffee-shop сервиса, мы поменяем его Virtual Service:

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: coffee-shop
spec:
  hosts:
  - "*"
  gateways:
  - coffee-shop-gateway
  http:
  - route:
    - destination:
        host: coffee-shop
        port:
          number: 9080
        subset: v2
      weight: 30
    - destination:
        host: coffee-shop
        port:
          number: 9080
        subset: v1
      weight: 70
```

Как вы видете, трафик делится на два subset-а 70% для `v1` и 30% для `v2`.

Это правило может изменяться со временем, пока в результате весь трафик не будет обрабатываться новой версией сервиса.
Данные процесс как правило автоматизируется и реализуется в рамках Continuous Deployment конвейера.

Давайте применим новое правило маршрутизации и постепенно будем смещать процентаж трафика с `v1` к `v2`.

>**Примечание**
>
>Если вы будете использовать браузер, не забудьте делать 
>полное обновление страницы, чтобы избавиться от эффектов >кеширования браузером.

Вы должы наблюдать как coffee-shop переключается с первой на вторую версию в соответствии с весами указанными в новом правиле.

Теперь, когда мы научились управлять трафиком сервисов, давайте посмотрим как можно сделать наши микросервисы отказоустойчивыми в [следующем разделе](07-resiliency.md).
