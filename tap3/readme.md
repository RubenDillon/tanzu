# Deploy Tanzu Application Platform v1.5 using Azure with Harbor (using FREE public certificates)

## Requirements
```
    1. An Azure subscription to deploy TKG
    2. A management cluster deployed on Azure
    3. An AWS subscription for route53 DNS service (we create solateam.be domain)
    4. Tanzu Mission Control
    5. To deploy 
        - TAP 1.4.1 we need Kubernetes v1.24, 1.25 and 1.26. We will be using 1.24.10 on a TKG deployed on Azure
    6. Sign in to VMware Tanzu Network and accept or confirm that you have accepted the EULAs for each of the following:
        - https://network.tanzu.vmware.com/products/tanzu-cluster-essentials/#/releases/1238179 (Cluster Essentials for Tanzu)
        - https://network.tanzu.vmware.com/products/tanzu-application-platform/#/releases/1260040 (Tanzu App Platform)
    
```

## Create the environment
```
    1. Join the Management Cluster to TMC. 
    2. Create a Cluster Group (I create "TAP" cluster group)
    3. Deploy a TKG cluster 
    
            3 instance for Control plane (Standard_D4s_v3 = 8 vCPU and 16 GB RAM each in 3 availability zones)
            6 instances for Workers nodes (Standard_D8s_v3 = 8 vCPU and 16 GB RAM each) Two nodes by each availability zone
    
    4. Deploy Cert-Manager using the defaults
    5. Deploy Contour using the contour-values.yaml from this git
    6. Create the namespace tanzu-system-service-discovery 
    
            kubectl create ns tanzu-system-service-discovery
    
    7. Create un secreto con las credenciales de AWS
            
            kubectl create secret generic route53-credentials --from-literal=aws_access_key_id=<AWS_ACCESS_KEY_ID> --from-literal=aws_secret_access_key=<AWS_SECRET_ACCESS_KEY> -n tanzu-system-service-discovery
            
            for example
            
            kubectl create secret generic route53-credentials --from-literal=aws_access_key_id=ASIA3... --from-literal=aws_secret_access_key=VmM0EpFiTZM8RiNEWm.... -n tanzu-system-service-discovery
            
    8. Deploy DNS-external from TMC Catalog or with the CLI using the DNS-external-values.yaml from this git. Modify the file as needed.
            
            tanzu package available list external-dns.tanzu.vmware.com -A
            
                to know the available packages and to select the newest, for example the following version 0.12.2+vmware.4-tkg.2
                
            tanzu package install external-dns \
            --package external-dns.tanzu.vmware.com \
            --version 0.12.2+vmware.4-tkg.2 \
            --values-file external-dns.yaml \
            --namespace tanzu-system-service-discovery
    
    9. Create the namespace tanzu-system-registry
       
            kubectl create ns tanzu-system-registry
        
    10. Create a Cluster issuer using the following command

        kubectl apply -f - <<'EOF'
        apiVersion: cert-manager.io/v1
        kind: ClusterIssuer
        metadata:
          name: letsencrypt-contour-cluster-issuer
          namespace: tanzu-system-ingress
        spec:
          acme:
            email: "rdillon@vmware.com"
            privateKeySecretRef:
              name: acme-account-key
            server: https://acme-v02.api.letsencrypt.org/directory
            solvers:
            - http01:
                ingress:
                  class: contour
        EOF
    
    11. Request the certificate as follow
    
        kubectl apply -f - <<'EOF'
        apiVersion: cert-manager.io/v1
        kind: Certificate
        metadata:
          name: harbor-cert
          namespace: tanzu-system-registry
        spec:
          secretName: harbor-cert-tls
          duration: 2160h
          renewBefore: 360h
          subject:
            organizations:
            - vmware
          commonName: harbor.solateam.be
          isCA: false
          privateKey:
            size: 2048
            algorithm: RSA
            encoding: PKCS1
          dnsNames:
          - harbor.solateam.be
          - notary.harbor.solateam.be
          issuerRef:
            name: letsencrypt-contour-cluster-issuer
            kind: ClusterIssuer
        EOF
    
    12. Now we will expect to wait more or less 10 minutes until we could have the certificate. To verify it use the following command
    
            kubectl get certificates -n tanzu-system-registry harbor-cert
    
    13. When you have TRUE in ready column, you will continue to the next step that is obtain the certificates
        
            kubectl get secret harbor-cert-tls -n tanzu-system-registry -o=jsonpath={.data."tls\.crt"} | base64 --decode > tls-crt.txt
            
            kubectl get secret harbor-cert-tls -n tanzu-system-registry -o=jsonpath={.data."tls\.key"} | base64 --decode > tls-key.txt
        
    14. Verify if you have defined a default storage class. If you dont have it, use the following command (custom-storageclass is my storage class that is not default)
    
            kubectl patch storageclass custom-storageclass -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
    
    14. Deploy Harbor from the TMC Catalog or using the CLI. Use the harbor-values.yaml from this git. Modify the fields whith yours certificate and the name of your default storage class.
    
                kubectl create ns harbor
                
                tanzu package available list harbor.tanzu.vmware.com -A
                
                        Review the last version available. In my case 2.6.3+vmware.1-tkg.1
                        
                tanzu package install harbor \
                --package-name harbor.tanzu.vmware.com \
                --version 2.6.3+vmware.1-tkg.1 \
                --values-file harbor-values.yaml \
                --namespace harbor
                
                
    
    15. Connect to harbor and create "tap" project as public.
            
```      

## Configure the cluster
```
    1. We connect to the cluster
    
    2. Then run the following
    
        kubectl create clusterrolebinding default-tkg-admin-privileged-binding --clusterrole=psp:vmware-system-privileged --group=system:authenticated
            
    3. Create the namespace "tap-install"

```
## create the secret using TMC
```
        Secret name: tap-registry
        Image registry URL: registry.tanzu.vmware.com
        Username: <tanzu-net-username>
        Password: <tanzu-net-password>
        namespace: tap-install
        export to all namespaces

```

## Create the repository
```
        Name: tanzu-tap-repository
        Repository URL: registry.tanzu.vmware.com/tanzu-application-platform/tap-packages:1.5
```

## Obtain where is running ENVOY
```
      In our example we assume that is running on the following IP Address: 20.246.129.194
     
```  

## Configure Lets Encryt for the internal communications

```
    Run the cluster-issuer.yaml file
    
```

## Install Tanzu Application Platform package from TMC Catalog
```
    1. Select the Tanzu Application Platform
    
    2. Select tap as name and 1.5 for the version
    
    3. Copy the tap-values.yaml from this git. If you want to use Testing you will use the tap-values-w.testing.yaml
    
    4. Monitor the install by running
            
            tanzu package installed get tap -n tap-88xxx
            
    5. Verify all packages are successfully reconciled
            
            tanzu package installed list -A
          
     6. If you use the testing version of the supply chain we need to create a Tekton pipeline for the testing
     
            kubectl apply -f pipeline.yaml
```  

## Create developer namespace (single user access)
```
    1. Run the following command to Create the secret
    
            tanzu secret registry add registry-credentials --server harbor.solateam.be --username admin --password 'PASSw0rd2019202020212022' --namespace default
    
    2. To add secrets, a service account to execute the supply chain, and RBAC rules to authorize the service account to the developer namespace, run
    
cat <<EOF | kubectl -n default apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: tap-registry
  annotations:
    secretgen.carvel.dev/image-pull-secret: ""
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: e30K
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: default
secrets:
  - name: registry-credentials
imagePullSecrets:
  - name: registry-credentials
  - name: tap-registry
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: default-permit-deliverable
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: deliverable
subjects:
  - kind: ServiceAccount
    name: default
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding 
metadata:
  name: default-permit-workload
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: workload
subjects:
  - kind: ServiceAccount
    name: default
EOF

```

## OPTIONAL: Deploy the Tanzu Build Services full Depedencies (depending on the tap-values.yaml configuration of builservice)

```
    1. Setup environment variables
    
            export INSTALL_REGISTRY_USERNAME=MY-REGISTRY-USER
            export INSTALL_REGISTRY_PASSWORD=MY-REGISTRY-PASSWORD
            export INSTALL_REGISTRY_HOSTNAME=MY-REGISTRY
            export TAP_VERSION=VERSION-NUMBER
            export INSTALL_REPO=TARGET-REPOSITORY
            
            where

            INSTALL_REGISTRY_HOSTNAME is registry.tanzu.vmware.com
            INSTALL_REPO is tanzu-application-platform
            INSTALL_REGISTRY_USERNAME and INSTALL_REGISTRY_PASSSWORD are the credentials to run docker login registry.tanzu.vmware.com
            TAP_VERSION is your Tanzu Application Platform version. For example, 1.3.5
            
    2. Get the latest version of the buildservice package by running:

            tanzu package available list buildservice.tanzu.vmware.com --namespace tap-install

    3. Add the Tanzu Build Service full dependencies package repository by running:

            tanzu package repository add tbs-full-deps-repository \
            --url ${INSTALL_REGISTRY_HOSTNAME}/${INSTALL_REPO}/tbs-full-deps:VERSION \
            --namespace tap-install
            
            Where VERSION is the version of the buildservice package you retrieved earlier.

    4. Install the full dependencies package by running or update the current TAP deployment 

            tanzu package install full-tbs-deps -p full-tbs-deps.tanzu.vmware.com -v VERSION -n tap-install
            
            Where VERSION is the version of the buildservice package you retrieved earlier.
            
            or update
            
            tanzu package installed update tap -p tap.tanzu.vmware.com -v $TAP_VERSION  --values-file tap-values.yaml -n tap-install
            

```

## Review the Self-Guided Workshop
```
    1. run the following command to review the activated portals
            kubectl get trainingportals
    2. then connect to the defined URL
            http://learning-center-guided.solateam.be
```

## Deploy the different Learning Portals
```
    1. Run the following command

            kubectl apply -f learning/training-portal.yaml
            
            kubectl apply -f learning/portal.yaml
    
    2. Review the deployment
    
            kubectl get workshops
            
    3. To see what we have created
     
            kubectl get learningcenter-training -o name
     
    4. To see the sessions created
     
            kubectl get workshopsessions
            
    5. To found the portals information and admin users use the following
     
            kubectl get trainingportals
            
            
    6. To deploy a Spring Boot workshop, go to /learning on this git      
            
            kubectl apply -f resources/spring-workshop.yaml
            
            kubectl apply -f resources/spring-portal
            
            kubectl get trainingportals
            
            
    7. Use another example... learning-center-workshop-samples/lab-markdown-sample
     
            kubectl apply -f earning-center-workshop-samples/lab-markdown-sample/resources/workshop.yaml
            
            kubectl apply -f learning-center-workshop-samples/lab-markdown-sample/resources/training-portal.yaml
            
            kubectl get trainingportals
     
    With this we will have four Learning portals
    
            lab-k8s-fundamentals     http://lab-k8s-fundamentals-ui.solateam.be 
            lab-markdown-sample      http://lab-markdown-sample-ui.solateam.be
            lab-spring-boot-k8s-gs   http://lab-spring-boot-k8s-gs-ui.solateam.be
            learning-center-guided   http://learning-center-guided.solateam.be
    

```


## Configure VS Code
```
        1. Deploy VS Code on your machine
        2. Connect to Tanzu Network and download TANZU Developers Tools for Visual Studio Code
        3. Open VS Code.
        4. Press cmd+shift+P to open the Command Palette and run Extensions: Install from VSIX....
        5. Select the extension file tanzu-vscode-extension.vsix.
        6. If you do not have the following extensions, and they do not automatically install, install them from VS Code Marketplace:
                - Debugger for Java
                - Language Support for Java(™) by Red Hat
                - YAML
        7. Ensure Language Support for Java is running in Standard Mode. You can configure it in the Settings menu by going to Code > Preferences >      Settings under Java > Server: Launch Mode.
        8. When the JDK and Language Support for Java are configured correctly, you see that the integrated development environment creates a directory target where the code is compiled.
        9. Configure VMware Tanzu Developer Tools for VS Code
                - Confirm Delete: This controls whether the extension asks for confirmation when deleting a workload.
                - Enable Live Hover:  Reload VS Code for this change to take effect.
                - Source Image: solateam.be/tap
                - Namespace: default.   
        
```

## Using code to deploy an example

```
            
    1. Deploy the application using an example from github
    
            tanzu apps workload create tanzu-java-web-app \
            --git-repo https://github.com/RubenDillon/application-accelerator-samples \
            --sub-path tanzu-java-web-app \
            --git-branch main \
            --type web \
            --label app.kubernetes.io/part-of=tanzu-java-web-app \
            --yes \
            --namespace default
            
       If you deploy the Supply Chain with Test... we need to add the following to the deployment
       
             --label apps.tanzu.vmware.com/has-tests=true \
            
    2. View the build and runtime logs 
    
            tanzu apps workload tail tanzu-java-web-app --since 10m --timestamp --namespace default
            
            kubectl get workload,gitrepository,pipelinerun,images.kpack,podintent,app,services.serving
    
    3. After ends you could access the TAP GUI to see the process and using the following command
    
            tanzu apps workload get tanzu-java-web-app --namespace default
            
    4. To register the application, go to the tap-gui.solateam.be and click "Register Entity" and use the following as input
    
            https://github.com/RubenDillon/application-accelerator-samples/blob/main/tanzu-java-web-app/catalog/catalog-info.yaml
            
            
```

## Using VS Code to deploy and iterate the example

```
        1. Clone the repository
            
            git clone https://github.com/RubenDillon/application-accelerator-samples.git
            
         2. Open VS Code and open the TANZU-JAVA-WEB-APP folder from the cloned github resource
         
         3. Review the following files
         
                config/workload.yaml
                catalog/catalog-info.yaml
                tiltfile
       
         4. Use the TANZU:Live Update Start and review the terminal to see the process to attach the application to the platform
         
         5. Modify /src/main/java/com/example/springboot/HelloController.java 
         
                return "Greetings from Spring Boot + Tanzu!"; (change it to something and see the change in the application already deployed)    
   
```


## Deploy a more complex application

```

    1. Use the Application accelerator and select the "Where-to-Eat" application
    
    2. Use the defaults and download the zip file
    
    3. Upload to your github. In my case I upload into https://github.com/RubenDillon/where-to-eat
    
    4. Create the application into TAP
    
tanzu apps workload create where-to-eat \
--git-repo https://github.com/RubenDillon/where-to-eat \
--git-branch main \
--type web \
--label app.kubernetes.io/part-of=tanzu-where-to-eat \
--yes \
--namespace default

    5. View the build and runtime logs 
    
            tanzu apps workload tail where-to-eat --since 10m --timestamp --namespace default
    
    5. Iterate with the application (To be defined...)
    
                https://github.com/RubenDillon/application-accelerator-samples/blob/main/where-for-dinner/catalog/catalog-info.yaml
    
    
```

## Another
```
        Weather application using Steeltoe
        
            tanzu apps workload create weatherforecast-steeltoe \
            --git-repo https://github.com/RubenDillon/application-accelerator-samples \
            --sub-path weatherforecast-steeltoe \
            --git-branch main \
            --type web \
            --label app.kubernetes.io/part-of=weatherforecast-steeltoe \
            --yes \
            --namespace default
            
            
            To import use
            https://github.com/RubenDillon/application-accelerator-samples/blob/main/weatherforecast-steeltoe/catalog/catalog-info.yaml

```

## Steps to create a TAP workload from an existing application 
```
        Using a typical Hello World application based on .NET that I upload to my github
        
        1. Create the worload.yaml using the following command
        
                tanzu apps workload create my-workload --git-repo https://github.com/RubenDillon/Kubernetes/dotnet-core-hello-world.git --git-branch master > workload.yaml
        
        
            tanzu apps workload create hello-world \
            --git-repo https://github.com/RubenDillon/Kubernetes \
            --sub-path dotnet-core-hello-world \
            --git-branch main \
            --type web \
            --label app.kubernetes.io/part-of=hello-world \
            --yes \
            --namespace default

```

## Defining Scan policy and TAP-GUI access to scan
```
        1. Create the Scan policy applying the scan-policy.yaml and scan-template.yaml
        
                kubectl apply -f scan-policy.yaml
                
                kubectl apply -f scan-template.yaml
                
       ?? 2. Create a service account with read-write permission in the meta data store  
        
                kubectl apply -f create-svc-scan.yaml

       ?? 2. Retrieve the read-write secret from meta data store
        
                kubectl get secrets metadata-store-read-write-client -n metadata-store -o jsonpath="{.data.token}" | base64 -d
                
        3. Use the following configuration for TAP ( tap-values-w.test-scan.yaml to use the environment with Testing and Scanning) and change the BEARER in tap-gui for the data obtained in last point (no include the last sign "%")
        
        4. Update the application deployment setting the label has-tests to true
        
        
tanzu apps workload create tanzu-java-web-app \
  --git-repo https://github.com/vmware-tanzu/application-accelerator-samples \
  --sub-path tanzu-java-web-app \
  --git-branch main \
  --type web \
  --label apps.tanzu.vmware.com/has-tests=true \
  --label app.kubernetes.io/part-of=tanzu-java-web-app \
  --yes
        
        
   tanzu apps workload create hello-world \
            --git-repo https://github.com/RubenDillon/Kubernetes \
            --sub-path dotnet-core-hello-world \
            --git-branch main \
            --type web \
            --label apps.tanzu.vmware.com/has-tests=true \
            --label app.kubernetes.io/part-of=hello-world \
            --yes \
            --namespace default
   
   
        

```

## Commands to use with Tanzu Application Platform
```

    kubectl get ScanPolicy
    
    kubectl get PipeLine
    
    tanzu apps cluster-supply-chain list
    
    tanzu accelerator list
    
    tanzu accelerator create...
    
    kubectl get ImageScan
    
    kubectl get SourceScan
    
```



# APPENDIX

## Using Harbor as local registry to deploy TAP

```
        1. Connect to Harbor and to Tanzu Network
                docker login harbor.solateam.be
                docker login registry.tanzu.vmware.com
        
        2. Create the variables to run the commands
         
            export INSTALL_REGISTRY_USERNAME=admin
            export INSTALL_REGISTRY_PASSWORD=PASSw0rd2019202020212022
            export INSTALL_REGISTRY_HOSTNAME=harbor.solateam.be
            export TAP_VERSION=1.4.2
            export INSTALL_REPO=export INSTALL_REPO=tap
         
        3. To review which versions of TAP are available run the following
       
            imgpkg tag list -i registry.tanzu.vmware.com/tanzu-application-platform/tap-packages | grep -v sha | sort -V
                        

        4. Move the images to your local registry using the following
        
            imgpkg copy -b registry.tanzu.vmware.com/tanzu-application-platform/tap-packages:${TAP_VERSION} --to-repo ${INSTALL_REGISTRY_HOSTNAME}/${INSTALL_REPO}/tap-packages
            
            This will take almost 1h 20minutes


        5.  Get the latest version of the buildservice package by running:
            
                    tanzu package available list buildservice.tanzu.vmware.com --namespace tap-install
            
        6. Relocate the Tanzu Build Service full dependencies
         
                    imgpkg copy -b registry.tanzu.vmware.com/tanzu-application-platform/full-tbs-deps-package-repo:VERSION \
  --to-repo ${INSTALL_REGISTRY_HOSTNAME}/${INSTALL_REPO}/tbs-full-deps
  
                where VERSION is the number that we review in the last command. This process takes another 1h 30 minutes
         
         
        7.  Add the Tanzu Build Service full dependencies package repository 
            
                    tanzu package repository add tbs-full-deps-repository \
                    --url ${INSTALL_REGISTRY_HOSTNAME}/${INSTALL_REPO}/tbs-full-deps:VERSION \
                    --namespace tap-install
            
            where VERSION is the number that we review in the last command. 
            
        9. Create the tap-install namespace
         
            kubectl create ns tap-install
            
        10. Create a registry secret with the following command
         
                tanzu secret registry add tap-registry \
                --username ${INSTALL_REGISTRY_USERNAME} --password ${INSTALL_REGISTRY_PASSWORD} \
                --server ${INSTALL_REGISTRY_HOSTNAME} \
                --export-to-all-namespaces --yes --namespace tap-install
         
        11. Create the TAP repository
                
                tanzu package repository add tanzu-tap-repository \
                --url ${INSTALL_REGISTRY_HOSTNAME}/${INSTALL_REPO}/tap-packages:$TAP_VERSION \
                --namespace tap-install
         
        12. Use the following command to review the status of the repository
         
                tanzu package repository get tanzu-tap-repository --namespace tap-install
         
        13. Deploy using the following command (first you need download tap-values.yaml)
         
                tanzu package install tap -p tap.tanzu.vmware.com -v $TAP_VERSION --values-file tap-values.yaml -n tap-install
                
         14.  Install full dependencies package by running:
        
                tanzu package install full-tbs-deps -p full-tbs-deps.tanzu.vmware.com -v VERSION -n tap-install
                
        14. To review the process 
         
                tanzu package installed get tap -n tap-install
                  

```

### Troubleshooting “Builder default is not ready” message

```
        1. Restart kpack by deleting the kpack-controller and kpack-webhook pods in the kpack namespace. Deleting these resources triggers their recreation:
                kubectl delete pods --all --namespace kpack

        2. Verify status of the replacement pods:
                kubectl get pods --namespace kpack

        3. Verify the workload status after the new kpack pods STATUS are Running:
                tanzu apps workload get YOUR-WORKLOAD-NAME

```

### Troubleshoot namespace Terminating
```

    For example the ns tanzu-system-service-discovery is in Terminating status...
    
    kubectl get namespace tanzu-system-service-discovery -o json \
  | tr -d "\n" | sed "s/\"finalizers\": \[[^]]\+\]/\"finalizers\": []/" \
  | kubectl replace --raw /api/v1/namespaces/tanzu-system-service-discovery/finalize -f -
    
    
```


  Basado en 
  - https://confluence.eng.vmware.com/display/CNA/TAP+-+How+to+install+TAP+using+TMC
  - https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.0/tap/GUID-install.html
  - https://confluence.eng.vmware.com/pages/viewpage.action?spaceKey=CNA&title=Installing+Harbor+with+LE+certs+via+TMC
  - https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.5/tap/GUID-scst-store-retrieve-access-tokens.html
  - https://confluence.eng.vmware.com/display/MAPBUSUP/TAP+Support+Workshop+-+Day+1
  - https://docs.vmware.com/en/VMware-Tanzu-Kubernetes-Grid/2.1/tkg-deploy-mc-21/mgmt-reqs-prep-azure.html#tkg-app
  - https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.5/tap/tap-gui-plugins-scc-tap-gui.html#scan
  - https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.4/tap/tap-gui-troubleshooting.html
  - https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.4/tap/troubleshooting-tap-troubleshoot-using-tap.html

