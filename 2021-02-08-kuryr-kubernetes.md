---
date: 8 February 2021
---

I tried the following but it didn't work

    git clone git@github.com:openstack/kuryr-kubernetes
    cd kuryr-kubernetes
    ./tools/generate_k8s_resource_definitions.sh
    sed -i 's/imagePullPolicy: Never/imagePullPolicy: Always/g' *.yml

    kubectl apply -f config_map.yml -n kube-system
    kubectl apply -f controller_service_account.yml -n kube-system
    kubectl apply -f certificates_secret.yml -n kube-system
    kubectl apply -f cni_service_account.yml -n kube-system
    kubectl apply -f controller_deployment.yml -n kube-system
    kubectl apply -f cni_ds.yml -n kube-system

    kubectl delete -f config_map.yml -n kube-system
    kubectl delete -f controller_service_account.yml -n kube-system
    kubectl delete -f certificates_secret.yml -n kube-system
    kubectl delete -f cni_service_account.yml -n kube-system
    kubectl delete -f controller_deployment.yml -n kube-system
    kubectl delete -f cni_ds.yml -n kube-system