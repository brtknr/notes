Running legacy Kubernetes cluster on cumulus

    openstack coe cluster create --cluster-template kubernetes-1.17.2-20200205 legacy --keypair cumulus --labels heat_container_agent_tag=ussuri-stable-1,etcd_tag=v3.3.17 --merge-labels

But I am still seeing this with fedora atomic driver:

    error: error reading /etc/kubernetes/ca-bundle.crt: no such file or directory

I guess updating to fedora-coreos would solve the issue?