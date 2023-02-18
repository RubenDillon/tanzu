GIT para poder desplegar TAP usando TMC 

Create cluster using TMC
    modify the default configuration allowing more space on the storage nodes
          Control Plane add space for etcd nodes.... mount the volume at /var/lib/etcd
          Workers add space for containerd ......... mount the volume at /var/lib/containerd

Enable Continuous Delivery and deploy Cert Manager and Contour 

Obtain where is running ENVOY
      in our example we assume that is running on the following IP Address: 10.220.8.22
      
Deploy Harbor using the values that are in this git
    Create the project tap on Harbor
    Desplegar el certificado de Harbor
            kubectl -n harbor get secret harbor-tls -o=jsonpath="{.data.ca\.crt}" | base64 -d
    Editar y copiar el certificado
            kubectl edit kubeadmconfigtemplate tap-vsphere7

Create a namespace called tap-install

Create a secret for Tanzu Net and export it to all namespaces
    Secret name: tap-registry
    Image registry URL: registry.tanzu.vmware.com
    Username: <tanzu-net-username>
    Password: <tanzu-net-password>

Verify secret is reconciled and is exported

Add Tanzu Application Platform package repository to the cluster
    Name: tanzu-tap-repository
    Repository URL: registry.tanzu.vmware.com/tanzu-application-platform/tap-packages:1.4.1

Verify tanzu-tap-repository package repository is successfully added

Install Tanzu Application Platform package
    The Tanzu Application Platform package is at bottom of list

Select version from drop-down eg 1.4.1 and then click "Install Package"

Name package and confirm version
    Installed package name: tap
    Package version: 1.0.1

Package install resources
    Leave defaults

Configure values
    Copy the light or full profile from TAP docs and configure the parameters as guided in that doc

Monitor the install

Verify all packages are successfully reconciled

Ta-da... cluster created and TAP installed all from TMC
  
.......................

  Basado en https://confluence.eng.vmware.com/display/CNA/TAP+-+How+to+install+TAP+using+TMC
