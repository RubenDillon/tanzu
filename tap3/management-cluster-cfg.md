### Prepare SSH key

https://docs.vmware.com/en/VMware-Tanzu-Kubernetes-Grid/1.5/vmware-tanzu-kubernetes-grid-15/GUID-mgmt-clusters-azure.html 

    sudo ssh-keygen -t rsa -b 4096 -C “rdillon@vmware.com” 
    sudo eval $(ssh-agent)
    sudo ssh-add ~/.ssh/id_rsa
    sudo cat ~/.ssh/id_rsa.pub

copy key txt (all) 

### Validate the Base image is accepted (Azure)

az vm image terms show --publisher vmware-inc --offer tkg-capi --plan k8s-1dot22dot5-ubuntu-2004

The output should contain "accepted": true
