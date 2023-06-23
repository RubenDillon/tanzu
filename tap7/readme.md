# Deploying Tanzu Application Platform v1.5.2 on Azure Cloud

### using AKS with Harbor and GitLab using FREE public certificates from Let's Encrypt 

In this tutorial we will be deploying Test and Scan supply chain and GitLab 
authentication and integration.   



Requirements
============

1. An Azure subscription to deploy AKS
3. An AWS subscription for route53 DNS service (we create latamteam.name domain)
4. Tanzu Mission Control
5. TAP 1.5.2 use Kubernetes v1.24, 1.25 and 1.26. We will be using 1.24.10 on a AKS 

## NOTE: 
We will be using lets Encrypt to obtain public free digital certificates.
Using Lets Encrypt has a limitation.  A certificate for the same domain
cannot be requested more than 5 times a week. That's why we must be
careful when we modify the supply chain and install / uninstall
continuously TAP.


Create the DNS records
======================
1. For this environment we will be using DNS Route53 from AWS
    
2. We create a DNS domain called latamteam.name for this environment
    
3. In that environment we will create the following records
	- gitlab.latamteam.name
	- harbor.latamteam.name
	- notary.harbor.latamteam.name
	- tap-gui.latamteam.name
	- *.default.latamteam.name
	- *.learning.latamteam.name
	- pgs.latamteam.name
				

Create the environment
======================

1. Deploy azure client or upgrade it (in my case, Im using a Mac)
```
brew update && brew install azure-cli
```
    
2. Log to Azure 
```
az login
```
    
3. Define your subscription
```
az account set --subscription dddddxxxx-xx-xxxxx-xxxxxxxx
```
    
4. Create the resource group where AKS will be deployed
```
az group create --name LATAM-TAP-RG --location eastus
```
    
5. Verify the kubernetes versions available for AKS
```
az aks get-versions --location eastus
```
    
6. Create the AKS cluster

	to test.............. Standard_D4ds_v4 creates nodes with 4 vCPU and 16 GB RAM

	to have a real use... Standard_D8ds_v5 creates nodes with 8 vCPU and 32 GB RAM

```
az aks create -g LATAM-TAP-RG -n latam-tap-azure --enable-managed-identity --node-count 6 --enable-addons monitoring --enable-msi-auth-for-monitoring --generate-ssh-keys --node-vm-size Standard_D8ds_v5 --kubernetes-version 1.24.10
```

7. Deploy kubectl
```
sudo su
az aks install-cli
```
    
8. Download the kubectl 
```
az aks get-credentials --name latam-tap-azure --resource-group LATAM-TAP-RG
```
    
9. Test the connection
```
kubectl get nodes
```

10. In TMC create a Cluster Group (I create "TAP" Cluster group)

11. Review that you are on the correct cluster
```
kubectl config get-contexts
```
    
12. Attach the AKS Cluster to TMC under TAP cluster group 
	
13. Accept the EULA and deploy the Tanzu cli. Follow this instructions https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.5/tap/install-tanzu-cli.html
    
14. Deploy Cluster Essentials. Follow this instructions https://docs.vmware.com/en/Cluster-Essentials-for-VMware-Tanzu/1.5/cluster-essentials/deploy.html
    	
15. Deploy Cert-Manager
```
tanzu package available list -A
tanzu package available list cert-manager.tanzu.vmware.com -A	
kubectl create ns cert-manager
tanzu package install cert-manager --package cert-manager.tanzu.vmware.com --namespace cert-manager --version 1.10.2+vmware.1-tkg.1
```
NOTE: at this moment we have available for example 1.10.2+vmware.1-tkg.1   
  
16. Deploy Contour 
```    		
tanzu package available list contour.tanzu.vmware.com -A			
kubectl create ns contour
		
tanzu package install contour \
--package contour.tanzu.vmware.com \
--version 1.23.5+vmware.1-tkg.1 \
--values-file contour-values.yaml \
--namespace contour
```
NOTE: at this time we have available for example 1.23.5+vmware.1-tkg.1

17. Obtain the Public IP Address where Envoy is running
```    
kubectl get svc -A | grep envoy
```
      
18. Create the namespace tanzu-system-registry
```       
kubectl create ns tanzu-system-registry
```
        
19. Create a Cluster issuer using the following command
```
kubectl apply -f - <<'EOF'
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-contour-cluster-issuer
  namespace: cert-manager
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
```
    
20. Request the certificate as follow
    
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
          commonName: harbor.latamteam.name
          isCA: false
          privateKey:
            size: 2048
            algorithm: RSA
            encoding: PKCS1
          dnsNames:
          - harbor.latamteam.name
          - notary.harbor.latamteam.name
          issuerRef:
            name: letsencrypt-contour-cluster-issuer
            kind: ClusterIssuer
        EOF
    
21. Now we will expect to wait more or less 10 minutes until we could have the certificate. To verify it use the following command
```
kubectl get certificates -n tanzu-system-registry harbor-cert
```
    
22. When you have TRUE in ready column, you will continue to the next step that is obtain the certificates
```        
kubectl get secret harbor-cert-tls -n tanzu-system-registry -o=jsonpath={.data."tls\.crt"} | base64 --decode > tls-crt.txt            
kubectl get secret harbor-cert-tls -n tanzu-system-registry -o=jsonpath={.data."tls\.key"} | base64 --decode > tls-key.txt
```       
 
23. Deploy Harbor using the Tanzu packages (follow the steps from Tanzu documentation)
```
kubectl create ns harbor
tanzu package install harbor --package harbor.tanzu.vmware.com --version 2.6.3+vmware.1-tkg.1 --values-file harbor-values.yaml --namespace harbor
```                 
    
24. Connect to harbor and create "tap" project as public.
        

Configure the cluster
=
1. We connect to the cluster
            
2. Create the namespace "tap-install"
    
3. Create a internal registry secret
```    
tanzu secret registry add registry-credentials \
--server   harbor.latamteam.name \
--username admin \
--password "PASSw0rd2019202020212022" \
--namespace tap-install \
--export-to-all-namespaces \
--yes
```
    
4. Wait a couple of minutes and review if registry-credentials is in default namespace. 
   If it not, create it for the default namespace.


create the Tanzu Net secret using TMC
=
```
Secret name: tap-registry
Image registry URL: registry.tanzu.vmware.com
Username: <your-tanzu-net-username>
Password: <your-tanzu-net-password>
namespace: tap-install
export to all namespaces
```

Create the repository
=
```
Name: tanzu-tap-repository
Repository URL: registry.tanzu.vmware.com/tanzu-application-platform/tap-packages:1.5.2
```

Review the status of the repository
=
Run the following command
```
tanzu package repository list -A
```

Obtain where is running ENVOY
=
This is needed to register this IP address in the DNS namespaces (harbor.latamteam.name, tap-gui.latamteam.name....)

```
      	kubectl get svc -A | grep envoy 
```  

Configure Lets Encryt for the internal communications
=
Run the cluster-issuer.yaml file
```
	kubectl apply -f cluster-issuer.yaml 
```

Create the Gitlab environment
=

	- https://docs.docker.com/engine/install/ubuntu/
	- https://docs.gitlab.com/ee/install/docker.html

1. Create a Ubuntu 20.04 machine
	
2. Deploy docker on that machine 
```
sudo apt-get update
sudo apt-get install ca-certificates curl gnupg
		
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg	
		
echo \
"deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
"$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
		
sudo apt-get update
		
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```		
		
3. Assign the public IP Address of this machine to the DNS record gitlab.latamteam.name
	
4. Deploy GitLab CE version from docker or the place you select
```
export GITLAB_HOME=/srv/gitlab
export GITLAB_HOME=$HOME/gitlab
		
sudo docker run --detach \
--hostname gitlab.latamteam.name \
--publish 443:443 --publish 80:80  \
--name gitlab \
--restart always \
--volume $GITLAB_HOME/config:/etc/gitlab \
--volume $GITLAB_HOME/logs:/var/log/gitlab \
--volume $GITLAB_HOME/data:/var/opt/gitlab \
--shm-size 256m \
gitlab/gitlab-ee:latest		
```
	
5. Use the following command to obtain the root password
```
sudo docker exec -it gitlab grep 'Password:' /etc/gitlab/initial_root_password
```

6. Adding Lets Encrypt to Gitlab
```
sudo docker exec -it gitlab /bin/bash
```

7. Edit /etc/gitlab/gitlab.rb and add/change the following
```
external_url "https://gitlab.latamteam.name"         # Must use https protocol
letsencrypt['contact_emails'] = ['rdillon@vmware.com'] # Optional
```

8 Reconfigure gitlab
```
gitlab-ctl reconfigure
```

9. Wait a couple of minutes and visit https://gitlab.latamteam.name
		
10. Login to the Gitlab portal
	- On the top bar, select Main menu > Admin.
	- On the left sidebar, select Settings > General.
	- Expand the Visibility and access controls section.
	- Select each of Import sources to allow. At least select github and Repository by URL
	- Select Save changes.
		
11. Import an project example
	- Select "Create a project"
	- Select "Import project"
	- Select "Repository by URL"
	- Add https://github.com/RubenDillon/tanzu-java-web-app in Git repository URL
	- In "Project URL" select root
	- Set "Visibility Level" to Internal 
	- Select "Create Project"


Create a GIT repository 
=
1. Create a New Repository, as public and name it "tap-gitops"
	
2. Using your machine select where you want to clone this repository
```	
git clone http://gitlab.latamteam.name/root/tap-gitops.git 
```

3. Then create a Git repository
```
git init
(optional) git remote add origin git@gitlab.latamteam.name:root/tap-gitops.git
```

Create a file ("prueba.txt") on the directory then summit to the github
```		
git add prueba.txt
git commit -m "Mensaje de confirmación"
git push
```
4. Review that the file were uploaded to tap-gitops
	
5. Connect to Tanzu Networks and go to Tanzu Application Platform. Select tap-gui-catalogs-latest and then the catalog with yelb application
	(Tanzu Application Platform GUI Yelb Catalog)
	
6. Copy at tap-gitops, then restore the file 
```
tar -xvf tap-gui-yelb-catalog.tgz
```		

7. To deply the yelb application to the cluster, use the file yelb-deploy.yaml from this git
```
kubectl apply -f yelb-deploy.yaml
```
		
8. Access the application using your browser http://yelb.latamteam.name
	- Modify your /etc/hosts using your envoy external address
		
9. Upload the catalog
```
git add .
git commit -m "adding catalog"
git branch -M main
git push -u origin main
```
		
10. Review that the file were uploaded to tap-gitops	

 
GitLab authentication
=

1. Create a new public project in Harbor and call it "tap-apps" and "tap-gitops"

2. Create your OAuth App.

	- Redirect URI should point to the auth backend: https://tap-gui.latamteam.name/api/auth/gitlab/handler/frame		
	- The set of permissions granted to the application are: api, read_api, read_user, read_repository, write_repository, openid, and email.
		
3. Generate a new Client Secret and take a note of the Client ID and the Client Secret
	
4. Modify the tap-gui part of the tap-values-OOTB-basic.yaml file to add ClientId and clientSecret values
	- where ClientID is obtained from GitLab Apps (Developer settings) and ClientSecret.. is the client secret generated for that App in GitLab		
5. Create a new personal Token on GitLab 
	
6. Create a Secret on the default namespace as git-secret.yaml (from this github). Use the token as password and modify the user name.
	
7. Apply the secret


GitOps integration
=

1. Update (patch) the default service account with the new secret
```	
kubectl patch serviceaccount default --patch '{"secrets": [{"name": "github-http-secret"}]}'		
kubectl patch serviceaccount default --patch '{"secrets": [{"name": "registry-credentials"}]}'
```				

2. Review the default service account
```	
kubectl describe serviceaccount default		
```


Install Tanzu Application Platform package from TMC Catalog
=

1. Select the Tanzu Application Platform
    
2. Select tap as name and 1.5.2 for the version
    
3. Copy the tap-values-OOTB-basic on the TMC UI
    
4. Monitor the install by running
```            
tanzu package installed get tap -n tap-xxxxxx
```
            
5. Verify all packages are successfully reconciled
```            
tanzu package installed list -A
```
          
6. The UI (tap-gui) receive a public certificate from lets encrypt. If you want to review it 
```    
kubectl get secret -n tap-gui tap-gui-cert -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -text	    
```  

Prepare the developer namespace
=
1. Create a namespace using kubectl (if you change the namespace will be used for developers)
```
kubectl create namespace xxxy (we are using default namespace.. for that we don't need to create it)
```

2. Label the namespace
```
kubectl label namespaces default apps.tanzu.vmware.com/tap-ns=""
```
	
3. Review the configuration 
```
kubectl get secrets,serviceaccount,rolebinding,pods,workload,configmap,limitrange -n default
```

Test an Example in a basic Supply Chain
=	
1. Now we will be deploying the example using that repository
```	
tanzu apps workload apply tanzu-java-web-app \
--git-repo https://gitlab.latamteam.name/root/tanzu-java-web-app \
--git-branch main \
--type web \
--app tanzu-java-web-app \
--tail \
--yes
```
	  
2. View the build 
```            
kubectl get workload,gitrepository,pipelinerun,images.kpack,podintent,app,services.serving
```
    
3. After ends you could access the TAP GUI to see the process and review the app using the following command
```    
tanzu apps workload get tanzu-java-web-app --namespace default
```
            
4. The application will be in the Catalog of the TAP-GUI UI (because TAP discovers where catalog files are)            	


Modify the supply chain to use Testing and Scanning
=

 - https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.5/tap/getting-started-add-test-and-security.html
 - https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.5/tap/namespace-provisioner-ootb-supply-chain.html#testing--scanning-supply-chain-3 
 
1. Modify the tap-values-ootb-test-scan-auth.yaml from this github

2. Apply this file into TAP package using the TMC Catalog. If you decide uninstall TAP and then use this tap-values to deploy again,
		remember to add all the previous steps like patch service account and anything that already we did.
       
3. Create the Scan policy applying the scan-policy-free.yaml and scan-template.yaml
```        
kubectl apply -f scan-policy-free.yaml                
kubectl apply -f scan-template.yaml
```		
Because we are not using the default scan-policy we apply this one that creates scan-policy-free policy, 
and we define their use in tap-values

       
4. Create a Tekton pipeline 
```     
kubectl apply -f pipeline.yaml      	 
kubectl get pipeline.tekton.dev,scanpolicies
```
	        
5. Delete the example application
```
tanzu apps workload delete tanzu-java-web-app -y    
```        	
        
Test an Example in a Testing and Scanning supply chain
=
	
1. Now we will be deploying the example using that repository
```	
tanzu apps workload apply tanzu-java-web-app \
--git-repo https://gitlab.latamteam.name/root/tanzu-java-web-app \
--git-branch main \
--type web \
--app tanzu-java-web-app \
--label apps.tanzu.vmware.com/has-tests="true" \
--param-yaml testing_pipeline_matching_labels='{"apps.tanzu.vmware.com/language": "java"}' \
--tail \
--yes
```
	
2. For scan policy, we already define a default in our configuration (tap-values...yaml), but if we dont do that.. 
the default policy (scan-policy) is very restrictive and we need to define which one.. we want to use..
	
	- --param scanning_source_policy="lax-scan-policy" \
	- --param scanning_image_policy="lax-scan-policy" \

3. View the build 
```            
kubectl get workload,gitrepository,pipelinerun,images.kpack,podintent,app,services.serving
```
    
4. After ends you could access the TAP GUI to see the process and review the app using the following command
```    
tanzu apps workload get tanzu-java-web-app --namespace default
```

5. Review the deliverables of the deployment
```	 
kubectl get deliverables		
kubectl get deliverable tanzu-java-web-app -o yaml > deliverable-tanzu-java-web-app.yaml
```

6. Review the Knative service of the deployment
```		
kubectl get service.serving.knative.dev/tanzu-java-web-app
```

Use KubeNEAT to generate a readable delivery file 
=

1. Deploy KubeNEAT
```
wget -O - https://github.com/itaysk/kubectl-neat/releases/download/v2.0.3/kubectl-neat_linux_amd64.tar.gz | \
sudo tar -C /usr/local/bin -zxvf - kubectl-neat
```

2. Use it to review a better readable file
```
kubectl-neat < deliverable-tanzu-java-web-app.yaml > deliverable-limpio.yaml
```
3. You could use that file to deploy the application into another cluster. For example, this environment will be for
development and another cluster (run cluster) for production. In that cluster, we could deploy with this deliverable file.



Review the Self-Guided Workshop
=

1. run the following command to review the activated portals
```
kubectl get trainingportals
```

2. then connect to the defined URL http://learning-center-guided.learning.latamteam.name

NOTE: This is a good example on how to build a workshop. Only one thing that we need to consider
      	about this particular example. Everything works, except when we try to reach the registry
	inside the workshop. This is a limitation on containerd when we use http, as we are using
	with this deployment of Learning Center. This is not the case when we use a secure
	connection using https.

- https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.5/tap/learning-center-getting-started-learning-center-operator.html	 

Deploy the different Learning Portals
=

1. Run the following command
```
kubectl apply -f learning/training-portal.yaml            
kubectl apply -f learning/portal.yaml
```
    
2. Review the deployment
```    
kubectl get workshops
```

3. To see what we have created
```     
kubectl get learningcenter-training -o name
```
     
4. To see the sessions created
```     
kubectl get workshopsessions
```            

5. To found the portals information and admin users use the following
```     
kubectl get trainingportals
```            
            
6. To deploy a Spring Boot workshop, go to /learning on this git      
```            
kubectl apply -f learning/spring-workshop.yaml 
kubectl apply -f learning/spring-portal
kubectl get trainingportals
```            
            
7. Use another example... learning-center-workshop-samples/lab-markdown-sample
```     
kubectl apply -f earning-center-workshop-samples/lab-markdown-sample/resources/workshop.yaml            
kubectl apply -f learning-center-workshop-samples/lab-markdown-sample/resources/training-portal.yaml            
kubectl get trainingportals
```     

8. With this we will have four Learning portals
    
            - lab-k8s-fundamentals     http://lab-k8s-fundamentals-ui.learning.latamteam.name
            - lab-markdown-sample      http://lab-markdown-sample-ui.learning.latamteam.name
            - lab-spring-boot-k8s-gs   http://lab-spring-boot-k8s-gs-ui.learning.slatamteam.name
            - learning-center-guided   http://learning-center-guided.learning.latamteam.name


Configure Pull Request (PR) for the supply chain
=

1. Modify the tap-values-ootb_test_Scan-auth-PR.yaml and copy in the TAP package to add Pull Request.
If you decide uninstall TAP and then use this tap-values to deploy again,
remember to add all the previous steps like patch service account and anything that already we did.
 
2. Delete the tanzu-java-web-app and deploy it again
```
tanzu apps workload delete tanzu-java-web-app --namespace default -y
```	

3. Deploy tanzu-java-web-app again
```	
tanzu apps workload apply tanzu-java-web-app \
--git-repo https://gitlab.latamteam.name/root/tanzu-java-web-app \
--git-branch main \
--type web \
--app tanzu-java-web-app \
--label apps.tanzu.vmware.com/has-tests="true" \
--param-yaml testing_pipeline_matching_labels='{"apps.tanzu.vmware.com/language": "java"}' \
--tail \
--yes
```	

4. When the Supply Chain reach the Config Writer state, you will have a "Approve" button. 

5. That button direct you to the gitlab merge request 

6 With that, the supply chain continues and deploy the application in a couple of minutes


Configure VS Code
=

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
     - Source Image: harbor.latamteam.name/tap-apps/tanzu-java-web-app
     - Namespace: default.   
   
10. Do the same with the "Tanzu App Accelerator Extension for Visual Studio Code" 	
11. Once deployed configure tap-gui.latamteam.name as the TAP GUI URL


## Using VS Code to deploy and iterate the example

```
            
         1. Open VS Code and open the TANZU-JAVA-WEB-APP folder from the cloned gitlab resource

	 2. Do a docker login against harbor

		docker login harbor.latamteam.name -u admin
         
         3. Review the following files
         
                config/workload.yaml
                catalog/catalog-info.yaml
                tiltfile
		
	 4. Probably (if you use another name to your cluster) you will need to add a first line providing the name of the cluster
	 
	 	allow_k8s_contexts('<name of your cluster>')
	 
	 5. Remember that you probably need to add the label for pipeline (needs to looks like the following)

		" --label apps.tanzu.vmware.com/has-tests=true " +
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
			Owner: root
			Repository Name: tanzu-java-web-app2
			Repository Branch: main
			
	    10. Open the app and modify the tiltfile to include the following
	    
			allow_k8s_contexts('<your cluster>')

               		" --label apps.tanzu.vmware.com/has-tests=true " +
               		" --param-yaml testing_pipeline_matching_labels='{"+"apps.tanzu.vmware.com/language"+": "+"java"+"}' " + 
	    
	    11. Use the LiveView to deploy it into TAP... wait until you need to Approve the Request... and finally see the deployment.
	
   
```


## Try another application as example
```
        A Weather application using Steeltoe framework (.NET core)

	Use the VSCode to create this example from the Application Accelerator and push to gitlab...

	Then you could use VS Code or run it from terminal using the following command
	    
	    tanzu apps workload create weatherforecast-steeltoe \
            --git-repo https://gitlab.latamteam.name/root/weatherforecast-steeltoe \
            --git-branch main \
            --type web \
            --label app.kubernetes.io/part-of=weatherforecast-steeltoe \
            --label apps.tanzu.vmware.com/has-tests=true \
            --yes --tail\
            --namespace default
	    
	Automatically this application will be in the Application Catalog from TAP-GUI 
            
	  

 	Another Java application 

	Use the VSCode to create this example from the Application Accelerator and push to gitlab.

	Then you could use VS Code or run it from terminal using the following command
	  
	    tanzu apps workload create java-server-side-ui \
            --git-repo https://gitlab.latamteam.name/root/tap-gitops \
            --sub-path java-server-side-ui \
            --git-branch main \
            --type web \
            --label app.kubernetes.io/part-of=java-server-side-ui \
	    --param-yaml testing_pipeline_matching_labels='{"apps.tanzu.vmware.com/language": "java"}' \
	    --label apps.tanzu.vmware.com/has-tests=true \
            --yes --tail \
            --namespace default
	    
	Automatically this application will be in the Application Catalog from TAP-GUI 


	    
	An Angular front end application

	Use the VSCode to create this example from the Application Accelerator and push to gitlab

	Then you could use VS Code or run it from terminal using the following command

	    tanzu apps workload create angular-frontend \
            --git-repo https://gitlab.latamteam.name/root/tap-gitops \
            --sub-path angular-frontend \
            --git-branch main \
            --type web \
            --label app.kubernetes.io/part-of=angular-frontend \
	    --label apps.tanzu.vmware.com/has-tests=true \
            --yes --tail \
            --namespace default
	    
	   Automatically this application will be in the Application Catalog from TAP-GUI 
	    
	    
	    
	A Node.js (using express.js) example

	Use the VSCode to create this example from the Application Accelerator and push to gitlab

	Then you could use VS Code or run it from terminal using the following command
	    
	    tanzu apps workload create node-express \
            --git-repo https://gitlab.latamteam.be/root/tap-gitops \
            --sub-path node-express \
            --git-branch main \
            --type web \
            --label app.kubernetes.io/part-of=node-express \
	    --label apps.tanzu.vmware.com/has-tests=true \
            --yes --tail \
            --namespace default
	    
	    Automatically this application will be in the Application Catalog from TAP-GUI 


	    Another (but.. this is from an image)
	    
	    tanzu apps workload create simple-web-app \
            --image ghcr.io/vmware-tanzu-learning/simple-web-app:v1.1.0 \
	    --label apps.tanzu.vmware.com/has-tests=true \
            --type web \
            --yes
	    
	    curl -k https://simple-web-app.default.tanzulatam.name/hello
	    
	
```


## Steps to create a TAP workload from an existing application 
```
        We will deploy a Hello World application based on .NET that is available in Azure examples
        
            tanzu apps workload create hello-world \
            --git-repo https://github.com/Azure-Samples/dotnetcore-docs-hello-world \
            --git-branch master \
            --type web \
            --label app.kubernetes.io/part-of=hello-world \
            --label apps.tanzu.vmware.com/has-tests=true \
            --yes \
            --tail \
            --namespace default

	Usng VS Code you could use snippet to create the workload.yaml and the catalog-info.yaml

	Open a new file and write tanzu and wait the response of VS Code

	VS Code will suggest a snippets
		
	
	Another example... using Azure AI
	
	tanzu apps workload create chatgpt \
            --git-repo https://github.com/Azure-Samples/chatgpt-quickstart \
            --git-branch main \
            --type web \
            --label app.kubernetes.io/part-of=chatgpt \
            --label apps.tanzu.vmware.com/has-tests=true \
            --yes \
            --tail \
            --namespace default
	    
	Approve and confirm the Merge and wait until the application is running
	
	To use the application follow the instructions from https://github.com/Azure-Samples/chatgpt-quickstart	

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
	
	1. To provide a persistent storage for the TAP-GUI we need a PostgreSQL database. For that reason we create another instance 
	
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
			
	5. Test the connection from another ubuntu machine
		sudo apt-get install postgresql-client
		
	    Test from a Mac machine
	    	brew doctor
		brew update
		brew install libpq		
		brew link --force libpq
		
	     Connect to the PostgreSQL
		
		psql -U postgres -p 5432 -h pgs.latamteam.name
			where pgs.latamteam.name is the DNS record generated por the postgreSQL docker running on a VM
			
	6. Then apply to the TAP configuration on TMC the content of the following under tap-gui
    backend:
      database:
        client: pg
        connection:
          host: pgs.latamteam.name
          password: secreto
          port: 5432
          ssl:
            rejectUnauthorized: false
          user: postgres
        pluginDivisionMode: database
	    
	    
	 7. Stop the tap-gui (or all Kubernetes service) and start it again
	 
	 8. In the PostgreSQL you will see a couple of new databases. Those are used by TAP-GUI.
	 
	 	backstage_plugin_auth
		backstage_plugin_catalog
		backstage_plugin_pendo-analytics
		backstage_plugin_scaffolder
		backstage_plugin_search
		backstage_plugin_user-settings

						
```

## API Portal
```
	1. We will be creatig an example

kubectl apply -f - <<EOF
apiVersion: apis.apps.tanzu.vmware.com/v1alpha1
kind: APIDescriptor
metadata:
  name: petstore3
spec:
  type: openapi
  description: A sample APIDescriptor to validate package installation successful
  system: test-installation
  owner: test-installation
  location:
    path: "/api/v3/openapi.json"
    baseURL:
      url: https://petstore3.swagger.io
EOF

        2. Verify that the APIDescriptor status shows Ready:
		kubectl get apidescriptors
			 
	3. Go to the TAP-GUI to the API Portal and you will see this example
		
	4. If we already installed steeltoe weather forecast application, we could add that application to API Portal
			 
kubectl apply -f - <<EOF                                                                                        
apiVersion: apis.apps.tanzu.vmware.com/v1alpha1
kind: APIDescriptor
metadata:
  name: weather-forecast-steeltoe
spec:
  type: openapi
  description: A sample APIDescriptor to validate package installation successful
  system: test-installation
  owner: default-team
  location:
    path: "/v1/swagger.json"
    baseURL:
      url: https://weatherforecast-steeltoe.default.solateam.be/swagger
EOF			 

			
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

## Troubleshooting Certificates error
```
	- https://letsencrypt.org/docs/duplicate-certificate-limit/

	We need to be careful about installing and uninstalling TAP components that require digital certificates.
	Lets Encrypt will revoke new certificates when it detects this situation.

	You probably will have problems to connect TAP-GUI interface and if review the certificates, we will have
	see in False the READY column. To review the state of secrets, run the following

		kubectl get certificates -A

	For example, if we want to see what happen with tap-gui-cert, run the following

		kubectl describe certificates tap-gui-cert -n tap-gui

	Yo will see something like that...

		The certificate request has failed to complete and will be retried: Failed to wait
		for order resource "tap-gui-cert-7r56d-3305844173" to become ready: order is in "errored"
		state: Failed to create Order: 429 urn:ietf:params:acme:error:rateLimited: Error creating
		new order :: too many certificates (5) already issued for this exact set of domains in
		the last 168 hours: tap-gui.latamteam.name, retry after 2023-06-24T20:24:42Z:
		see https://letsencrypt.org/docs/duplicate-certificate-limit/

	We will need until the time exposed by the log


```


## Troubleshooting “Builder default is not ready” message

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
			
## Troubleshooting error in Config Writer (Supply Chain)

```
	1. When you deploy an application and have an error in the Config Writer step,  and the log tell you something like user 
	   don't found when TAP tries to write a commit to github.com
			
	2. You need to review the following
		- The default account have the secrets needed (patch it)
		- the secret have the correct github token
		- the values for the TAP deployment have the right github token and OAuth information
			
```

## Troubleshoot TAP-GUI update configure in TAP-VALUES
```
	1. Run the following command to renew the tap-gui pod
		kubectl rollout restart -n tap-gui deployment server

	or delete the backstage pods
		kubectl delete pod -l app=backstage -n tap-gui

```

## Troubleshoot namespace Terminating
```

    For example the ns tanzu-system-service-discovery is in Terminating status...
    
    kubectl get namespace tanzu-system-service-discovery -o json \
  | tr -d "\n" | sed "s/\"finalizers\": \[[^]]\+\]/\"finalizers\": []/" \
  | kubectl replace --raw /api/v1/namespaces/tanzu-system-service-discovery/finalize -f -
    
    
```
	
	
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
