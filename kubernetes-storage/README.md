# Что сделано:

1. Поставили снапшотер:
```
#Задали весию сапшотера
$ SNAPSHOTTER_VERSION=v2.1.0

# Поставили CRD
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/${SNAPSHOTTER_VERSION}/config/crd/snapshot.storage.k8s.io_volumesnapshotclasses.yaml
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/${SNAPSHOTTER_VERSION}/config/crd/snapshot.storage.k8s.io_volumesnapshotcontents.yaml
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/${SNAPSHOTTER_VERSION}/config/crd/snapshot.storage.k8s.io_volumesnapshots.yaml

# Поставили RBAC и контороллер
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/${SNAPSHOTTER_VERSION}/deploy/kubernetes/snapshot-controller/rbac-snapshot-controller.yaml
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/${SNAPSHOTTER_VERSION}/deploy/kubernetes/snapshot-controller/setup-snapshot-controller.yaml
``` 


2. Поставили CSI Host Path Driver (репу в гитигнор добавил, чтоб не мешалась)
```
git clone https://github.com/kubernetes-csi/csi-driver-host-path.git
./csi-driver-host-path/deploy/kubernetes-1.17/deploy.sh
```

3. Пробуем работу:
```
cd ./kubernetes-storage/hw/

#Создаем StorageClass
$ kubectl apply -f sc.yaml

#Создаем PVC
$ kubectl apply -f pvc.yaml

#Создаем pod
$ kubectl apply -f pod.yaml

#Запишем тестовые данные
$ kubectl exec -ti storage-pod -- /bin/sh -c 'echo "Hello Otus!" >> /data/test-file'

#Посомтрим что фалик записался
$ kubectl exec -ti storage-pod -- /bin/sh -c 'cat /data/test-file'

#Сделаем снапшот
$ kubectl apply -f snapshot.yaml

#Полюбуемся на наш снапшот
$ kubectl get volumesnapshot -oyaml

#Удалим под и хранилку
$ kubectl delete pod storage-pod 
$ kubectl delete pvc storage-pvc

#Восстановим хранилку
$ kubectl apply -f restore.yaml

#Восстановим под
$ kubectl apply -f pod.yaml

#О, боги! Наш файл на месте! Магия случилсь!
$ kubectl exec -ti storage-pod -- /bin/sh -c 'cat /data/test-file'


