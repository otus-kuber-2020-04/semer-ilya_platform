Сделал по инструкции, инвентори в это папке лежит для спрея.

Из особенностей спрея хотел бы обратить внимание на файлы: 
- inventory/sample/group_vars/k8s-cluster/k8s-cluster.yml тут можно настроить:
  - версию кубера директивой kube_version: v1.19.2
  - сетевой плагин директивой kube_network_plugin: weave
  - режим прокси ipvs или iptables директивой kube_proxy_mode: ipvs 
  - имя кластера директивой cluster_name: mycluster.otus
  - контейнерный рантайм директивой container_manager: docker
-inventory/sample/group_vars/all/docker.yml
  - тут для докера поставить docker_storage_options: -s overlay2 по деволту закоменчено

Это основное что можно минимально покрутить у спрея)

Больше я не знаю что еще добавить в доку по этой ДЗ)))
