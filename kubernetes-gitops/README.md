#GITOPS
Сделано все по заданию, реализован паплайн по поднятию и удалению кластера кубера в GCP и сборка бразов приложения...

Выкладываю что просили:

ссылка на репу: https://gitlab.com/FUCKir89/microservices-demo

```
kuber@ilyas-MacBook-Pro namespaces % kubectl logs helm-operator-6db458857f-7h7bm -n flux | grep hipster
ts=2020-10-05T20:08:13.96803647Z caller=helm.go:69 component=helm version=v3 info="Created a new Deployment called \"frontend-hipster\" in microservices-demo\n" targetNamespace=microservices-demo release=frontend
```

```
kuber@ilyas-MacBook-Pro frontend % kubectl get canary -A
NAMESPACE            NAME       STATUS        WEIGHT   LASTTRANSITIONTIME
microservices-demo   frontend   Succeded      0        2020-10-05T23:12:54Z
```

```
Events:
  Type     Reason  Age                From     Message
  ----     ------  ----               ----     -------
  Warning  Synced  21m                flagger  frontend-primary.microservices-demo not ready: waiting for rollout to finish: observed deployment generation less then desired generation
  Normal   Synced  20m (x2 over 21m)  flagger  all the metrics providers are available!
  Normal   Synced  20m                flagger  Initialization done! frontend.microservices-demo
  Normal   Synced  6m15s              flagger  New revision detected! Scaling up frontend.microservices-demo
  Normal   Synced  5m15s              flagger  Starting canary analysis for frontend.microservices-demo
  Normal   Synced  5m15s              flagger  Advance frontend.microservices-demo canary weight 10
  Normal   Synced  4m15s              flagger  Advance frontend.microservices-demo canary weight 20
  Normal   Synced  3m15s              flagger  Advance frontend.microservices-demo canary weight 30
  Normal   Synced  2m15s              flagger  Copying frontend.microservices-demo template spec to frontend-primary.microservices-demo
  Normal   Synced  75s                flagger  Routing all traffic to primary
  Normal   Synced  15s                flagger  Promotion completed! Scaling down frontend.microservices-demo
```
