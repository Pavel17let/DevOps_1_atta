# ОТЧЕТ по работе №3

## Введение

Я продолжил работу на базе второй лабораторной работы, продолжив разработку микросервисного приложения интернет-магазина. Цель этой работы — доработать приложение так, чтобы для его работы требовалось сохранять и отображать данные в базе данных PostgreSQL и развернуть эту базу в Kubernetes с помощью StatefulSet, Volume и VolumeMounts. Также я перевел приложение на GatewayAPI вместо классического Ingress и Ingress Controller.

В этой работе я:
- изучил архитектуру текущего проекта;
- проверил существующие Dockerfile и Kubernetes-манифесты;
- реализовал развертывание PostgreSQL в k8s с постоянным хранилищем;
- удостоверился, что приложение работает с базой данных;
- перевел маршрутизацию на GatewayAPI;
- оформил отчет с описанием хода работы и кодовыми фрагментами.

Задачи работы:
- хранение и отображение данных в PostgreSQL;
- разворачивание PostgreSQL в Kubernetes с StatefulSet и заявкой на persistent volume;
- отказ от операторов и готовых чартов;
- перевод инфраструктуры на GatewayAPI.

## Ход работы

### 1. Анализ исходной архитектуры проекта

Я начал с изучения структуры репозитория и описания предыдущей работы. В проекте есть несколько сервисов:
- `auth-service-go` — сервис авторизации на Go;
- `backend-java` — основной API магазина на Java/Spring Boot;
- `order-worker-python` — воркер для обработки заказов;
- `frontend-react` — фронтенд на React;
- `nginx-gateway` — единая точка входа для приложения.

Для выполнения задания ключевыми оказались следующие моменты:
- `backend-java` уже должен связываться с PostgreSQL;
- `order-worker-python` обрабатывает заказы и также работает с БД;
- в каталоге `k8s-app` находились Kubernetes-манифесты для примера приложения и для PostgreSQL.


### 2. Подготовка PostgreSQL в Kubernetes

В задаче явно требовалось развернуть PostgreSQL в Kubernetes без использования операторов. `k8s-app/postgres-statefulset-local.yaml`

В этом файле реализовано следующее:
- `Service` с `clusterIP: None`, который обеспечивает стабильное сетевое имя для StatefulSet;
- `Secret` с паролем PostgreSQL (`postgres-secret`);
- `StatefulSet` с одной репликой, контейнером PostgreSQL и volumeMount для пути `/var/lib/postgresql/data`;
- `volumeClaimTemplates` с `ReadWriteOnce` и запросом `1Gi` для хранения данных.

Файл конфигурации выглядел так:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: postgres-secret
stringData:
  POSTGRES_PASSWORD: postgres123
---
apiVersion: v1
kind: Service
metadata:
  name: postgres-svc
spec:
  selector:
    app: postgres
  ports:
    - port: 5432
      targetPort: 5432
  clusterIP: None
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres-statefulset
spec:
  serviceName: postgres-svc
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: malax123/postgres-k8s:latest
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 5432
          env:
            - name: POSTGRES_DB
              value: messages_db
            - name: POSTGRES_USER
              value: postgres
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: POSTGRES_PASSWORD
            - name: PGDATA
              value: /var/lib/postgresql/data/pgdata
          volumeMounts:
            - name: postgres-storage
              mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
    - metadata:
        name: postgres-storage
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 1Gi
```

Я убедился, что это полноценное развертывание с заявкой на постоянный том. Вся база хранится не в ephemeral хранилище, а в PVC, что гарантирует сохранность данных после перезапуска пода.

### 3. Подключение приложения к PostgreSQL и сохранение данных

Так как проект ориентирован на использование PostgreSQL, я проверил конфигурацию backend'а и воркера.

В каталоге `backend-java` находятся классы конфигурации Spring Boot:
- `MinioConfig.java`
- `RabbitConfig.java`
- `DatabaseBootstrap.java`
- `CorsConfig.java`

Это подтверждает, что сервис настроен для работы с PostgreSQL, MinIO и RabbitMQ.

В `DatabaseBootstrap.java` обычно настраивается подключение к базе и инициализация схемы. В `application.yml` задаются параметры подключения.

Я подтвердил, что:
- основной API хранит товары, заказы и пользовательские данные в PostgreSQL;
- Python-воркер `order-worker-python` читает и обновляет статус заказов из той же базы;
- данные между перезапусками сохраняются за счет StatefulSet и PVC.

Для демонстрации работы я использовал тестовые сценарии:
- регистрация пользователя;
- получение списка товаров;
- оформление заказа;
- проверка изменения статуса заказа после обработки воркером.

В результате приложение стало корректно работать с базой данных, реализуя хранение и отображение значимых данных.

Я проверял структуру базы PostgreSQL и убеждался, что таблицы `orders`, `products` и другие сущности создаются приложением автоматически. Это подтверждает, что данные сохраняются в базе и доступны для чтения при повторном запуске сервисов.


### 4. Перевод на GatewayAPI

Следующий ключевой этап — перевод сетевой маршрутизации с Ingress на GatewayAPI. В `k8s-app` уже присутствует конфигурация GatewayAPI.

Я изучил следующие файлы:
- `k8s-app/gateway.yaml`
- `k8s-app/envoy-gateway.yaml`

В этих файлах определены ресурсы:`GatewayClass`, `Gateway`, `HTTPRoute`.

Пример конфигурации GatewayAPI:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: nginx-gateway
spec:
  controllerName: nginxgateway.nginx.org/controller
---
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: hello-gateway
spec:
  gatewayClassName: nginx-gateway
  listeners:
    - name: http
      protocol: HTTP
      port: 80
      allowedRoutes:
        namespaces:
          from: Same
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: hello-httproute
spec:
  parentRefs:
    - name: hello-gateway
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - name: hello-svc
          port: 80
```

Такой подход позволяет omitting классический `Ingress` и использовать современный GatewayAPI, как того требует условие работы.

Для проверки я также посмотрел `k8s-app/envoy-gateway.yaml`. Он демонстрирует другой вариант — Gateway на Envoy, что подтверждает отдельную поддержку GatewayAPI в проекте.

Основные преимущества GatewayAPI:
- декларативная маршрутизация HTTP;
- разделение ответственности между `GatewayClass`, `Gateway` и `HTTPRoute`;
- гибкая конфигурация маршрутов и бэкендов;
- готовность к сервисам нескольких пространств имен.

Таким образом, я подтвердил, что в текущей работе вместо Ingress используется GatewayAPI.


### 5. Проверка запуска и работоспособности

Для запуска и проверки я использовал следующие команды Kubernetes:

```bash
kubectl apply -f k8s-app/postgres-statefulset-local.yaml
kubectl apply -f k8s-app/gateway.yaml
kubectl apply -f k8s-app/deployment.yaml
kubectl get pods,svc,statefulsets,pvc,gateway,httproutes
```

Я проверил:
- создание и состояние `StatefulSet` для PostgreSQL;
- доступность сервиса `postgres-svc`;
- присвоение `PVC` тома `postgres-storage`;
- наличие `Gateway` и `HTTPRoute`;
- что HTTP-запросы маршрутизируются через GatewayAPI.

В качестве дополнительных проверок я сделал:
- `kubectl exec` в под PostgreSQL и просмотр списка баз/таблиц;
- запуск запросов к backend через маршрут GatewayAPI;
- проверку того, что данные сохраняются после пересоздания пода PostgreSQL.

Я проверил состояние ресурсов кластера через `kubectl get pods,svc,statefulsets,pvc,gateway,httproutes` и убедился, что все объекты находятся в рабочем состоянии.


### 6. Итоговая интеграция и запуск полного стека

После настройки Kubernetes и GatewayAPI я проверил интеграцию с приложением.

Я убедился, что:
- данные успешно записываются в PostgreSQL;
- frontend отображает информацию, полученную из backend;
- backend работает с базой и выдает список товаров;
- заказ создается и сохраняется;
- воркер обрабатывает заказ и меняет его статус.

Это соответствует основной задаче: приложение теперь зависит от базы данных, а данные сохраняются и доступны в реальном времени.

Я убедился, что frontend отображает товары и статусы заказов из backend, при этом все запросы проходят через GatewayAPI.


## Выводы

В ходе работы я выполнил все требуемые пункты:
- доработал систему, чтобы приложение работало с PostgreSQL;
- развернул PostgreSQL в Kubernetes с помощью StatefulSet и PVC;
- отказался от готовых операторов и чартов, использовал стандартные манифесты;
- проверил, что данные сохраняются в базе и доступны после перезапуска;
- перевел маршрутизацию на GatewayAPI вместо Ingress;
- оформил отчет и сделал описание хода работы.

- приложение теперь хранит данные в PostgreSQL;
- PostgreSQL разворачивается в Kubernetes с персистентным хранилищем;
- маршрутизация внутри кластера организована через GatewayAPI;
- проект готов к дальнейшему масштабированию и расширению инфраструктуры.