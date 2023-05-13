# Deploy Tanzu Application Platform v1.5 using Azure with Harbor (using FREE public certificates)
## Deploy Test, Scan and Github authentication and GitOps  

## Requirements
```
    1. An Azure subscription to deploy TKG
    2. A management cluster deployed on Azure
    3. An AWS subscription for route53 DNS service (we create solateam.be domain)
    4. Tanzu Mission Control
    5. To deploy 
        - TAP 1.5.0 we need Kubernetes v1.24, 1.25 and 1.26. We will be using 1.24.10 on a TKG deployed on Azure
    6. Sign in to VMware Tanzu Network and accept or confirm that you have accepted the EULAs for each of the following:
        - https://network.tanzu.vmware.com/products/tanzu-cluster-essentials/#/releases/1238179 (Cluster Essentials for Tanzu)
        - https://network.tanzu.vmware.com/products/tanzu-application-platform/#/releases/1260040 (Tanzu App Platform)
    7. Follow the instructions to deploy the Management cluster on Azure (management-cluster-cfg.md, Azure-credentials.md)
    8. Review my Youtube channel where I deploy TAP on Azure (https://www.youtube.com/watch?v=hlf6Ton5MMA) 
    
```

## Create the environment
```
    1. Join the Management Cluster to TMC. 
    2. Create a Cluster Group (I create "TAP" cluster group)
    3. Deploy a TKG cluster (I use the tanzu cli on the video that I upload to Youtube) 
    
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
    
    4. Create a internal registry secret
    
    tanzu secret registry add registry-credentials \
    --server   harbor.solateam.be \
    --username admin \
    --password "PASSw0rd2019202020212022" \
    --namespace tap-install \
    --export-to-all-namespaces \
    --yes

```
## create the Tanzu Net secret using TMC
```
        Secret name: tap-registry
        Image registry URL: registry.tanzu.vmware.com
        Username: <your-tanzu-net-username>
        Password: <your-tanzu-net-password>
        namespace: tap-install
        export to all namespaces

```

## Create the repository
```
        Name: tanzu-tap-repository
        Repository URL: registry.tanzu.vmware.com/tanzu-application-platform/tap-packages:1.5.0

```

## Review the status of the repository
```
Run the following command
	
	tanzu package repository list -A
	
	tanzu package repository get tanzu-tap-repository -n tkg-system

```

## Obtain where is running ENVOY
```
This is needed to register this address in the DNS namespaces (harbor.solateam.be, tap-gui.solateam.be....)
      
      	kubectl get svc -A | grep envoy
     
```  

## Configure Lets Encryt for the internal communications

```
Run the cluster-issuer.yaml file

	kubectl apply -f cluster-issuer.yaml
    
```

## Create a GIT repository (for future use)
```
	Using your Github account create a New Repository, as public and name it "tap-gitops"
	
	Using your machine select where you want to clone this repository
	
		git clone https://github.com/RubenDillon/tap-gitops.git (for example using my account)

	Then create a Git repository

    		git init
    		XX git remote add origin git@github.com:ruben-latam/tap-gitops.git

		create a file ("prueba.txt") on the directory then summit to the github
		
		git add prueba.txt
		git commit -m "Mensaje de confirmación"
		git push

	Review that the file were uploaded to tap-gitops

```

## Github authentication
```

- https://backstage.spotify.com/learn/standing-up-backstage/configuring-backstage/7-authentication/

        1. Create a new public project in Harbor and call it "tap-apps"

	2. Go to https://github.com/settings/applications/new to create your OAuth App.

		Homepage URL should be https://github.com/login/oauth/authorize
		Authorization callback URL should point to the auth backend, https://tap-gui.solateam.be/api/auth/github
		
	3. Generate a new Client Secret and take a note of the Client ID and the Client Secret
	
	4. Modify the tap-gui part of the tap-values-OOTB-test-scan-auth.yaml file to looks like the following with your values
	
tap_gui:
  app_config:
    auth:
      environment: development
      providers:
        github:
          development:
            clientId: Iv1.61f0b55ce5c4a0c0
            clientSecret: b9f8a2db250dcdfab889d64d33aa333a3b5822f3
    catalog:
      locations:
        - target: https://github.com/RubenDillon/tap/blob/main/catalog-info.yaml
          type: url
  metadataStoreAutoconfiguration: true
  service_type: ClusterIP
	
where ClientID is obtained from Github Apps (Developer settings) and ClientSecret.. is the client secret generated for that App in Github		

	5. Create a new personal Token on Github (your user, settings, developer)
	
	6. Create a Secret on the default namespace as git-secret.yaml (from this github)
	
	7. Apply the secret
	
	8. Modify the ootb_Supply_chain_test_scan part of the tap-values-OOTB-test-scan-auth.yaml file to looks like the following with your values

ootb_supply_chain_test_scan:
  registry:
    server: "harbor.solateam.be"
    repository: "tap-apps"

  gitops:
    server_address: https://github.com/
    repository_owner: RubenDillon
    branch: main
    username: rubendillon
    email: ruben_dillon@hotmail.com
    commit_message: supplychain@cluster.local
    ssh_secret: github-http-secret                 # secret created on the default namespace
    commit_strategy: direct

  cluster_builder: default
  service_account: default


```


## Install Tanzu Application Platform package from TMC Catalog
```
    1. Select the Tanzu Application Platform
    
    2. Select tap as name and 1.5 for the version
    
    3. Copy the tap-values-OOTB-test-scan-auth.yaml on the TMC UI
    
    4. Monitor the install by running
            
            tanzu package installed get tap -n tap-88xxx
            
    5. Verify all packages are successfully reconciled
            
            tanzu package installed list -A
          
    6. The UI (tap-gui) receive a public certificate from lets encrypt. If you want to review it 
    
    	    kubectl get secret -n tap-gui tap-gui-cert -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -text
	    
```  

## Prepare the developer namespace
```

 	1. Create a namespace using kubectl (if you change the namespace will be used for developers)
		kubectl create namespace xxxy (we are using default namespace.. for that we don't need to create it)

	2. Label the namespace
		kubectl label namespaces default apps.tanzu.vmware.com/tap-ns=""
	
	3. Review the configuration 
		kubectl get secrets,serviceaccount,rolebinding,pods,workload,configmap,limitrange -n default
```

## Configure the Test and Scanning components
```

 - https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.5/tap/getting-started-add-test-and-security.html
 - https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.5/tap/namespace-provisioner-ootb-supply-chain.html#testing--scanning-supply-chain-3  

       1. Create the Scan policy applying the scan-policy.yaml and scan-template.yaml
        
                kubectl apply -f scan-policy.yaml.  
		
		Because we are not using the default scan-policy we apply this one that creates scan-policy-free policy, 
		and we define their use in tap-values
                
               revisar si no se tiene que usar ----> kubectl apply -f scan-template.yaml
                
	2. Review Tekton pipeline and scan policies 
     
	 	kubectl get pipeline.tekton.dev,scanpolicies
```

### GitOps integration
```


	1. Update (patch) the default service account with the new secret
	
		kubectl patch serviceaccount default --patch '{"secrets": [{"name": "github-http-secret"}]}'
		
	2. Review the default service account
	
		kubectl describe serviceaccount default
		
```
        
### Test an Example
```
	
	1. Deploy the following example
	
	
		tanzu apps workload apply tanzu-java-web-app \
		--git-repo https://github.com/RubenDillon/application-accelerator-samples \
		--sub-path tanzu-java-web-app \
		--git-branch main \
		--type web \
		--app tanzu-java-web-app \
		--label apps.tanzu.vmware.com/has-tests="true" \
		--param-yaml testing_pipeline_matching_labels='{"apps.tanzu.vmware.com/language": "java"}' \
		--tail \
		--yes
	
	
	  2. For scan policy, we already define a default in our configuration (tap-values...yaml), but if we dont do that.. 
	  the default policy (scan-policy) is very restrictive and we need to define which one.. we want to use..
	
		--param scanning_source_policy="lax-scan-policy" \
		--param scanning_image_policy="lax-scan-policy" \
	
	  3. View the build 
            
           	 kubectl get workload,gitrepository,pipelinerun,images.kpack,podintent,app,services.serving
    
   	  4. After ends you could access the TAP GUI to see the process and review the app using the following command
    
            	tanzu apps workload get tanzu-java-web-app --namespace default
            
          5. To register the application, go to the tap-gui.solateam.be and click "Register Entity" and use the following as input
    
            	https://github.com/RubenDillon/application-accelerator-samples/blob/main/tanzu-java-web-app/catalog/catalog-info.yaml



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
                - Source Image: harbor.solateam.be/tap-apps/tanzu-java-web-app
                - Namespace: default.   
        
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
		
	 4. Probably (if you use another name to your cluster) you will need to add a first line providing the name of the cluster
	 
	 	allow_k8s_contexts('<name of your cluster>')
	 
	 5. Remember that you probably need to add the label for pipeline (needs to looks like the following)
	 
	 	" --param-yaml testing_pipeline_matching_labels='{"+"apps.tanzu.vmware.com/language"+": "+"java"+"}' " +
	 
       
         6. Use the TANZU:Live Update Start and review the terminal to see the process to attach the application to the platform
         
         7. Modify /src/main/java/com/example/springboot/HelloController.java 
         
                return "Greetings from Spring Boot + Tanzu!"; (change it to something and see the change in the application already deployed)    
   
```


## Try another applications as example
```
            A Weather application using Steeltoe framework (.NET core)
        
tanzu apps workload create weatherforecast-steeltoe \
            --git-repo https://github.com/RubenDillon/application-accelerator-samples \
            --sub-path weatherforecast-steeltoe \
            --git-branch main \
            --type web \
            --label app.kubernetes.io/part-of=weatherforecast-steeltoe \
	    --param-yaml testing_pipeline_matching_labels='{"apps.tanzu.vmware.com/language": "java"}' \
            --label apps.tanzu.vmware.com/has-tests=true \
            --yes --tail\
            --namespace default
	    
            To import use
            https://github.com/RubenDillon/application-accelerator-samples/blob/main/weatherforecast-steeltoe/catalog/catalog-info.yaml


 	    Another Java application 

	    tanzu apps workload create java-server-side-ui \
            --git-repo https://github.com/RubenDillon/application-accelerator-samples \
            --sub-path java-server-side-ui \
            --git-branch main \
            --type web \
            --label app.kubernetes.io/part-of=java-server-side-ui \
	    --param-yaml testing_pipeline_matching_labels='{"apps.tanzu.vmware.com/language": "java"}' \
	    --label apps.tanzu.vmware.com/has-tests=true \
            --yes --tail \
            --namespace default
	    

            To import use
	    https://github.com/RubenDillon/application-accelerator-samples/blob/main/java-server-side-ui/catalog/catalog-info.yaml


	    
	    An Angular front end application
	    
	    tanzu apps workload create angular-frontend \
            --git-repo https://github.com/RubenDillon/application-accelerator-samples \
            --sub-path angular-frontend \
            --git-branch main \
            --type web \
            --label app.kubernetes.io/part-of=angular-frontend \
	    --param-yaml testing_pipeline_matching_labels='{"apps.tanzu.vmware.com/language": "java"}' \
	    --label apps.tanzu.vmware.com/has-tests=true \
            --yes --tail \
            --namespace default
	    
	    To import use
	    https://github.com/RubenDillon/application-accelerator-samples/blob/main/angular-frontend/catalog/catalog-info.yaml
	    
	    
	    
	    A Node.js (using express.js) example
	    
	    tanzu apps workload create node-express \
            --git-repo https://github.com/RubenDillon/application-accelerator-samples \
            --sub-path node-express \
            --git-branch main \
            --type web \
            --label app.kubernetes.io/part-of=node-express \
	    --param-yaml testing_pipeline_matching_labels='{"apps.tanzu.vmware.com/language": "java"}' \
	    --label apps.tanzu.vmware.com/has-tests=true \
            --yes --tail \
            --namespace default

		NOTE: use --label apps.tanzu.vmware.com/has-tests=true only if you configure testing on the supply chain
	    
	    
	    To import use
	    https://github.com/RubenDillon/application-accelerator-samples/blob/main/node-express/catalog/catalog-info.yaml
	    
	    
	    
	    Another (but.. this is from an image)
	    
	    tanzu apps workload create simple-web-app \
            --image ghcr.io/vmware-tanzu-learning/simple-web-app:v1.1.0 \
	    --param-yaml testing_pipeline_matching_labels='{"apps.tanzu.vmware.com/language": "java"}' \
	    --label apps.tanzu.vmware.com/has-tests=true \
            --type web \
            --yes
	    
	    
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
	    or
	    	kubectl rollout restart deployment/kpack-controller -n kpack

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
  - https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.5/tap/getting-started-add-test-and-security.html
