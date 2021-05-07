---
date: 12 February 2021
---

# Ceph-CSI + Manila-CSI - Magnum

## Step 1: Ceph CSI

helm repo add ceph-csi https://ceph.github.io/csi-charts
helm install --namespace "ceph-csi-cephfs" "ceph-csi-cephfs" ceph-csi/ceph-csi-cephfs --create-namespace

## Step 2: Manila CSI

    helm repo add cpo https://kubernetes.github.io/cloud-provider-openstack
    helm install manila-csi cpo/openstack-manila-csi -f values.yaml
    helm install manila-csi cpo/openstack-manila-csi -f values.yaml
    cat values.yaml

    
    # Enabled Manila share protocols
    shareProtocols:
      - protocolSelector: CEPHFS
        fwdNodePluginEndpoint:
          dir: /var/lib/kubelet/cephfs.csi.ceph.com
          sockFile: csi.sock

https://github.com/kubernetes/cloud-provider-openstack/blob/master/docs/manila-csi-plugin/using-manila-csi-plugin.md#controller-service-volume-parameters

3. Add Storage class with magnum secrets, manila share type and cephfs kernel module
cat <<END | kubectl apply -f -
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: csi-manila-cephfs
provisioner: cephfs.manila.csi.openstack.org
parameters:
  # Manila share type
  type: default_share_type 
  cephfs-mounter: kernel
  csi.storage.k8s.io/provisioner-secret-name: os-trustee
  csi.storage.k8s.io/provisioner-secret-namespace: kube-system
  csi.storage.k8s.io/node-stage-secret-name: os-trustee
  csi.storage.k8s.io/node-stage-secret-namespace: kube-system
  csi.storage.k8s.io/node-publish-secret-name: os-trustee
  csi.storage.k8s.io/node-publish-secret-namespace: kube-system
END
4. Add PVC and POD
cat <<END | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: new-cephfs-share-pvc
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
  storageClassName: csi-manila-cephfs
END
kubectl apply -f https://raw.githubusercontent.com/brtknr/cloud-provider-openstack/csimanila/examples/manila-csi-plugin/cephfs/dynamic-provisioning/pod.yaml
5. Look into logs for troubleshooting
kubectl logs ceph-csi-cephfs-nodeplugin-slww5 -n ceph-csi-cephfs -c csi-cephfsplugin
I0209 16:17:25.887144       1 nodeserver.go:153] ID: 45 Req-ID: 9d6eb9ed-dd71-40af-9163-22b923228462 cephfs: mounting volume 9d6eb9ed-dd71-40af-9163-22b923228462 with Ceph FUSE driver
kubectl logs manila-csi-openstack-manila-csi-nodeplugin-7sxkk -c cephfs-registrar
kubectl logs manila-csi-openstack-manila-csi-nodeplugin-fv2xh -c cephfs-registrar
kubectl logs manila-csi-openstack-manila-csi-nodeplugin-gw9zc -c cephfs-registrar
kubectl logs manila-csi-openstack-manila-csi-controllerplugin-0 -c cephfs-nodeplugin
kubectl logs ds/manila-csi-openstack-manila-csi-nodeplugin -c cephfs-nodeplugin
kubectl logs statefulset/manila-csi-openstack-manila-csi-controllerplugin -c cephfs-nodeplugin