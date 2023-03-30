### Instalar y configurar el cliente de Azure 
https://docs.microsoft.com/en-us/cli/azure/install-azure-cli-linux?pivots=apt 

sudo curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash

### Crear la aplicación TKG en Azure 
https://tanzucommunityedition.io/docs/v0.11/azure-mgmt/ 

tenant ID:  xxxx
Suscription ID: yyyy
App (client) ID:  zzzz
Secret: (value) : yyyy
(secret ID) : xxxx

SSH Key:
ssh-rsa AAAAB3NzaC1yc2EAAAA.......30jezCw== “rdillon@vmware.com”

### steps
1.	Log in to the Azure Portal.
2.	Record your Tenant ID by hovering over your account name at upper-right, or else browse to Azure Active Directory > <Your Azure Org> > Properties > Tenant ID. The value is a GUID, for example b39138ca-3cee-4b4a-a4d6-cd83d9dd62f0.
3.	Browse to Active Directory > App registrations and click + New registration.
4.	Enter a display name for the app, such as tce, and select who else can use it. You can leave the Redirect URI (optional) field blank.
5.	Click Register. This registers the application with an Microsoft Azure service principal account.
6.	An overview pane for the app appears. Record its Application (client) ID value, which is a GUID.
7.	From the Microsoft Azure Portal, browse to Subscriptions. At the bottom of the pane, select one of the subscriptions you have access to, and record its Subscription ID. Click the subscription listing to open its overview pane.
8.	Select to Access control (IAM) and click Add a role assignment.
9.	In the Add role assignment pane
    -	Select the Owner role
    -	Leave Assign access to selection as “Azure AD user, group, or service principal”
    -	Under Select enter the name of your app, tce. It appears underneath under Selected Members
10.	Click Save. A popup appears confirming that your app was added as an owner for your subscription.
11.	From the Microsoft Azure Portal > Azure Active Directory > App Registrations, select your tce app under Owned applications. The app overview pane opens.
12.	From Certificates & secrets > Client secrets click + New client secret.
13.	In the Add a client secret popup, enter a Description, choose an expiration period, and click Add.
14.	The new secret is listed with its generated value under Client Secrets. Record the value.

### Accept the base image

https://docs.vmware.com/en/VMware-Tanzu-Kubernetes-Grid/1.5/vmware-tanzu-kubernetes-grid-15/GUID-mgmt-clusters-azure.html#tkg-app 

1.	Sign in to the Azure CLI as your tce client application.
az login --service-principal --username AZURE_CLIENT_ID --password AZURE_CLIENT_SECRET --tenant AZURE_TENANT_ID
Where AZURE_CLIENT_ID, AZURE_CLIENT_SECRET, and AZURE_TENANT_ID are your tce app’s client ID and secret and your tenant ID, 
2.	Run the az vm image terms accept command, specifying the --plan and your Subscription ID.
In Tanzu Kubernetes Grid v.1.5.1, the default cluster image --plan value is k8s-1dot22dot5-ubuntu-2004, based on Kubernetes version 1.22.5 and the machine OS, Ubuntu 20.04. 

az vm image terms accept --publisher vmware-inc --offer tkg-capi --plan k8s-1dot22dot5-ubuntu-2004 --subscription AZURE_SUBSCRIPTION_ID

Where AZURE_SUBSCRIPTION_ID is your Azure subscription ID.
You must repeat this to accept the base image license for every version of Kubernetes or OS that you want to use when you deploy clusters, and every time that you upgrade to a new version of Tanzu.
