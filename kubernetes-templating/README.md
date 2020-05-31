## Kubernetes templating

### Что сделано:
 - Зарегались на GCP
 - Завели кластер кубера там GKE
 - Поставили ингрес контроллер на жинксе
 - Поставили сертманагер
 - Поставили хельмом chartmuseum
 - Заставили ингрес при деплое запрашивать выпуск нового серта
 - Поставили harbor
 - Разобрались с HELM
 - Вынесли сервисы frontend, redis, paymentservice, shippingservice и задеплоили их разными методами
 - Сделали кастомизацию сервиса recommendationservice

### Как запустить:
```bash
# Если у нас GKE рекомендуется создать clusterrolebinding для него (подставить свои данные)
kubectl create clusterrolebinding cluster-admin-binding     --clusterrole=cluster-admin     --user=$(gcloud config get-value core/account)

# Добавляем репу
helm repo add stable https://kubernetes-charts.storage.googleapis.com
helm update

# Ставим жинкс ингрес
kubectl create ns nginx-ingress
helm upgrade --install nginx-ingress stable/nginx-ingress --wait --namespace=nginx-ingress --version=1.11.1

# Ставим серт манагер
kubectl create ns cert-manager
kubectl label namespace cert-manager certmanager.k8s.io/disable-validation="true"
kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v0.15.1/cert-manager-legacy.crds.yaml
kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v0.15.1/cert-manager-legacy.yaml
kubectl apply -f kubernetes-templating/cert-manager/clusterissuer-letsencrypt-production.yaml

# Ставим chartmuseum

helm upgrade --install chartmuseum stable/chartmuseum --wait --namespace=chartmuseum --version=2.3.2 -f kubernetes-templating/chartmuseum/values.yaml

# Ставим harbor
kubectl create ns harbor
helm upgrade --install harbor harbor/harbor --wait --namespace=harbor --version=1.3.2 -f kubernetes-templating/harbor/values.yaml

# Запускаем хипстершоп через хельм
kubectl create ns hipster-shop
helm dep update kubernetes-templating/hipster-shop
helm upgrade --install hipster-shop kubernetes-templating/hipster-shop --namespace hipster-shop --set frontend.service.NodePort=31234

# Запускаем сервисы через кубконфиг
kubecfg update services.jsonnet --namespace hipster-shop

# Запустить кастомизированные сервис recommendationservice
kubectl create ns hipster-shop-prod
kubectl apply -k kubernetes-templating/kustomize/overrides/hipster-shop-prod
или
kubectl apply -k kubernetes-templating/kustomize/overrides/hipster-shop

```


### Как проверить:

 - helm version
 - helm repo list
 - helm list -n nginx-ingress
 - kubectl get svc -n nginx-ingress
 - helm list -n cert-manager
 - kubectl get certificates -A
 - helm list -n chartmuseum
 - kubectl get secrets -n chartmuseum
 - curl https://$(kubectl get ingress chartmuseum-chartmuseum -n chartmuseum -o jsonpath={.spec.rules[0].host})
 - helm list -n harbor
 - curl https://$(kubectl get ingress harbor-harbor-ingress -n harbor -o jsonpath={.spec.rules[0].host})
 - curl http://$(kubectl get ingress frontend -n hipster-shop -o jsonpath={.spec.rules[0].host})

### Задания со *

1. Для работы с chartmuseum как с репой надо спулить себе его ( helm pull stable/chartmuseum --version=2.3.2 ) и разархивировать ( tar -xvf chartmuseum-2.3.2.tgz ). Не забыть подкинуть свой values.yaml

2. Хельмфайл. Чтоб запустить его- выполнить (протестить после всего дз на чистом кластере):
```bash
helmfile -f kubernetes-templating/helmfile/helmfile.yaml -e <env> apply
```

3. Дополнительно через Kapitan или qbec не делал




