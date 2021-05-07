---
date: 5 March 2021
---

# Addition second network interface

Sometimes we may not want a second network interface to create DNS nameserver entries inside `/etc/resolv.conf`. This can be achieved in OpenStack neutron by setting `dns_nameserver` to `0.0.0.0`.

    openstack subnet set  --dns-nameserver 0.0.0.0 Storage

Additionally, remove gateway IP (unfortunately there does not seem to be a way of doing this via OpenStack CLI at the moment):

    neutron subnet-update --no-gateway Storage

Weirdly, the following seems to work:

    openstack subnet set --gateway=None Storage

    
Calico

    kubectl edit ds/calico-node -n kube-system

        - name: IP_AUTODETECTION_METHOD
          value: cidr=10.0.0.0/24
