# F5 with K8n Demo

Based on the UDF Demo -> ASE CC K8s Exercise

K8n Master

        systemctl status kubelet

## Demo

### Dashboard

1. Open the Kubernetes dashboard
2. Navigate to the deployments menu option
3. Click the _+Create_ button in the upper right hand corner of the dashboard
4. Click on the tab labelled Create an App. Fill out the fields in the following manner:
    - App Name:	exercise1
    - Container Image: f5devcentral/f5-demo-httpd
    - Number of Pods: 2
    - Service: None
5. Using the dashboard investigate the deployment you just made
    - Look at the pods menu item. Which nodes were the pods deployed on?
    - Look at the deployments menu item again. Notice there is now a deployment.
    - Click on the three dots to the right of your deployment. From the flyout menu select View/edit YAML.
6. Delete the existing deployment by choosing Delete from the previous flyout menu
7. Recreate the same deployment using YAML (/f5-cis/1-f5demo-app.yml). Click the + Create button. Verify the Create from Text Input tab is selected. Paste the following YAML and then click Upload

### CLI
#### Exp
1. Create/expose a service with the following command

        $ kubectl expose deployment/exercise1 --name exercise1-svc --port=80

2. Investigate the service you just created in the dashboard. Under the services section click on the “exercise1-svc” name. By default Kubernetes has created a service of serviceType _ClusterIP_

3. Verify the DNS entry that kubedns created for your service. When pods are created they are able to resolve against kubedns by default.

        $ kubectl get pod
        $ kubectl exec -it [name of pod] -- nslookup exercise1-svc

#### Expose Service via NodePort
1. Create a new deployment (so we can more easily differentiate our test applications):

        $ kubectl create -f ./f5-cis/2-exercise2.yml

2. Create a new service using _NodePort_

        $ kubectl expose deployment/exercise2 --name exercise2-svc --type=NodePort

3. Determine what port(s) the service is exposed on:
    
        $ kubectl get service exercise2-svc

#### Deploy F5 Container Ingress Services
1. Under _Systems -> Users -> Partitions_ create partition _kubernetes_
2. Create a Secret for BIG-IP. The user credentials for the BIG-IP need to be stored in a Kubernetes secret. Run the commands in the following section from the bash prompt on kmaster.

        $ kubectl create secret generic bigip-login --namespace kube-system --from-literal=username=admin --from-literal=password=admin

3. Service Account. You will also need a Service Account that the controller will run as.

        $ kubectl apply -f ./f5-cis/cis-sa.yaml -n kube-system

4. RBAC Permissions. Grant appropriate permissions to the Service Account.

        $ kubectl apply -f ./f5-cis/cis-rbac.yaml -n kube-system

5. Deploy the Controller. The following command will deploy the controller.

        $ kubectl apply -f ./f5-cis/f5-cc-deployment.yaml -n kube-system

6. You can verify the controller is running from the dashboard or using kubectl from bash on kmaster. Note this pod is running in the “kube-system” namespace (include -n kube-system if using kubectl).

7. Create a new Deployment/Application

        $ kubectl create -f ./f5-cis/3-exercise3.yml

8. Create a Service

        $ kubectl create -f ./f5-cis/3-exercise3-svc.yml

9. Basic L4 Services

        $ kubectl create -f ./f5-cis/4-as3-basic-configmap.yaml

10. Deploying a WAF Policy:

- show that there is preconfigured security policy in ASM, which we are reffering to

        $ kubectl apply -f ./f5-cis/5-waf-configmap.yaml

11. Execute the following command from kmaster:

        $ curl http://app.example.com/ -v -H "X-Hacker: cat /etc/paswd"

12. On the BIG-IP go to Security -> Event Logs and you should see the blocked request

13. Scale the deployment

        $ kubectl apply -f ./f5-cis/6-scale.yml

- change the _replicas_ parameter to 6 and redeploy
- check the _LTM_ Pool chnages after each modification and deployment