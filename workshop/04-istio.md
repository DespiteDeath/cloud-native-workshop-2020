# Istio

Istio — это service mesh технология, которая позволяет нам добавлять 
дополнительный функционал к нашим микросервисам без изменения их реализации.
Istio добавляет sidecar-контейнеры к каждому запущенному pod-у, которые действует
как прокси для наших приложений. Каждый запущенный pod получает отдельный проксирующий
sidecar-контейнер, который перехватывает, проверяет и функционально усиливает соединение.

Istio добавляет route по умолчанию между Kubernetes сервисами.
Тем не менее рекомендуется явно добавлять route Istio между желаемыми сервисами.

Istio интегрируется с Kubernetes, расширяя API-модель Kubernetes, добавлением 
своих ресурсов. Это позволяет разработчикам использовать ресурсы Istio так, как 
если бы они были включены в Kubernetes, `kubectl` CLI.

Ресурсы Istio также определены в формате YAML и очень похожи на ресурсы Kubernetes.
Мы помещаем следующие файлы в те же каталоги, что и Kubernetes definitions, 
либо в отдельные файлы, либо по одному на каждый ресурс.

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
mesh-а. По умолчанию все Kubernetes сервисы являются частью service mesh.

Virtual Service barista определяет, что все запросы, которые обращаются к
сервису barista, направляются в экземпляры barista с subset-ом (то есть дополнительным 
label-ом)`v1`. На практике это действует как маршрут по умолчанию. Subset-ы, среди 
других политик запросов, определены в Destination Rules соответствующих сервисов.

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

Таким образом, pod-ы, которые содержат желаемые label-ы (здесь это `version`)
cчитаются частью subset-а. Использование label-а `version` — еще одна best practice.

Мы можем поместить эти определения в один файл, например с именем 
`routing.yaml` или в отдельные.

Чтобы этот пример работал, наши pod-ы должны содержать label.
Поэтому мы должны изменить и повторно применить оба Deployment Definitions.

Мы добавим label `version: v1` в раздел metadata для обоих наших приложений
(файлы YAML). Затем мы снова обновим deployment definitions для нашего 
кластера (`kubectl apply`).

## Gateway


Чтобы сделать наше приложение доступным извне service mesh, необходимо создать gateway. Использование Istio Gateway вместо Kubernetes Ingress, который мы использовали ранее, позволяют Istio проверять и маршрутизировать траффик.

Создадим Istio routing definition со следующим содержанием, чтобы определить gateway,
который направляет траффик в кластер:

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

Gateway задает wildcard хост (`*`), который соответствует всем 
именам хостов или IP-адресам. Он должен быть связан с Virtual Service
через свойство `gateways`. Это будет частью спецификации coffee-shop's 
Virtual Service.

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

Маршрутизация теперь выглядит немного иначе. Поскольку мы определяем 
конкретный gateway, только траффик от этого gateway будет направляться
на этот Virtual Service, а не из любого другого сервиса внутри mesh-a.
(для этого потребуется также явно добавить gateway по умолчанию в `mesh`)

Правило маршрутизации HTTP также требует, чтобы мы указали номер порта,
поскольку входящий трафик исходил из другого порта (`80`).

Следующее описание определяет Destination Rule, которое похоже на barista’s 
Destination Rule.

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

### Доступ к приложениям

Если мы хотим запустить наше приложение и получить к нему доступ через service 
mesh, необходимо получить доступ к gateway извне кластера. Это требует того, 
чтобы gateway, Virtual Service и Destination Rule были применены к service mesh.

Если мы создали бесплатный кластер, нам снова потребуется доступ к (gateway)сервису
через IP-адрес узла кластера. Таким образом, мы получаем node port сервиса
`istio-ingressgateway`.

    kubectl get services -n istio-system istio-ingressgateway

Мы можем получить HTTP/2 node port напрямую, используя следующий шаблон Go:

    kubectl get services -n istio-system istio-ingressgateway --template '{{range .spec.ports}}{{if eq .name "http2"}}{{.nodePort}}{{end}}{{end}}'

> **Примечание**
>
> В качестве напоминания, как видно из последнего раздела, мы получим 
> IP-адрес узла с помощью одной из следующих команд.
>
>     ibmcloud ks workers cloud-native
>
> Если у вас установлен `jq` CLI, вы также можете напрямую извлечь IP-адрес, вызвав
>
>     ibmcloud ks workers cloud-native --json | yq -r '.[0].publicIP'

Затем мы можем получить доступ к сервису, используя IP-адрес узла и
порт сервиса `istio-ingressgateway`:

    curl <ip-address>:<node-port>/health -i
    ...
    curl <ip-address>:<node-port>/coffee-shop/resources/orders -i

Этот сценарий полностью работает без Kubernetes ingress.
Кроме deployment-ов и service-ов требуются только Istio ресурсы.

Аналогичным образом мы можем использовать `/orders` для создания новых заказов на кофе

    curl <ip-address>:<node-port>/coffee-shop/resources/orders -i -XPOST \
      -H 'Content-Type: application/json' \
      -d '{"type":"Espresso"}'

> **Примечание**
>
> Если у вас есть платный кластер, мы можем получить IP-адресс шлюза
> через IP-адрес балансировщика нагрузки службы `istio-ingressgateway`:
> 
>     kubectl get services -n istio-system istio-ingressgateway \
>       -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
>
> Мы используем этот IP-адрес и HTTP-порты по умолчанию (`80` or `443`,
> соответственно) для доступа к приложению из-вне кластера:
>
>     curl <gateway-ip-address>/health -i

Теоретически, это означает, что оба наших сервиса работают как положено
и могут общаться друг с другом. Однако этого предположения вряд ли достаточно
для системы, работающей в продуктиве.

Давайте рассмотрим, как Istio улучшает наблюдаемость [в следующем разделе](05-istio-observability.md).
