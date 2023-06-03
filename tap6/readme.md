# Deploy Tanzu Application Platform v1.5.1 on Azure Cloud
# using AKS with Harbor (using FREE public certificates)
## Deploy Test, Scan and Github authentication and GitOps   

## Requirements
```
    1. An Azure subscription to deploy AKS
    3. An AWS subscription for route53 DNS service (we create solateam.be domain)
    4. Tanzu Mission Control
    5. To deploy 
        - TAP 1.5.0 we need Kubernetes v1.24, 1.25 and 1.26. We will be using 1.26.3 on a AKS deployed on Azure
    6. Sign in to VMware Tanzu Network and accept or confirm that you have accepted the EULAs for each of the following:
        - https://network.tanzu.vmware.com/products/tanzu-cluster-essentials/#/releases/1238179 (Cluster Essentials for Tanzu)
        - https://network.tanzu.vmware.com/products/tanzu-application-platform/#/releases/1260040 (Tanzu App Platform)
    
```

## Create the environment
```
    1. Deploy azure client or upgrade it (in my case, Im using a Mac)
          brew update && brew install azure-cli

    2. Log to Azure 
          az login
    
    3. Define your subscription
          az account set --subscription dddddxxxx-xx-xxxxx-xxxxxxxx
    
    4. Create the resource group where AKS will be deployed
          az group create --name TAP-RG --location eastus
          
    6. Verify the kubernetes versions available for AKS
          az aks get-versions --location eastus
          
    5. Create the AKS cluster
          az aks create -g TAP-RG -n tap-on-azure --enable-managed-identity --node-count 6 --enable-addons monitoring --enable-msi-auth-for-monitoring --generate-ssh-keys --node-vm-size Standard_D4ds_v4 --kubernetes-version 1.26.3

    6. Deploy kubectl
          sudo su
          az aks install-cli
          
    7. Download the kubectl 
          az aks get-credentials --name tap-on-azure --resource-group TAP-RG
          
    8. Test the connection
          kubectl get nodes

    9. Create a Cluster Group (I create "TAP" Cluster group)
    
    10. Attach the AKS Cluster to TMC under TAP cluster group 
    
    11. Activate the Helm Chart Service and deploy Cert-Manager using the defaults from Helm Chart
    
    12. Deploy Contour using the defaults from Helm Chart 
    
    13. Obtain the Public IP Address where Envoy is running
    
    		kubectl get svc -A | grep envoy
		
	In our case is running on 20.124.139.24
      
    14. Create the namespace tanzu-system-registry
       
            kubectl create ns tanzu-system-registry
        
    15. Create a Cluster issuer using the following command

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
          commonName: harbor.tap.solateam.be
          isCA: false
          privateKey:
            size: 2048
            algorithm: RSA
            encoding: PKCS1
          dnsNames:
          - harbor.tap.solateam.be
          - notary.harbor.tap.solateam.be
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

## Create a GIT repository 
```
	Using your Github account create a New Repository, as public and name it "tap-gitops"
	
	Using your machine select where you want to clone this repository
	
		git clone https://github.com/RubenDillon/tap-gitops.git (for example using my account)

	Then create a Git repository

    		git init
    		XX git remote add origin git@github.com:RubenDillon/tap-gitops.git

		create a file ("prueba.txt") on the directory then summit to the github
		
		git add prueba.txt
		git commit -m "Mensaje de confirmación"
		git push

	Review that the file were uploaded to tap-gitops
	
	Connect to Tanzu Networks and go to Tanzu Application Platform. Select tap-gui-catalogs-latest and then the catalog with yelb application
	(Tanzu Application Platform GUI Yelb Catalog)
	
	Download and copy at tap-gitops, then restore the file 
	
		tar -xvf tap-gui-yelb-catalog.tgz
		
	Upload the catalog
	
		git add .
		git commit -m "adding catalog"
		git branch -M main
		git push -u origin main
		
	Review that the file were uploaded to tap-gitops	


```

## Github authentication
```

- https://backstage.spotify.com/learn/standing-up-backstage/configuring-backstage/7-authentication/
- https://backstage.io/docs/getting-started/configuration/#setting-up-a-github-integration
- https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.5/tap/scc-git-auth.html
- https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.5/tap/tap-gui-plugins-application-accelerator-git-repo.html#configuration

        1. Create a new public project in Harbor and call it "tap-apps" and "tap-gitops"

	2. Go to https://github.com/settings/applications/new to create your OAuth App.

		Homepage URL should be https://github.com/login/oauth/authorize
		Authorization callback URL should point to the auth backend, https://tap-gui.solateam.be/api/auth/github
		
	   The set of permissions granted to the application are: api, read_api, read_user, read_repository, write_repository, openid, and email.
		
	3. Generate a new Client Secret and take a note of the Client ID and the Client Secret
	
	4. Modify the tap-gui part of the tap-values-OOTB-test-scan-auth.yaml file to add ClientId and clientSecret values

		where ClientID is obtained from Github Apps (Developer settings) and ClientSecret.. is the client secret generated for that App in Github		

	5. Create a new personal Token on Github (your user, settings, developer)
	
	6. Create a Secret on the default namespace as git-secret.yaml (from this github). Use the token as password and modify the user name.
	
	7. Apply the secret
	
	8. Modify the ootb_Supply_chain_test_scan part of the tap-values-OOTB-test-scan-auth.yaml with your data.


```

### GitOps integration
```

	1. Update (patch) the default service account with the new secret
	
		kubectl patch serviceaccount default --patch '{"secrets": [{"name": "github-http-secret"}]}'
		
	2. Review the default service account
	
		kubectl describe serviceaccount default
		
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
        
### Test an Example
```

	1. Connect to your github environment and create a new repository
	
	2. Select to "import" and then copy the following link "https://github.com/vmware-tanzu-learning/tanzu-java-web-app"
	
	3. Now we have created a repository with the address like https://github.com/<your user>/tanzu-java-web-app. In my case 
	
		https://github.com/RubenDillontanzu-java-web-app
	
	4. Now we will be deploying the example using that repository
	
		tanzu apps workload apply tanzu-java-web-app \
		--git-repo https://github.com/RubenDillon/tanzu-java-web-app \
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


	  6. Review the deliverables of the deployment
	 

		kubectl get deliverables
		
		kubectl get deliverable tanzu-java-web-app -o yaml > deliverable-tanzu-java-web-app.yaml


		7. Review the Knative service of the deployment
		
		kubectl get service.serving.knative.dev/tanzu-java-web-app


```

## Using KubeNEAT to generate a readable file 
```

	1. Deploy KubeNEAT

          wget -O - https://github.com/itaysk/kubectl-neat/releases/download/v2.0.3/kubectl-neat_linux_amd64.tar.gz | \
          sudo tar -C /usr/local/bin -zxvf - kubectl-neat

	2. Use it to review a better readable file

		kubectl-neat < deliverable-tanzu-java-web-app.yaml > deliverable-limpio.yaml

	3. You could use that file to deploy the application into another cluster. For example, this environment will be for
	   development and another cluster (run cluster) for production. In that cluster, we could deploy with this
	   deliverable file.

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
        
	10. Do the same with the "Tanzu App Accelerator Extension for Visual Studio Code" 
	
	11. Once deployed configure tap-gui.solateam.be as the TAP GUI URL
```

## Using VS Code to deploy and iterate the example

```
        1. Copy the following repository to tap-gitops
            
            https://github.com/RubenDillon/application-accelerator-samples.git
            
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
		
		
	 8. Using the terminal review your current path using pwd. Then move to the tanzu-java-web-app folder and commit the code
	 
	 	git -C /Users/rubendillon/tanzu/tap-gitops/tanzu-java-web-app add .
		
		git -C /Users/rubendillon/tanzu/tap-gitops/tanzu-java-web-app commit -a -m "Initial Commit of Tanzu Java Web App"
		
		git push
		
		
		
	  7. Revisar el archivo delivery.yml y subirlo al git
	  
	  	 git -C /Users/rubendillon/tanzu/tap-gitops/config/default/tanzu-java-web-app add delivery.yml
		 
		 git -C /Users/rubendillon/tanzu/tap-gitops/config/default/tanzu-java-web-app  commit -a -m "Adding deliverable"
	 
	 	 git -C /Users/rubendillon/tanzu/tap-gitops/config/default/tanzu-java-web-app pull -r
		 
		 git -C /Users/rubendillon/tanzu/tap-gitops/config/default/tanzu-java-web-app push -u origin main
		 
		 
		 
		 
	   8. From the Application Accelerator link from the toolbar select create an App
	   
	   9. Select Tanzu Java Web App and complete the following
	   		Use JAVA 17 and Spring Boot v3.0
	   		harbor.solateam.be/tap-apps
			
		Complete the git repository information
			Owner: RubenDillon
			Repository Name: tanzu-java-web-app2
			Repository Branch: main
			
	    10. Open the app and modify the tiltfile to see like the following
	    
allow_k8s_contexts('tkg2-tap-01')

SOURCE_IMAGE = os.getenv("SOURCE_IMAGE", default='harbor.solateam.be/tap-apps/tanzu-java-web-app3-source')
LOCAL_PATH = os.getenv("LOCAL_PATH", default='.')
NAMESPACE = os.getenv("NAMESPACE", default='default')
OUTPUT_TO_NULL_COMMAND = os.getenv("OUTPUT_TO_NULL_COMMAND", default=' > /dev/null ')

k8s_custom_deploy(
    'tanzu-java-web-app3',
    apply_cmd="tanzu apps workload apply -f config/workload.yaml --debug --live-update" +
               " --local-path " + LOCAL_PATH +
               " --source-image " + SOURCE_IMAGE +
               " --namespace " + NAMESPACE +
               " --label apps.tanzu.vmware.com/has-tests=true " +
               " --param-yaml testing_pipeline_matching_labels='{"+"apps.tanzu.vmware.com/language"+": "+"java"+"}' " +
               " --yes " +
               OUTPUT_TO_NULL_COMMAND +
               " && kubectl get workload tanzu-java-web-app3 --namespace " + NAMESPACE + " -o yaml",
    delete_cmd="tanzu apps workload delete -f config/workload.yaml --namespace " + NAMESPACE + " --yes",
    deps=['pom.xml', './target/classes'],
    container_selector='workload',
    live_update=[
      sync('./target/classes', '/workspace/BOOT-INF/classes')
    ]
)

k8s_resource('tanzu-java-web-app3', port_forwards=["8080:8080"],
            extra_pod_selectors=[{'carto.run/workload-name': 'tanzu-java-web-app3', 'app.kubernetes.io/component': 'run'}])
	    
	    
	    
	    11. Use the LiveView to deploy it into TAP... wait until you need to Approve the Request... and finally see the deployment.
	
   
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
	    
	    
	    or using our new Repository... tap-gitops
	    
	    tanzu apps workload create weatherforecast-steeltoe \
            --git-repo https://github.com/RubenDillon/tap-gitops \
            --sub-path weatherforecast-steeltoe \
            --git-branch main \
            --type web \
            --label app.kubernetes.io/part-of=weatherforecast-steeltoe \
	    --param-yaml testing_pipeline_matching_labels='{"apps.tanzu.vmware.com/language": "java"}' \
            --label apps.tanzu.vmware.com/has-tests=true \
            --yes --tail\
            --namespace default
	  

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

		
	  or
	  
	    tanzu apps workload create java-server-side-ui \
            --git-repo https://github.com/RubenDillon/tap-gitops \
            --sub-path java-server-side-ui \
            --git-branch main \
            --type web \
            --label app.kubernetes.io/part-of=java-server-side-ui \
	    --param-yaml testing_pipeline_matching_labels='{"apps.tanzu.vmware.com/language": "java"}' \
	    --label apps.tanzu.vmware.com/has-tests=true \
            --yes --tail \
            --namespace default



	    
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
	    
	    or
	    
	     tanzu apps workload create angular-frontend \
            --git-repo https://github.com/RubenDillon/tap-gitops \
            --sub-path angular-frontend \
            --git-branch main \
            --type web \
            --label app.kubernetes.io/part-of=angular-frontend \
	    --param-yaml testing_pipeline_matching_labels='{"apps.tanzu.vmware.com/language": "java"}' \
	    --label apps.tanzu.vmware.com/has-tests=true \
            --yes --tail \
            --namespace default
	    
	    
	    
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
	    
	    or use
	    
	    tanzu apps workload create node-express \
            --git-repo https://github.com/RubenDillon/tap-gitops \
            --sub-path node-express \
            --git-branch main \
            --type web \
            --label app.kubernetes.io/part-of=node-express \
	    --param-yaml testing_pipeline_matching_labels='{"apps.tanzu.vmware.com/language": "java"}' \
	    --label apps.tanzu.vmware.com/has-tests=true \
            --yes --tail \
            --namespace default
	    
	    
	    Another (but.. this is from an image)
	    
	    tanzu apps workload create simple-web-app \
            --image ghcr.io/vmware-tanzu-learning/simple-web-app:v1.1.0 \
	    --label apps.tanzu.vmware.com/has-tests=false \
            --type web \
            --yes
	    
	    curl -k https://simple-web-app.default.solateam.be/hello
	 
	
	    
	   Pendiente ------------------- ver como agregar una imagen comun ---------------------------    
	   ------------------------------------ no levanta el POD ya que no sabe que esta ------------
	   ----------------------------- en el puerto 8080 -------------------------------------------
	    
	    
	    tanzu apps workload create smario-web-app \
            --image bharathshetty4/supermario:latest \
	    --label apps.tanzu.vmware.com/has-tests=false \
            --type web \
            --yes
	
	
	
	tanzu apps workload apply tanzu-java-web-app \
		--git-repo https://github.com/RubenDillon/tap-gitops \
		--sub-path tanzu-java-web-app \
		--git-branch main \
		--type web \
		--app tanzu-java-web-app \
		--label apps.tanzu.vmware.com/has-tests="true" \
		--param-yaml testing_pipeline_matching_labels='{"apps.tanzu.vmware.com/language": "java"}' \
		--tail \
		--yes
	
	
	
```

## Automate the reading of Catalogs
```
	Modify the TAP installation adding the following to tap-gui
	
	- type: github-discovery
          target: https://github.com/RubenDillon/tap-gitops/blob/main/*/catalog/catalog-info.yaml

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

## Service Bindings
```
	- https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.5/tap/getting-started-claim-services.html
	- https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.5/tap/getting-started-consume-services.html
	
	1. To review the default deployment that we have it, use the following
	
		tanzu service class list
	
	2. To review RabbitMQ, the service that we will be using
	
		tanzu service class get rabbitmq-unmanaged
	
	3. Create a claim of Rabbit with a 3GB space
	
		tanzu service class-claim create rabbitmq-1 --class rabbitmq-unmanaged --parameter storageGB=3
	
	4. Review the progress running
	
		tanzu services class-claims get rabbitmq-1 --namespace default
	
	5. Now we deploy an example that consumes, this RabbitMQ. This application have two pieces, the consumer web and the producer.
	
  tanzu apps workload create spring-sensors-consumer-web \
  --git-repo https://github.com/tanzu-end-to-end/spring-sensors \
  --git-branch rabbit \
  --type web \
  --param-yaml testing_pipeline_matching_labels='{"apps.tanzu.vmware.com/language": "java"}' \
  --label apps.tanzu.vmware.com/has-tests=true \
  --label app.kubernetes.io/part-of=spring-sensors-rabbit \
  --annotation autoscaling.knative.dev/minScale=1 \
  --service-ref="rmq=services.apps.tanzu.vmware.com/v1alpha1:ClassClaim:rabbitmq-1"
	
	6. And run the following app as Producer
	

  tanzu apps workload create spring-sensors-producer \
  --git-repo https://github.com/tanzu-end-to-end/spring-sensors-sensor \
  --git-branch main \
  --type web \
  --param-yaml testing_pipeline_matching_labels='{"apps.tanzu.vmware.com/language": "java"}' \
  --label apps.tanzu.vmware.com/has-tests=true \
  --label app.kubernetes.io/part-of=spring-sensors-rabbit \
  --annotation autoscaling.knative.dev/minScale=1 \
  --service-ref="rmq=services.apps.tanzu.vmware.com/v1alpha1:ClassClaim:rabbitmq-1" \
  --tail -y

	7. Go to the consumer-web deployment and wait until the Producer is running and giving information to RabbitMQ

	8. Add the app to the catalog

             https://github.com/tanzu-end-to-end/spring-sensors-sensor/blob/main/catalog/catalog-info.yaml
	     
	   
        9. If we want to do a Service Binding with a Database (PostgreSQL for example), the procedure is more or less the same
	
	
		tanzu service class-claim create psql-1 --class postgresql-unmanaged --parameter storageGB=3
		
		tanzu services class-claims get psql-1
		
		tanzu service class-claim get psql-1
		
		tanzu apps workload create my-workload --image my-registry/my-app-image --service-ref db=services.apps.tanzu.vmware.com/v1alpha1:ClassClaim:psql-1
	     
	     
```

## A more complex application including MySQL as Service Binding
```

	1. Create a Spring-Sensors fork from https://github.com/RubenDillon/spring-sensors
	
	2. Do a git clone from your repository
	
		git clone https://github.com/<your github>/spring-sensors
	
	2. Open the folder on VS Code
	
	3. Run the application using the tiltfile and add the following information
		
		allow_k8s_contexts('tkg2-tap-01')
		
		
               " --label apps.tanzu.vmware.com/has-tests=true " +
               " --param-yaml testing_pipeline_matching_labels='{"+"apps.tanzu.vmware.com/language"+": "+"java"+"}' " +
		
		
	4. Wait until you have to approve the Pull Request. Press the Approval button on "Config Writer" step. 
	
	5. Select Merge in github and wait until the app be running and available
	
	6. Review from your local machine where spring-sensors is copied, go to the folder config and edit workload.yaml
	
		Modify the part where is the source code with your github space
	
	6. Using the terminal review your current path using pwd. Then move to the tap-gitops folder and commit the code
	 
	 	git -C /Users/rubendillon/tanzu/spring-sensors add .
		
		git -C /Users/rubendillon/tanzu/spring-sensors commit -a -m "Initial Commit of Spring Sensors"
		
		git push
		
		
	7. Now is time to create the deliverable.yaml file into gitops-deliverables folder to move the deployment to the run (production environment)


apiVersion: carto.run/v1alpha1
kind: Deliverable
metadata:
  name: spring-sensors
  labels:
    app.tanzu.vmware.com/deliverable-type: web
    app.kubernetes.io/part-of: spring-sensors
    app.kubernetes.io/component: deliverable
spec:
  source:
    git:
      ref:
        branch: main
      url: https://github.com/RubenDillon/spring-sensors	
	
		
	7. Subirlo al git
	  
	  	 git -C /Users/rubendillon/tanzu/spring-sensors/gitops-deliverables add deliverable.yaml
		 
		 git -C /Users/rubendillon/tanzu/spring-sensors/gitops-deliverables  commit -a -m "Adding deliverable"
	 
	 	 git -C /Users/rubendillon/tanzu/spring-sensors/gitops-deliverables pull -r
		 
		 git -C /Users/rubendillon/tanzu/spring-sensors/gitops-deliverables push -u origin main
		 
	8. Create the data base for the application
	
		tanzu service class list
		
		tanzu service class-claim create sensors-mysql --class mysql-unmanaged --parameter storageGB=3
		
		tanzu services class-claims get sensors-mysql
	
	8. Add the Data Services, we need to open the workload.yaml file from config
	
		  serviceClaims:
                    - name: db
                      ref:
                        apiVersion: services.apps.tanzu.vmware.com/v1alpha1
                        kind: ClassClaim
                        name: sensors-mysql	
	
	9. Push the changes to github
	
		 git -C /Users/rubendillon/tanzu/sp
		 ring-sensors/config add .
		 
		 git -C /Users/rubendillon/tanzu/spring-sensors/config  commit -a -m "Adding Data Services"
	 
	 	 git -C /Users/rubendillon/tanzu/spring-sensors/config pull -r
		  
		 git -C /Users/rubendillon/tanzu/spring-sensors/config push -u origin main
		 
		 
	10. Run the tiltfile again and wait up to have the Pull Request requirement in "Config Writer". 
	
	11. Approve the merge and wait until the app will be ready
	
	12. Now we could make modification to introduce problems with security. Modify spring-sensors/pom.xml from VS Code
	
		Search org.hsqldb
		
		replace 2.7.1 by 2.7.0
	
	13. Save the file
	
	14. Push the changes
	
	
		git -C /Users/rubendillon/tanzu/spring-sensors commit -a -m "Initial Commit of Spring Sensors"
		
	 	 git -C /Users/rubendillon/tanzu/spring-sensors pull -r
		  
		 git -C /Users/rubendillon/tanzu/spring-sensors push -u origin main
		 
		 
	15. You will found a vulnerability runned by Snyk on your VSCode or in the TAP-GuI the supply chain (hsqldb version 2.7.0)		
		
	
	16. To add this application to the Catalog, register the following Entity
	
		https://github.com/RubenDillon/spring-sensors/blob/main/catalog-info.yaml	
		
	

```

## Configure Persitent Storage for the GUI
```

	- https://snapshooter.com/learn/postgresql/deploy-postgres-with-docker
	- https://hevodata.com/learn/docker-postgresql
	- https://backstage.spotify.com/learn/standing-up-backstage/configuring-backstage/5-config-2
	- https://blog.devart.com/configure-postgresql-to-allow-remote-connection.html	
	
	1. To provide a persisten storage for the TAP-GUI we need a PostgreSQL database. For that reason we create another instance 
	
	2. That new instance we created as follow
		Ubuntu 20.04 LTS Server
		4vCPU and 16 GB RAM (Standard D4s v3)
		open ports 22, 80, 443 and 5432
		connected to the network where the worker nodes of the TAP workload is running
		postgresql as name
		
	3. Deploy PostgreSQL
		
		sudo apt-get install postgresql
		sudo -u postgres psql
			you will receive the information about PostgreSQL version
		postgres=# ALTER USER postgres PASSWORD 'secreto';
		\l
			this will list the current databases on PostgreSQL
		CREATE DATABASE prueba;
			this will create a test database 
		\l
			review the databases 
		\q
			to exit
			
	4. Configure PostgreSQL to receive remote connections
		vi /etc/postgresql/12/main/postgresql.conf
			replace #listen_addresses = 'localhost' for #listen_addresses = '*'	
		vi /etc/postgresql/12/main/pg_hba.con
		    	replace....	host    all             all             127.0.0.1/32            md5 
			by......... 	host    all             all             0.0.0.0/0           	md5 
			add........	hostssl	all		all		0.0.0.0/0		md5
		sudo ufw allow 5432/tcp 
		sudo ufw allow 22/tcp 
		sudo service postgresql restart			
			
	5. Test the connection from another machine
		sudo apt-get install postgresql-client
		psql -U postgres -p 5432 -h postgresql.solateam.be
			where postgresql.solateam.be is the DNS record generated por the postgreSQL docker running on a VM
			
	6. Then apply to the TAP configuration on TMC the content of tap-values.yaml file under tap-gui.
			
						
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
