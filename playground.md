 

```sh
helm upgrade -i yhp --set global.storage_classname="" --set tags.aurora=false --set tags.icefjord=false --set tags.minio=false --set tags.monitor=false --set tags.postgres=false --set tags.stonewave=true --set tags.workflow=false helm-internal/yhp-chart --version 1.1.0 --create-namespace -n yh
```



export POD_NAME=$(kubectl get pods --namespace yh -l "app=prometheus,component=server" -o jsonpath="{.items[0].metadata.name}")
  kubectl --namespace yh port-forward $POD_NAME 9090

```sh
kubectl run client --image python:3.9.1-buster --namespace yh-longevity --command -- sleep infinity
```





```sh
sw-cli dispatch-events --token 'a940a882-7e6b-4908-b5b2-b233ea51f2ce' --event-set-name main --events-file longevity_static_access_2000000.log --host stonewave --nile-host parana --nile-port 9090 --datatype nginx__access_log
```



