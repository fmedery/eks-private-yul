# Introduction
* Vérification via un test simple que le ELB est crée un subnet privé
* Un `security group` est crée pour le ELB et accessible pour 0.0.0.0 (voir l'annotation dans 01_service.yaml)

# Test
```sh
cd /tmp/eks-private-yul/tests/
```

```sh
# Vérification des services
kubectl --kubeconfig $HOME/.kube/$EKS_CLUSTER_NAME.conf get svc
```

```sh
# Création du workload
kubectl --kubeconfig $HOME/.kube/$EKS_CLUSTER_NAME.conf create -f 00_pods.yaml
kubectl --kubeconfig $HOME/.kube/$EKS_CLUSTER_NAME.conf create -f 01_service.yaml
```

```sh
# vérification de la création du NLB
aws --region=${REGION} elb describe-load-balancers
```

```sh
# clean up
kubectl --kubeconfig $HOME/.kube/$EKS_CLUSTER_NAME.conf delete svc/nginx
kubectl --kubeconfig $HOME/.kube/$EKS_CLUSTER_NAME.conf delete pods/nginx
```