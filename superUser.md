

In order to login to Supervisor Control Plane VM, you’ll need root password that can be obtained from vCenter. 
So login to the vCenter server with your root account, switch your shell and run this little python script. 
It connects to the PSQL, queries the database and provides you the root password of the Supervisor VM as well as the VIP of the K8s api-server.

/usr/lib/vmware-wcp/decryptK8Pwd.py

<img width="1422" alt="image" src="https://user-images.githubusercontent.com/26331064/188155966-6b21dc8d-b97d-4952-a391-f069023cd1da.png">


Next step is to SSH to the IP address with the human-unreadable password. 
After that point, you can easily demonstrate that you have cluster-admin privileges and list/modify system level resources as well 
as all the custom resources that comes with Tanzu.

<img width="1468" alt="image" src="https://user-images.githubusercontent.com/26331064/188156001-4fe5358d-0655-45d8-838d-fd69492ee874.png">


https://orcunuso.wordpress.com/2021/03/30/tanzu-supervisor-cluster-with-cluster-admin-privileges/
