# Deploy Tanzu Application Platform v1.3.5 using Azure with Harbor (using FREE public certificates)

## Requirements
```
    1. An Azure subscription to deploy TKG
    2. A management cluster deployed on Azure
    3. An AWS subscription for route53 DNS service (we create solateam.be domain)
    4. Tanzu Mission Control
    5. To deploy 
        - TAP 1.3.5 we need Kubernetes v1.22, v1.23 or v1.24. We will be using 1.22.5 on a TKG deployed on Azure
```

## Create the environment
```
    1. Join the Management Cluster to TMC. 
    2. Create a Cluster Group (I create "TAP" cluster group)
    3. Deploy a TKG cluster 
    
            1 instance for Control plane (Standard_F8s_v2 = 8 vCPU and 16 GB RAM each)
            5 instances for Workers nodes (Standard_F8s_v2 = 8 vCPU and 16 GB RAM each)
    
    4. Deploy Cert-Manager using the defaults
    5. Deploy Contour using the contour-values.yaml from this git
    6. Create the namespace tanzu-system-service-discovery 
    
            kubectl create ns tanzu-system-service-discovery
    
    7. Create un secreto con las credenciales de AWS
            
            kubectl create secret generic route53-credentials --from-literal=aws_access_key_id=<AWS_ACCESS_KEY_ID> --from-literal=aws_secret_access_key=<AWS_SECRET_ACCESS_KEY> -n tanzu-system-service-discovery
            
            for example
            
            kubectl create secret generic route53-credentials --from-literal=aws_access_key_id=ASIA3... --from-literal=aws_secret_access_key=VmM0EpFiTZM8RiNEWm.... -n tanzu-system-service-discovery
            
    8. Deploy DNS-external from TMC Catalog using the DNS-external-values.yaml from this git. Modify the file as needed.
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
        
    14. Deploy Harbor from the TMC Catalog using harbor-values.yaml from this git. Modify the fields whith yours certificate.
    
    15. Connect to harbor and create "tap" project as public.
            
```      

## Configure the cluster
```
    1. We connect to the cluster and run allow-run-as.yaml from the kubernetes/TKGs git 
    
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
        Repository URL: registry.tanzu.vmware.com/tanzu-application-platform/tap-packages:1.3.5
```

## Obtain where is running ENVOY
```
      In our example we assume that is running on the following IP Address: 20.246.129.194
```      

## Install Tanzu Application Platform package from TMC Catalog
```
    1. Select the Tanzu Application Platform
    2. Select tap as name and 1.3.5 for the version
    3. Copy the tap-values.yaml from this git. Modify the file as needed.
    4. Monitor the install by running
            
            tanzu package installed get tap -n tap-install
            
    5. Verify all packages are successfully reconciled
            
            tanzu package installed list -A
          
```  

## Review the Self-Guided Workshop
```
    1. run the following command to review the activated portals
            kubectl get trainingportals
    2. then connect to the defined URL
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
# Deploy an example

```

    1. Deploy the application using an example from github
    
            tanzu apps workload create tanzu-java-web-app \
            --git-repo https://github.com/vmware-tanzu/application-accelerator-samples \
            --sub-path tanzu-java-web-app \
            --git-branch main \
            --type web \
            --label app.kubernetes.io/part-of=tanzu-java-web-app \
            --yes \
            --namespace default
            
            
    2. View the build and runtime logs 
    
            tanzu apps workload tail tanzu-java-web-app --since 10m --timestamp --namespace default
    
    3. After ends you could access the TAP GUI to see the process and using the following command
    
            tanzu apps workload get tanzu-java-web-app --namespace default
            
    4. To register the application, go to the tap-gui.solateam.be and click "Register Entity"
    
            input: https://github.com/vmware-tanzu/application-accelerator-samples/blob/main/tanzu-java-web-app/catalog/catalog-info.yaml
    
    
    5. Iterate with the application (To be defined...)
    
```


  Basado en 
  - https://confluence.eng.vmware.com/display/CNA/TAP+-+How+to+install+TAP+using+TMC
  - https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.0/tap/GUID-install.html
  - https://confluence.eng.vmware.com/pages/viewpage.action?spaceKey=CNA&title=Installing+Harbor+with+LE+certs+via+TMC

