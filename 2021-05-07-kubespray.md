When Kubelet fails to start with this error `Failed to start ContainerManager invalid kernel flag: kernel/panic, expected value: 10, actual value: 5`:
    
    sudo su
    cat > /etc/sysctl.d/90-kubelet.conf << EOF
    vm.overcommit_memory=1
    kernel.panic=10
    kernel.panic_on_oops=1
    EOF
    sysctl -p /etc/sysctl.d/90-kubelet.conf
    systemctl restart kubelet
