# F5 with K8s Demo

## Useful URLs
- [CIS Using ClusterIP Mode](https://clouddocs.f5.com/training/community/containers/html/class1/module2/module2.html)
- [CIS and AS3 Extension integration](https://clouddocs.f5.com/containers/v2/kubernetes/kctlr-k8s-as3.html)
- [Routes](https://clouddocs.f5.com/containers/latest/userguide/routes.html)
- [Install F5 Container ingress services](https://devcentral.f5.com/s/articles/CIS-and-Kubernetes-Part-2-Install-F5-Container-ingress-services)

## Demo Notes
Based on the UDF Demo -> ASE CC K8s Exercise

## Gernal Notes
K8s Master

```shell
# Check the system status
$ systemctl status kubelet
$ kubectl get node
```

## Troubleshooting
- [CIS documentation](https://clouddocs.f5.com/containers/v2/troubleshooting/kubernetes.html)

1. Find the name of the k8s-bigip-ctlr Pod.
    ```bash
    kubectl get pod --namespace=kube-system
    ```
2. Check the status of the Pod.
    ```bash
    kubectl get pod k8s-bigip-ctlr-687734628-7fdds -o yaml --namespace=kube-system
    ```
3. View the Controller logs.
    ```bash
    kubectl logs k8s-bigip-ctlr-687734628-7fdds --namespace=kube-system
    # tail -f  
    kubectl logs -f k8s-bigip-ctlr-687734628-7fdds --namespace=kube-system
    ```

    View logs for a container that isn’t responding:
    ```bash
    kubectl logs --previous k8s-bigip-ctlr-687734628-7fdds --namespace=kube-system
    ```
  
4. To change the log level for the BIG-IP Controller:
  - Edit the Deployment yaml and add the following to the args section: `"--log-level=DEBUG"`
  - Replace the BIG-IP Controller deployment: `kubectl replace -f f5-k8s-bigip-ctlr.yaml`
  - Verify the Deployment updated successfully: `kubectl describe deployment k8s-bigip-ctlr -o wide --namespace=kube-system`

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

#### Expose Service via ClusterIP

1. Create/expose a service with the following command
    ```shell
    $ kubectl expose deployment/exercise1 --name exercise1-svc --port=80
    ```

2. Investigate the service you just created in the dashboard. Under the services section click on the “exercise1-svc” name. By default Kubernetes has created a service of serviceType _ClusterIP_

3. Verify the DNS entry that kubedns created for your service. When pods are created they are able to resolve against kubedns by default.
    ```shell
    $ kubectl get pod
    $ kubectl exec -it [name of pod] -- nslookup exercise1-svc
    ```

#### Expose Service via NodePort

1. Create a new deployment (so we can more easily differentiate our test applications):
    ```shell
    $ kubectl create -f ./f5-cis/2-exercise2.yml
    ```

2. Create a new service using _NodePort_
    ```shell
    $ kubectl expose deployment/exercise2 --name exercise2-svc --type=NodePort
    ```

3. Determine what port(s) the service is exposed on:
    ```shell    
    $ kubectl get service exercise2-svc
    ```

#### Deploy F5 Container Ingress Services

1. Under _Systems -> Users -> Partitions_ create partition _kubernetes_

2. Create a Secret for BIG-IP. The user credentials for the BIG-IP need to be stored in a Kubernetes secret. Run the commands in the following section from the bash prompt on kmaster.
    ```shell
    $ kubectl create secret generic bigip-login --namespace kube-system --from-literal=username=admin --from-literal=password=admin
    ```

3. Service Account. You will also need a Service Account that the controller will run as.
    ```shell
    $ kubectl apply -f ./f5-cis/cis-sa.yaml -n kube-system
    ```

4. RBAC Permissions. Grant appropriate permissions to the Service Account.
    ```shell
    $ kubectl apply -f ./f5-cis/cis-rbac.yaml -n kube-system
    ```

5. Deploy the Controller. The following command will deploy the controller.
    ```shell
    $ kubectl apply -f ./f5-cis/f5-cc-deployment.yaml -n kube-system
    ```

6. You can verify the controller is running from the dashboard or using kubectl from bash on kmaster. Note this pod is running in the “kube-system” namespace (include -n kube-system if using kubectl).
    ```shell
    $ kubectl -n kube-system get pods
    ```

7. Create a new Deployment/Application
    ```shell
    $ kubectl create -f ./f5-cis/3-exercise3.yml
    ```

8. Create a Service
    ```shell
    $ kubectl create -f ./f5-cis/3-exercise3-svc.yml
    ```

9. Basic L4 Services
    ```shell
    $ kubectl create -f ./f5-cis/4-as3-basic-configmap.yaml
    ```

10. Deploying a WAF Policy:
- show that there is preconfigured security policy in ASM, which we are referring to
    ```shell
    $ kubectl apply -f ./f5-cis/5-waf-configmap.yaml
    ```

11. Execute the following command from kmaster:
    ```shell
    $ curl http://app.example.com/ -v -H "X-Hacker: cat /etc/paswd"
    ```

12. On the BIG-IP go to Security -> Event Logs and you should see the blocked request

13. Scale the deployment
- change the _replicas_ parameter to 6 and redeploy
- check the _LTM_ Pool chnages after each modification and deployment
    ```shell
    $ kubectl apply -f ./f5-cis/6-scale.yml
    ```

