Kubernetes
==========

Чтобы запустить наши приложения в кластере Kubernetes, нам нужно создать определенные ресурсы.
В конечном счете все, что мы делаем - это определяем нашу среду исполнения, используя Infrastructure-as-Code, аналогично тому, что мы делали с использованием файлов `Dockerfile`, но на этот раз для всей среды.

Сначала нам надо определить, сколько реплик наших отдельных образов приложений мы хотим запустить, как они подключены, настроены и т.д.
Все это хранится так же, как и программный код, который должен находиться в нашем проекте.

Давайте рассмотрим ресурсы, которые нам нужно создать.

## Pod-ы

Pod — это наименьший (базовый) модуль в Kubernetes. Это абстракция над нашим экземпляром приложения, которая содержит **_обычно_** один контейнер.
То есть каждый из контейнеров Docker, которые мы создали в прошлом разделе, теперь будет работать внутри контейнеров Kubernetes, по одному на каждый экземпляр контейнера.
Если мы хотим иметь несколько копий наших приложений, мы создаем несколько pod-ов.

Ниже приведен фрагмент кода YAML, описывающего pod.
Пока создавать pod мы не будем, но давайте пройдемся по этому файлу, чтобы понять основные принципы.

```yaml
# ...
metadata:
  labels:
    app: coffee-shop
spec:
  containers:
  - name: coffee-shop
    image: de.icr.io/cee-<your-name>-workshop/coffee-shop:1
    ports:
    - containerPort: 9080
  restartPolicy: Always
# ...
```
Фрагмент выше определяет спецификацию одного pod-а, который содержит  контейнер, созданный из образа Docker coffee-shop.

Pod'ы в Kubernetes обычно не перезапускают. Если pod по какой-то причине останавливается, его уничтожают и создают заново.

Чтобы убедиться, что нам не нужно пересоздавать ресурсы модуля вручную, мы используем контроллеры, которые следят за созданием необходимого количества копий pod-ов - Kubernetes deployments.

## Deployments (развертывание)

Создадим Kubernetes deployment - эта сущность управляет одной или несколькими репликами наших микросервисов.
Взгляните на конфигурацию deployment:

```yaml
kind: Deployment
apiVersion: apps/v1
metadata:
  name: coffee-shop
spec:
  selector:
    matchLabels:
      app: coffee-shop
      version: v1
  replicas: 1
  template:
    metadata:
      labels:
        app: coffee-shop
        version: v1
    spec:
      containers:
      - name: coffee-shop
        image: de.icr.io/cee-<your-name>-workshop/coffee-shop:1
        ports:
        - containerPort: 9080
```

Этот deployment выглядит аналогично описанию pod-а, приведенному ранее, но здесь pod описывается в секции template (шаблон), по которому будут создаваться новые реплики pod-а.
Deployment обеспечиват развертывание необходимого количества копий pod-ов.
Если pod по какой-то причине остановился (специально или нет - бне имеет значения), deployment автоматически сделает замену остановленному pod-у, то есть создаст новую его копию.

Если мы хотим масштабировать наше приложение, изменить конфигурацию deployment, или развернуть другую версию приложения (используя другой образ Docker), мы просто изменяем YAML-конфигурацию deployment и обновляем его в кластере.
Kubernetes позаботится о том, чтобы такие изменения были правильно отработаны, с минимально возможным временем простоя.


## Контроль развертывания (liveness и readiness)

Для pod-а, в котором работает приложение Java, потребуется несколько минут, чтобы он был полностью готов к работе. Поскольку Kubernetes ничего не знает о содержимом работающего контейнера, он может только предполагать, что работающие модули сразу могут обрабатывать входящие запросы.
Однако это конечно же не так, и поэтому мы должны каким-то образом сообщить, когда контейнер будет полностью готов делать что-то полезное.

С этой целью мы включаем сервисы жизнеспособности (liveness) и готовности (readiness) в deployment.

Сервис жизнеспособности сообщает Kubernetes, работает ли  модуль в целом (для нас это - сервер приложений).
Если это не так, Kubernetes немедленно остановит pod и заменит его на новую копию.
Проверка готовности сообщает, готов ли модуль для выполнения полезной работы, то есть обработки входящего трафика.

Существуют разные типы таких [сервисов контроля развертывания](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#container-probes).

Мы будем использовать функцию exec, которая позволяет выполнить произвольный скрипт внутри контейнера.
Команды `curl` будут подключаться к серверам приложений и ресурсам проверки работоспособности соответственно.

Взглянем на окончательную конфигурацию deployment:

```yaml
kind: Deployment
apiVersion: apps/v1
metadata:
  name: coffee-shop
spec:
  selector:
    matchLabels:
      app: coffee-shop
      version: v1
  replicas: 1
  template:
    metadata:
      labels:
        app: coffee-shop
        version: v1
    spec:
      containers:
      - name: coffee-shop
        image: de.icr.io/cee-<your-name>-workshop/coffee-shop:1
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

Файлы YAML с этим контентом надо ссоздать в папках `deployment/` двух проектов микросервисов.
Один deployment будет называться `coffee-shop` (то описание, которое мы только что обсудили выше), а другой - `barista`.
Убедитесь, что все имена, метки, изображения и URL указаны верно.

Теперь мы наконец создадим эти ресурсы в нашем кластере Kubernetes.
Для этого нужно просто применить эти файлы к конфигурации кластера командой `kubectl apply`:

```
kubectl apply -f coffee-shop/deployments/
kubectl apply -f barista/deployments/
```

Такая команда применит (создаст - если таких пока нет, или обновит - если уже есть), все ресурсы, которые находятся в соответствующем каталоге.

Вы можете проверить, были ли ресурсы созданы успешно, запросив текущий статус deployment-ов и pod-ов:

```
kubectl get pods
kubectl get deployments
```
После короткого периоада запуска вы должны увидеть два pod-а, один - coffee-shop и один - barista, которые готовы, т.е. `READY: ... 1/1`.

Отлично! Наши приложения теперь работают в облаке, но как же к ним подключиться?

## Сервисы (Services)

Service Kubernetes - это логическая абстракция над «приложениями» (какими бы они ни были) и их репликами.
Service - это точка входа для нашего микросервиса.
Service-ы действуют как балансировщики нагрузки и распределяют запросы по разным модулям.

Внутри кластеров Service работает через виртуальный IP-адрес кластера и через DNS по соответствующему имени - это позволяет нам легко подключаться к именам хостов, используя понятные имена, как, например, `barista`, если в кластере есть service `barista`.

Давайте посмотрим на определение Service `coffee-shop`:

```yaml
kind: Service
apiVersion: v1
metadata:
  name: coffee-shop
  labels:
    app: coffee-shop
spec:
  selector:
    app: coffee-shop
  ports:
    - port: 9080
      name: http
  type: NodePort
```

Определение service соодержит только имя, некоторые метаданные меток и информацию по роутингу траффика: все pod-ы, которые соответствуют данному selector-у.
Если вы посмотрите на нашу конфигурацию deployment, то увидите, что все модули определяют одинаковую метку `app`.
Это специальная метка, с помощью которой service-ы понимают, в какие pod-ы можно отправлять запросы.
Этот service будет подключаться ко всем pod-ам с меткой `app: coffee-shop` через порт` 9080`.
Само собой, service-ы подключаются только к рабочим pod-ам.

Идем дальше. Создадим конфигурацию YAML для service-ов `coffee-shop` и `barista` - в папке `deployment/`.
Вы можете создать новый файл - рядом с конфигурацией deployment, или поместить все ресурсы Kubernetes в один файл YAML, в котором ресурсы (то есть объекты YAML) разделены линией из трех штрихов (`---`).
Опять же, убедитесь, что имя, метка и определение селектора соответствуют или приложению coffee-shop, или приложению barista.

Создадим/обновим эти ресурсы в кластере, выполняя те же команды, что и раньше:

```
kubectl apply -f coffee-shop/deployments/
kubectl apply -f barista/deployments/
```

Это хороший пример концепции Infrastructure-as-Code: мы указываем желаемое состояние и позволяем Kubernetes _применить_ конфигурацию к нашему кластеру.

Наши папки теперь также содержат определения service-ов.
Можете проверить, правильно ли они были созданы:

```
kubectl get services
```

## Accessing our applications

Теперь попробум подключиться к нашему приложению coffee-shop "снаружи" границ кластера.

Если мы создали кластер в облаке IBM с использованием аккаунта Lite, то подключиться к нашему приложению мы сможем через IP-адрес узла и порт узла service.
Поэтому мы получаем общедоступный IP-адрес нашего кластера:

Now, we will connect to our coffee-shop application from outside the cluster.

If we have created a lite cluster we have to connect to our application via the IP address of the (only) node and the node port of the service.
Therefore, we retrieve the public IP address of our cluster:

```
ibmcloud ks workers --cluster cloud-native
ID         Public IP       Private IP      Machine Type   State    Status   Zone    Version   
kube-xxx   159.122.186.7   10.144.188.64   free           normal   Ready    mil01   1.10.12_1541   
```

And the node port of our coffee-shop application:

```
kubectl get service coffee-shop
NAME          TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
coffee-shop   NodePort   172.21.23.149   <none>        9080:30995/TCP   2m
```

With the example details, we can access our coffee-shop application using the URL `159.122.186.7:30995`, by combining the public IP address and the node port of the service:

```
curl <ip-address>:<node-port>/coffee-shop/resources/orders -i
```

NOTE: If you have created a standard cluster, you can use a Kubernetes ingress resources.
However, in this workshop, we'll focus on Istio networking and thus will demonstrate Istio gateway resources instead (part of the next section).


## Kubernetes Config Maps

We can define environment variables directly in Kubernetes deployment definitions, or configure them in so called config maps.
A config map is a Kubernetes resources that stores configuration properties in the cluster.
It can be mapped to files or, as in our example, environment variables.

We create the following Kubernetes YAML definition:

```yaml
kind: ConfigMap
apiVersion: v1
metadata:
  name: coffee-config
data:
  location: CEE
```

This defines the config map `coffee-config`, which contains the property `location` with the value `CEE`.

In order to make that property available to the running pods later on, we include the value in our Kubernetes deployment definition:

```yaml
# ...
containers:
- name: coffee-shop
  image: de.icr.io/cee-<your-name>-workshop/coffee-shop:1
  ports:
  - containerPort: 9080
  env:
  - name: location
    valueFrom:
      configMapKeyRef:
        name: coffee-config
        key: location
  livenessProbe:
# ...
```

The above example maps the config map values to environment variables in the pods.
As MicroProfile Config ships with a default config source for environment variables, this property will automatically be available to our application.
Thus, the injected value for the `location` will be the enum value `CEE`.

You can have a look at the coffee order locations under the resource for single coffee orders.
You retrieve the URL of a single coffee order from the response of all orders:

```
curl <ip-address>:<node-port>/coffee-shop/resources/orders
curl <ip-address>:<node-port>/coffee-shop/resources/orders/<order-uuid>
```


## 12 факторов

The https://12factor.net/[12 factors^] of modern software-as-a-service applications describe what aspects developers should take into account.
Have a look at the described factors and contemplate, where we've already covered these aspects by using Enterprise Java with cloud-native technologies.
With MicroProfile and its programming model, combined with Docker and Kubernetes, we can easily build 12-factor microservices.
We'll discuss the impact of the 12 factors together.

В этом разделе мы настроили среду Kubernetes, которая управляет нашими микросервисами.

Теперь давайте посмотрим как можно интегрировать Istio в следующем
[разделе](04-istio.md).
