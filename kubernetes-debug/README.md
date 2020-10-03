# Что сделано:

1. Поставили дебагер
```
brew install aylei/tap/kubectl-debug
```
2. Поставили агент
```
brew install aylei/tap/kubectl-debug
```
3. Запускаем наш под с приложением
```
brew install aylei/tap/kubectl-debug
```
4. Пробуем дебажить
```
kubectl-debug nginx --agentless=false --port-forward=true
strace -p 1
```
5. Магия не слуяилась, так как был косяк с правами... Попробуме у агента сменить версию на крайнюю и повторить... И... Все что хотели получилось

6. Делаем кластер в GKE для Iptables-tailer запускаем там приложение
```
kubectl apply -f kit/deploy/crd.yaml
kubectl apply -f kit/deploy/rbac.yaml
kubectl apply -f kit/deploy/operator.yaml
kubectl apply -f kit/deploy/cr.yaml
```
7. Создаем политику для кальки
```
kubectl apply -f netperf-calico-policy.yaml
```
8. Перекатим cr
```
kubectl delete -f deploy/cr.yaml 
kubectl apply -f deploy/cr.yaml
```
9. Проверим логи и запустим iptables-tailer
```
kubectl apply -f deploy/kit-serviceaccount.yaml
kubectl apply -f deploy/kit-clusterrole.yaml
kubectl apply -f deploy/kit-clusterrolebinding.yaml
kubectl apply -f deploy/iptables-tailer.yaml
```
10. Исправим iptables-tailer.yaml и netperf-calico-policy.yaml и наблюдаем что все ок

