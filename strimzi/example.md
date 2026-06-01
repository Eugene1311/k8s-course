```shell
kubectl create namespace kafka
```
```shell
cd strimzi
kubectl apply -f ./strimzi.yml
```
```shell
kubectl get pod -n kafka --watch
kubectl logs deployment/strimzi-cluster-operator -n kafka -f
```
```shell
kubectl apply -f ./kafka-single-node.yaml
```
```shell
kubectl wait kafka/my-cluster --for=condition=Ready --timeout=300s -n kafka 
```
