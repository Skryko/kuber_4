Ниже готовый README.md. Скопируй его полностью в файл README.md. В местах ВСТАВИТЬ СКРИНШОТ просто вставишь свои изображения.

Домашнее задание к занятию «Хранение в K8s»

Цель работы

В рамках домашнего задания была выполнена работа с хранилищами в Kubernetes:

* настроен обмен файлами между контейнерами внутри одного Pod с помощью emptyDir;
* создан PersistentVolume и PersistentVolumeClaim для подключения локального хранилища;
* создан StorageClass и подключение хранилища через PVC;
* проверено поведение PV после удаления Deployment и PVC.

Работа выполнялась в тестовом Kubernetes-кластере Minikube.

⸻

Задание 1. Volume: обмен данными между контейнерами в Pod

Описание

Был создан Deployment data-exchange, состоящий из двух контейнеров:

* busybox — каждые 5 секунд записывает текущую дату и сообщение в файл;
* multitool — читает этот же файл с помощью команды tail -f.

Для обмена данными между контейнерами используется общий volume типа emptyDir.

emptyDir создаётся при запуске Pod и существует только пока существует сам Pod. Если Pod будет удалён, данные из emptyDir также будут удалены.

Манифест

Файл: containers-data-exchange.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: data-exchange
spec:
  replicas: 1
  selector:
    matchLabels:
      app: data-exchange
  template:
    metadata:
      labels:
        app: data-exchange
    spec:
      containers:
        - name: busybox
          image: busybox
          command: ["/bin/sh", "-c"]
          args:
            - while true; do echo "$(date) - message from busybox" >> /data/shared-file.txt; sleep 5; done
          volumeMounts:
            - name: shared-volume
              mountPath: /data
        - name: multitool
          image: wbitt/network-multitool
          command: ["/bin/sh", "-c"]
          args:
            - tail -f /data/shared-file.txt
          volumeMounts:
            - name: shared-volume
              mountPath: /data
      volumes:
        - name: shared-volume
          emptyDir: {}

Применение манифеста

kubectl apply -f containers-data-exchange.yaml

Проверка Pod

kubectl get pods

ВСТАВИТЬ СКРИНШОТ вывода kubectl get pods

Описание Pod

kubectl describe pods -l app=data-exchange

ВСТАВИТЬ СКРИНШОТ вывода kubectl describe pods -l app=data-exchange

Проверка чтения файла контейнером multitool

kubectl logs -f deployment/data-exchange -c multitool

Результат:

Sun Jun 28 00:20:24 UTC 2026 - message from busybox
Sun Jun 28 00:20:29 UTC 2026 - message from busybox
Sun Jun 28 00:20:34 UTC 2026 - message from busybox
Sun Jun 28 00:20:39 UTC 2026 - message from busybox

ВСТАВИТЬ СКРИНШОТ вывода kubectl logs -f deployment/data-exchange -c multitool

Вывод по заданию 1

Контейнер busybox успешно записывает данные в файл /data/shared-file.txt, а контейнер multitool читает этот же файл. Это подтверждает, что оба контейнера используют общий volume emptyDir, смонтированный в директорию /data.

⸻

Задание 2. PV и PVC

Описание

Во втором задании был создан локальный PersistentVolume, который использует директорию на ноде Minikube.

Также был создан PersistentVolumeClaim, через который Deployment подключает это хранилище внутрь контейнеров.

В отличие от emptyDir, данные в таком хранилище не зависят напрямую от жизненного цикла Pod.

Подготовка директории на ноде

minikube ssh "sudo mkdir -p /mnt/data-k8s/pv && sudo chmod 777 /mnt/data-k8s/pv"

Проверка директории:

minikube ssh "ls -ld /mnt/data-k8s/pv"

ВСТАВИТЬ СКРИНШОТ проверки директории /mnt/data-k8s/pv

Манифест

Файл: pv-pvc.yaml

apiVersion: v1
kind: PersistentVolume
metadata:
  name: data-exchange-pv
spec:
  capacity:
    storage: 1Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /mnt/data-k8s/pv
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-exchange-pvc
spec:
  storageClassName: ""
  volumeName: data-exchange-pv
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: data-exchange-pvc
spec:
  replicas: 1
  selector:
    matchLabels:
      app: data-exchange-pvc
  template:
    metadata:
      labels:
        app: data-exchange-pvc
    spec:
      containers:
        - name: busybox
          image: busybox
          command: ["/bin/sh", "-c"]
          args:
            - while true; do echo "$(date) - message from busybox to PV" >> /data/pv-file.txt; sleep 5; done
          volumeMounts:
            - name: pv-storage
              mountPath: /data
        - name: multitool
          image: wbitt/network-multitool
          command: ["/bin/sh", "-c"]
          args:
            - tail -f /data/pv-file.txt
          volumeMounts:
            - name: pv-storage
              mountPath: /data
      volumes:
        - name: pv-storage
          persistentVolumeClaim:
            claimName: data-exchange-pvc

Применение манифеста

kubectl apply -f pv-pvc.yaml

Проверка PV

kubectl get pv

Результат:

NAME               CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                       STORAGECLASS
data-exchange-pv   1Gi        RWO            Retain           Bound    default/data-exchange-pvc

ВСТАВИТЬ СКРИНШОТ вывода kubectl get pv

Проверка PVC

kubectl get pvc

Результат:

NAME                STATUS   VOLUME             CAPACITY   ACCESS MODES   STORAGECLASS
data-exchange-pvc   Bound    data-exchange-pv   1Gi        RWO

ВСТАВИТЬ СКРИНШОТ вывода kubectl get pvc

Проверка Pod

kubectl get pods -l app=data-exchange-pvc

Результат:

NAME                                 READY   STATUS    RESTARTS   AGE
data-exchange-pvc-56fbbbf46d-499xl   2/2     Running   0          45s

ВСТАВИТЬ СКРИНШОТ вывода kubectl get pods -l app=data-exchange-pvc

Проверка чтения файла контейнером multitool

kubectl logs -f deployment/data-exchange-pvc -c multitool

Результат:

Sun Jun 28 00:27:41 UTC 2026 - message from busybox to PV
Sun Jun 28 00:27:46 UTC 2026 - message from busybox to PV
Sun Jun 28 00:27:51 UTC 2026 - message from busybox to PV

ВСТАВИТЬ СКРИНШОТ вывода kubectl logs -f deployment/data-exchange-pvc -c multitool

Проверка файла на локальной ноде

minikube ssh "ls -l /mnt/data-k8s/pv && cat /mnt/data-k8s/pv/pv-file.txt"

ВСТАВИТЬ СКРИНШОТ проверки файла на ноде

Удаление Deployment и PVC

kubectl delete deployment data-exchange-pvc
kubectl delete pvc data-exchange-pvc

Проверка PV после удаления PVC

kubectl get pv

Результат:

NAME               CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS     CLAIM                       STORAGECLASS
data-exchange-pv   1Gi        RWO            Retain           Released   default/data-exchange-pvc

ВСТАВИТЬ СКРИНШОТ вывода kubectl get pv после удаления PVC

Подробное описание PV:

kubectl describe pv data-exchange-pv

Результат:

Name:            data-exchange-pv
StorageClass:
Status:          Released
Claim:           default/data-exchange-pvc
Reclaim Policy:  Retain
Access Modes:    RWO
VolumeMode:      Filesystem
Capacity:        1Gi
Source:
    Type:          HostPath
    Path:          /mnt/data-k8s/pv

ВСТАВИТЬ СКРИНШОТ вывода kubectl describe pv data-exchange-pv

Объяснение поведения PV после удаления PVC

После удаления PVC объект PV не удалился, а перешёл в состояние Released.

Это произошло из-за параметра:

persistentVolumeReclaimPolicy: Retain

Политика Retain означает, что Kubernetes не удаляет хранилище автоматически после удаления PVC. PV освобождается от заявки, но сам объект PV и данные остаются.

Проверка файла после удаления Deployment и PVC

minikube ssh "ls -l /mnt/data-k8s/pv && cat /mnt/data-k8s/pv/pv-file.txt"

Результат показывает, что файл pv-file.txt сохранился на локальном диске ноды.

ВСТАВИТЬ СКРИНШОТ проверки файла после удаления Deployment и PVC

Удаление PV

kubectl delete pv data-exchange-pv

Проверка:

kubectl get pv

ВСТАВИТЬ СКРИНШОТ вывода kubectl get pv после удаления PV

Проверка файла после удаления PV

minikube ssh "ls -l /mnt/data-k8s/pv && cat /mnt/data-k8s/pv/pv-file.txt"

ВСТАВИТЬ СКРИНШОТ проверки файла после удаления PV

Объяснение поведения файла после удаления PV

После удаления PV файл на локальном диске ноды остался.

Это произошло потому, что в задании использовался hostPath. Такой PV ссылается на обычную директорию файловой системы ноды:

/mnt/data-k8s/pv

Когда удаляется объект PersistentVolume, Kubernetes удаляет только ресурс из своего API, но не удаляет физические файлы из директории hostPath.

⸻

Задание 3. StorageClass

Описание

В третьем задании был создан StorageClass, PVC и Deployment, использующий PVC.

StorageClass был создан с параметром:

provisioner: kubernetes.io/no-provisioner

Это означает, что Kubernetes не создаёт PV автоматически. Поэтому PersistentVolume был создан вручную и связан с PVC через storageClassName.

Подготовка директории на ноде

minikube ssh "sudo mkdir -p /mnt/data-k8s/sc && sudo chmod 777 /mnt/data-k8s/sc"

Проверка директории:

minikube ssh "ls -ld /mnt/data-k8s/sc"

ВСТАВИТЬ СКРИНШОТ проверки директории /mnt/data-k8s/sc

Манифест

Файл: sc.yaml

apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-storage-manual
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: data-exchange-sc-pv
spec:
  capacity:
    storage: 1Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: local-storage-manual
  local:
    path: /mnt/data-k8s/sc
  nodeAffinity:
    required:
      nodeSelectorTerms:
        - matchExpressions:
            - key: kubernetes.io/hostname
              operator: In
              values:
                - minikube
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-exchange-sc-pvc
spec:
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi
  storageClassName: local-storage-manual
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: data-exchange-sc
spec:
  replicas: 1
  selector:
    matchLabels:
      app: data-exchange-sc
  template:
    metadata:
      labels:
        app: data-exchange-sc
    spec:
      containers:
        - name: busybox
          image: busybox
          command: ["/bin/sh", "-c"]
          args:
            - while true; do echo "$(date) - message from busybox to StorageClass PV" >> /data/sc-file.txt; sleep 5; done
          volumeMounts:
            - name: sc-storage
              mountPath: /data
        - name: multitool
          image: wbitt/network-multitool
          command: ["/bin/sh", "-c"]
          args:
            - tail -f /data/sc-file.txt
          volumeMounts:
            - name: sc-storage
              mountPath: /data
      volumes:
        - name: sc-storage
          persistentVolumeClaim:
            claimName: data-exchange-sc-pvc

Применение манифеста

kubectl apply -f sc.yaml

Проверка StorageClass

kubectl get sc

ВСТАВИТЬ СКРИНШОТ вывода kubectl get sc

Проверка PV

kubectl get pv

ВСТАВИТЬ СКРИНШОТ вывода kubectl get pv

Проверка PVC

kubectl get pvc

ВСТАВИТЬ СКРИНШОТ вывода kubectl get pvc

Проверка Pod

kubectl get pods -l app=data-exchange-sc

ВСТАВИТЬ СКРИНШОТ вывода kubectl get pods -l app=data-exchange-sc

Проверка чтения файла контейнером multitool

kubectl logs -f deployment/data-exchange-sc -c multitool

ВСТАВИТЬ СКРИНШОТ вывода kubectl logs -f deployment/data-exchange-sc -c multitool

Проверка файла на локальной ноде

minikube ssh "cat /mnt/data-k8s/sc/sc-file.txt"

ВСТАВИТЬ СКРИНШОТ проверки файла /mnt/data-k8s/sc/sc-file.txt

Вывод по заданию 3

StorageClass local-storage-manual был успешно создан. Так как используется provisioner: kubernetes.io/no-provisioner, автоматического создания PV не происходит. Поэтому PV был создан вручную.

PVC data-exchange-sc-pvc был связан с PV data-exchange-sc-pv, после чего Deployment data-exchange-sc смог использовать это хранилище.

Контейнер busybox записывает данные в файл /data/sc-file.txt, а контейнер multitool читает этот файл. Это подтверждает, что хранилище успешно подключено через PVC, созданный на основе StorageClass.

⸻

Общий вывод

В ходе выполнения домашнего задания были проверены разные способы работы с хранилищами в Kubernetes.

В первом задании использовался emptyDir. Это временное хранилище внутри Pod, которое подходит для обмена файлами между контейнерами одного Pod.

Во втором задании использовались PersistentVolume и PersistentVolumeClaim. Данные сохранялись в директории на локальной ноде Minikube и не удалялись после удаления Deployment и PVC благодаря политике Retain.

В третьем задании был создан StorageClass, а PVC был связан с PV через storageClassName. Это показало принцип работы StorageClass и подключения хранилища к Pod через PVC.

Таким образом, были отработаны базовые механизмы хранения данных в Kubernetes: emptyDir, PV, PVC и StorageClass.
