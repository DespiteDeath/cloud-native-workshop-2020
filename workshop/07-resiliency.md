Отказоустойчивость
==================

В этом разделе мы обсудим отказоустойчивость в мире микросервисов.
Мы посмотрим, какие решения, прозрачные для приложения, предлагает нам Istio.
А так же, как мы могли бы реализовать отказоустойчивость в нашем приложении.

Чтобы не запутаться в разных версиях coffee-shop `v1` и` v2`, давайте сначала вернемся к исходному состоянию и будем работать только с subset `v1`.  

Таким образом мы должны поменять Virtual Service, Destination Rule в исходное состояние (`kubectl apply`) и удалить Deployment `coffee-shop-v2` (`kubectl delete deployment coffee-shop-v2`). 

После того как Deployment будет удален, Pod-ы версии 2 так же будут удалены автоматически.

> **Причмечание**
>Несмотря на то что мы можем поменять содержимое файлов конфигурации инфраструктуры (`infrastructure-as-code`) и применить их через `kubectl apply`, Kubernetes не сможет отследить удаление ресурсов в файлах, поскольку он только создает и обновляет конфигурации указанных ресусрсов.
>Таким образом нам нужно явно `kubectl delete` ненужные ресурсы.


Фичи для отказоустойчивости
======

Istio предоставляет фичи для отказоусточивости сервисов, например такие как, timeout-ы, circuit breaker-ы, повторы запросов.
Поскольку sidecar прокси перехватывают все входящие и исходяшие подключения к контейнеру приложения, то они полностью контроллируют трафик по HTTP.

Это означает, что если, например, наше приложение не поддерживает timeout соединения, то  Istio сможет это обеспечить без изменения кода приложения.

Timeout
========

Istio позволяет определить timeout на уровне правил. Для этого, нам нужно воспользоваться правилами на уровне Virtual Service.

Давайте добавим timeout 1 секунда для соединений к нашему coffee-shop сервису и применим обновленный Virtual Service на нашем кластере:

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
    timeout: 1s
```

>**Примечание**
>
>Определение `timeout` содержится в специальном YAML объекте `route`.

Теперь общее время запросов за кофе не будет превышать одну секунду иначе sidecar прокси будет возврашать HTTP ошибку.


Testing resiliency
==================

The challenge, however, is now to see whether our changes took effect as desired.
We'd expect our applications to respond in much less than one second, therefore we would not see that error situation until it's in production.
Luckily, Istio ships with functionality that purposely produces error situations, in order to test the resiliency of our services.

The sidecars have two main means to do that: adding artificial delays, and failing requests.
We can instruct the routing rules to add these fault scenarios, if required only on a given percentage of the requests.

We modify the barista virtual service to add a 3 seconds delay for 50% of the requests:

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
    fault:
      delay:
        fixedDelay: 3s
        percent: 50
```

If we apply this updated resource to the cluster, we will notice that some of the connections will in fact fail, after roughly 1 second.
We don't see any request taking the whole 3 seconds, due to the timeout on the coffee-shop routing rules:

```
while true; do
curl <ip-address>:<node-port>/coffee-shop/resources/orders -i -XPOST \
  -H 'Content-Type: application/json' \
  -d '{"type":"Espresso"}' \
  | grep HTTP
sleep 1
done
```

>**Примечание**
>
>The `fault` property is only meant for testing purposes. Please don't apply this to any other environment where you don't want connections to be slowed down or to randomly fail.

Besides the obvious responses, we can also use our observability tools to inspect what is happening.
Have a look at the Grafana dashboards and the Jaeger traces again, to see how the failing requests are made visible.

This lab only covers timeouts and basic faults.
Istio also offers functionality for retries and circuit breakers which are also applied and configured declaratively via Istio resources.
Have a look at the further resources to learn more.


Application level
=================

Building a resilient microservice is key when designing microservices.
Apart from the infrastructure resilience, sometimes more fine-grained application level resilience is required.

[MicroProfile Fault Tolerance](https://github.com/eclipse/microprofile-fault-tolerance/) provides a simple yet flexible solution to build a resilient microservice at the application-level.
It offers capabilities for timeouts, retries, bulkheads, circuit breakers and fallback.

In general, application-level resiliency is more fine-grained while Istio's behavior is more coarse-grained.
As a recommendation, we can use MicroProfile together with Istio's fault handling.

Looks like we've finished the last section! [Conclusion](08-conclusion.md).
