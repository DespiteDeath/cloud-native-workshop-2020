# Istio

Istio — это service mesh технология, которая позволяет нам добавлять 
дополнительный функционал к нашим микросервисам без изменения их реализации.
Istio добавляет sidecar-контейнеры к каждому запущенному Pod-у, которые действует
как прокси для наших приложений. Каждый запущенный Pod получает отдельный проксирующий
sidecar-контейнер, который перехватывает, проверяет и функционально усиливает соединение.

Istio, по умолчанию, добавляет Route между сервисами в Kubernetes.
Тем не менее рекомендуется явно добавлять Route Istio между желаемыми сервисами.

Istio интегрируется с Kubernetes, расширяя API-модель Kubernetes, добавлением 
своих ресурсов. Это позволяет разработчикам использовать ресурсы Istio так, как 
если бы они были включены в `kubectl` CLI.

Ресурсы Istio тоже описываются в YAML формате и очень похожи на ресурсы Kubernetes.
Мы создадим описания ресурсов Istio в том же каталоге, что и Kubernetes манифесты, 
либо в ввиде одного файла, либо по одному на каждый ресурс.

## Virtual Service

Создадим Istio Virtual Service для приложения barista:

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: barista
spec:
  hosts:
  - barista
  http:
  - route:
    - destination:
        host: barista
        subset: v1
---
```

Virtual Service определяет правила маршрутизации для сервиса, который является частью
сервисной сетки. По умолчанию все Kubernetes сервисы являются частью сервисной сетки.

`Virtual Service barista` определяет, что все запросы, которые обращаются к
сервису `barista`, направляются к Pod-ам barista с subset-ом (то есть дополнительным 
label-ом)`v1`. На практике это действует как маршрут по умолчанию. Subset-ы описываются в Destination Rules соответствующих сервисов.

## Destination Rule

Создадим Destination Rule для приложения barista которое определяет subset `v1`.

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: barista
spec:
  host: barista
  subsets:
  - name: v1
    labels:
      version: v1
---
```

Таким образом, Pod-ы, которые содержат желаемые label-ы (здесь это `version`)
cчитаются частью subset-а. Использование label-а `version` не обязательно, но считается реомендованным способом.

Мы можем сохранить эти два определения Istio ресурсов в один файл `barista/deployment/routing.yaml` или по отдельности.

Чтобы наш пример сработал, Pod-ы должны быть помечены label-ами `version: v1`. Обратите внимание, что мы это уже сделали в Deployment манифестах  в разделе `metadata` для обоих приложений.

```yaml
...
template:
    metadata:
      labels:
        app: barista
        version: v1
    spec:
      containers:
      - name: barista
        image: de.icr.io/cee-albert-workshop/barista:1
...
```

Теперь применим эти изменения в кластере `kubectl apply -f barista/deployment/`.

## Gateway для входящего трафика


Чтобы сделать наше приложение доступным извне, необходимо создать шлюз для входящего трафика и свзяать его с приложением `coffee-shop`. Использование Istio Gateway вместо Kubernetes Ingress, позволит Istio анализировать и маршрутизировать входящий траффик.

Создадим описание ресурса Istio Gateway со следующим содержанием:

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: coffee-shop-gateway
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "*"
---
```

Gateway задает wildcard хост (`*`), который соответствует всем именам хостов или IP-адресам. Он должен быть связан с `Virtual Service coffee-shop` через свойство `gateways`. Это будет частью спецификации Virtual Service для приложения `coffee-shop`.

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
        subset: v1
---
```

Маршрутизация теперь будет по другому. Поскольку мы определяем конкретный gateway, то только траффик от этого gateway будет направляться на `Virtual Service coffee-shop`, а не от любого другого сервиса из сервисной сетки.

Для правила маршрутизации HTTP требуется чтобы мы указали номер порта, поскольку входящий трафик заходит с другого порта (`80`).

Следующее описание определяет Destination Rule coffee-shop, которое похоже на Destination Rule для приложения barista:

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
---
```

Сохраним спецификации ресурсов Istio в файл `coffee-shop/deployment/routing.yaml` и применим их к кластеру `kubectl apply -f barista/deployment/`.



### Доступ к приложениям

Если мы создали бесплатный (`Light`) кластер, то для того чтобы получить доступ к Gateway сервису, который мы только что создали, нам потребуется узнать nodePort сервиса реализуещго логику Ingress Gateway. По умолчанию, при установке Istio создается сервис  `istio-ingressgateway`. Его nodePort надо необходимо узнать следующей командой:
```
kubectl get services -n istio-system istio-ingressgateway
```

Либо мы можем получить HTTP/2 node port напрямую, используя следующий запрос:
```
kubectl get services -n istio-system istio-ingressgateway --template '{{range .spec.ports}}{{if eq .name "http2"}}{{.nodePort}}{{end}}{{end}}'
```

> **Примечание**
>
> В качестве напоминания, как вы помните, в предыдущем разделе мы получали публичный IP адрес нашего узла кластера с помощью следующей команды: 
>
>     ibmcloud ks workers --cluster cloud-native
>
>
> Если у вас установлен `jq`, вы можете напрямую извлечь IP-адрес вызвав:
>
>     ibmcloud ks workers --cluster cloud-native --json | jq -r '.[0].publicIP'


Теперь мы можем получить доступ к нашему сервису, используя IP-адрес узла и 
порт сервиса `istio-ingressgateway`:
```
curl <ip-address>:<node-port>/health -i

curl <ip-address>:<node-port>/coffee-shop/resources/orders -i
```

Аналогичным образом мы можем использовать `/orders` для создания новых заказов на кофе

    curl <ip-address>:<node-port>/coffee-shop/resources/orders -i -XPOST \
      -H 'Content-Type: application/json' \
      -d '{"type":"Latte"}'

Этот сценарий полностью работает без Kubernetes ingress. Теперь только требуются Istio ресурсы, кроме deployment-ов и service-ов .


> **Примечание**
>
> Если у вас платный кластер (`Standard`), то чтобы получить IP-адрес Gateway нужно узнать IP-адрес LoadBalancer у сервиса `istio-ingressgateway`:
> 
>     kubectl get services -n istio-system istio-ingressgateway \
>       -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
>
> Для доступа к приложению извне мы должны использовать этот IP-адрес и HTTP-порты по умолчанию (`80` или `443`):
>
>     curl <gateway-ip-address>/health -i

Теоретически, это означает, что оба наших сервиса работают как положено
и могут общаться друг с другом. Однако этого предположения вряд ли достаточно
для системы, работающей в продуктиве.

Давайте рассмотрим, как Istio помогает наблюдать за приложениями [в следующем разделе](05-istio-observability.md).
