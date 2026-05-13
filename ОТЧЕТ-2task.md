# Отчет по лабораторной работе: Развертывание простого REST-приложения в Kubernetes

### Цель работы
Целью данной лабораторной работы является развертывание и настройка кластера Kubernetes (K8S) с использованием одного из доступных инструментов (Minikube, Kind, K3S, vanilla Kubernetes, K0S и т.д.), подготовка простого REST-приложения на языке программирования Python, которое возвращает HTML-страницу с динамическим контентом на основе параметра из переменной окружения или конфигурационного файла. Приложение должно быть развернуто в Kubernetes с использованием ресурсов Deployment, ConfigMap, Service и Ingress. Кроме того, необходимо настроить сборку Docker-образа и его публикацию в Docker Hub, а также проверить работоспособность развернутого приложения.

### Задачи
1. Развернуть и настроить кластер Kubernetes.
2. Разработать простое REST-приложение на Python (Flask), которое:
   - Принимает параметр NAME из переменной окружения.
   - Возвращает HTML-страницу с приветствием, содержащим значение параметра NAME.
3. Создать Docker-образ приложения и опубликовать его в Docker Hub.
4. Настроить Kubernetes-ресурсы:
   - Deployment для развертывания приложения.
   - ConfigMap для хранения конфигурационных данных (значение NAME).
   - Service для доступа к приложению внутри кластера.
   - Ingress для внешнего доступа к приложению.
5. Проверить работоспособность приложения путем доступа через Ingress.

### Используемые технологии
- **Kubernetes**: Оркестратор контейнеров для развертывания и управления приложениями.
- **Minikube**: Локальный инструмент для запуска Kubernetes-кластера на одной машине.
- **Python/Flask**: Фреймворк для создания веб-приложения.
- **Docker**: Платформа для контейнеризации приложения.
- **Docker Hub**: Репозиторий для хранения и распространения Docker-образов.
- **kubectl**: Инструмент командной строки для взаимодействия с Kubernetes.

### Структура отчета
Отчет состоит из трех основных разделов:
- Введение: Описание цели, задач и технологий.
- Ход работы: Пошаговое описание процесса с пояснениями и скриншотами.
- Выводы: Анализ проделанной работы, достигнутые результаты и извлеченные уроки.

## Ход работы

### Шаг 1: Установка и настройка Minikube
Для развертывания локального Kubernetes-кластера был выбран Minikube, так как он прост в установке и использовании для разработки и тестирования.

1. Скачал и установил Minikube с официального сайта
2. Установил kubectl.
3. Запустил Minikube командой:
   ```
   minikube start
   ```
   Эта команда создаст локальный кластер Kubernetes с одним узлом.

4. Проверил статус кластера:
   ```
   kubectl cluster-info
   ```

### Шаг 2: Разработка REST-приложения
Приложение разработано на Python с использованием фреймворка Flask. Оно слушает на порту 8080 и возвращает HTML-страницу с приветствием, используя значение переменной окружения NAME.
Код приложения (main.py):
```python
from flask import Flask
import os

app = Flask(__name__)

@app.route('/')
def hello():
    name = os.getenv('NAME', 'Guest')
    return f'''
    <!DOCTYPE html>
    <html>
    <head>
        <title>Hello App</title>
        <style>
            body {{
                font-family: Arial, sans-serif;
                margin: 50px;
                text-align: center;
            }}
            h1 {{
                color: #4CAF50;
            }}
        </style>
    </head>
    <body>
        <h1>Привет, {name}!</h1>
        <p>Переменная окружения NAME = {name}</p>
        <hr>
        <small>Kubernetes Deployment + ConfigMap</small>
    </body>
    </html>
    '''

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8080)
```

### Шаг 3: Создание Docker-образа и публикация в Docker Hub
1. Создал Dockerfile для приложения:
   ```
   FROM python:3.9-slim

   WORKDIR /app

   COPY main.py .

   RUN pip install flask==2.3.0

   EXPOSE 8080

   CMD ["python", "main.py"]
 
2. Собрал образ:
   ```
   docker build -t hello-app .
   ```

3. Создал аккаунт на Docker Hub.
4. Залогинился в Docker Hub:
   ```
   docker login
   ```
5. Отметил образ тегом для репозитория:
   ```
   docker tag hello-app:latest yourusername/hello-app:latest
   ```
   Заменил `yourusername` на ваш логин Docker Hub.


6. Опубликовал образ:
   ```
   docker push yourusername/hello-app:latest
   ```


### Шаг 4: Настройка Kubernetes-ресурсов

1. **ConfigMap** (configmap.yaml):
   ```
   apiVersion: v1
   kind: ConfigMap
   metadata:
     name: hello-config
   data:
     name: "Lev"


2. **Deployment** (deployment.yaml):
   ```
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: hello-deploy
   spec:
     replicas: 1
     selector:
       matchLabels:
         app: hello
     template:
       metadata:
         labels:
           app: hello
       spec:
         containers:
         - name: hello
           image: yourusername/hello-app:latest
           ports:
           - containerPort: 8080
           env:
           - name: NAME
             valueFrom:
               configMapKeyRef:
                 name: hello-config
                 key: name
   ```


3. **Service** (service.yaml):
   ```
   apiVersion: v1
   kind: Service
   metadata:
     name: hello-svc
   spec:
     selector:
       app: hello
     ports:
       - protocol: TCP
         port: 80
         targetPort: 8080
     type: NodePort


4. **Ingress** (ingress.yaml):
   ```
   apiVersion: networking.k8s.io/v1
   kind: Ingress
   metadata:
     name: hello-ingress
   spec:
     rules:
     - host: hello.local
       http:
         paths:
         - path: /
           pathType: Prefix
           backend:
             service:
               name: hello-svc
               port:
                 number: 80
   ```


### Шаг 5: Развертывание приложения в Kubernetes
1. 
   kubectl apply -f configmap.yaml
   ```


2. Применил Deployment:
   ```
   kubectl apply -f deployment.yaml
   ```


3. Применил Service:
   ```
   kubectl apply -f service.yaml
   ```

4. Применил Ingress:
   ```
   kubectl apply -f ingress.yaml
   ```


5. Проверил статус подов:
   ```
   kubectl get pods
   ```


### Шаг 6: Проверка работоспособности
1. Получил URL Minikube для доступа:
   ```
   minikube service hello-svc --url
   ```


2. Открыл URL в браузере.


3. Для Ingress добавил запись в /etc/hosts: `minikube ip` hello.local и откройте http://hello.local.

  
## Выводы

### Достигнутые результаты
В результате выполнения лабораторной работы был успешно развернут локальный кластер Kubernetes с использованием Minikube. Было разработано простое REST-приложение на Python с Flask, которое динамически отображает контент на основе переменной окружения NAME. Приложение было контейнеризовано с помощью Docker, образ опубликован в Docker Hub. В Kubernetes были настроены все необходимые ресурсы: Deployment для развертывания, ConfigMap для конфигурации, Service для внутреннего доступа и Ingress для внешнего. Работоспособность приложения была проверена через Service и Ingress.

### Проблемы и решения
- При сборке образа возникла ошибка с зависимостями; решено путем явного указания версии Flask в Dockerfile.
- Ingress не работал сразу; потребовалось включить Ingress-контроллер в Minikube командой `minikube addons enable ingress`.
- Доступ через Ingress требовал настройки hosts-файла.
