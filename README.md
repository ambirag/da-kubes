# da-kubes
Selenium Grid based Distributed Automation using Kubernetes, Docker, Traefik and Helm

Presented in Selenium Camp, Kiev in 2019 Feb - https://www.youtube.com/watch?v=MmdGyqNqs9w

## URLs
Once you have successfully setup da-kube in AWS EKS, and if  your dns_suffix is ".hub.test.yourcompany.com"
Then the following urls should work:

[Traefik URL] (https://traefik-web-ui.hub.test.yourcompany.com)

[Dashboard URL] (https://dashboard.hub.test.yourcompany.com)

[Any hubs you create] (https://<namespace_of_the_hub>.hub.test.yourcompany.com/grid/console)

## Install
Kubernetes version of Distributed Automation

[Install Minikube] (https://kubernetes.io/docs/setup/minikube/)

[Install Kubectl] (https://kubernetes.io/docs/tasks/tools/install-kubectl/)

[DA Kubes] (https://ewegithub.sb.karmalab.net/EWE/da-kubes)

```
$ brew install kubernetes-helm
$ helm init --upgrade
```

## Local Setup

```
$ git clone https://github.com/ambirag/da-kubes
$ cd da-kubes
$ helm install selenium --name da
$ helm list
$ helm status da
$ minikube service da-selenium-hub --url -n da
http://192.168.99.100:30456/grid/console (ClusterIP:NodePort)

```
### Need another hub setup?
```
$ minikube service da-selenium-hub --url -n da
```

## Tear Down

```
helm list
helm del --purge da
```

# AWS Setup

## Create Route53 Record
Create a Route53 record for ".hub.test.yourcompany.com"

## Create CloudFormation stack
- Update cloudformation.yaml with at-least two subnets (choose which has more free IP addresses), VPC and AWS account number and AMI ids
- Create a new cloudformation stack using cloudformation/cloudformation.yaml file
- Wait for this task to complete
- Update config file with correct cluster end point and certificate authority

## Associate Role to Cluster control panel
Update aws-auth-cm-<acname>.yml with  account and role (role, get it from the IAM and search for 'da-kube')

```rolearn: arn:aws:iam::AWSAccountNumber:role/DA-Kube-NodeInstanceRole-xxxxx```

```kubectl apply -f config/test/aws-auth-cm.yaml --kubeconfig=config/test/kubeconfig-<acname>```

Run ```kubectl get nodes --kubeconfig=config/test/kubeconfig``` and check if nodes are attached to the cluster
```$ kubectl get nodes --kubeconfig=config/test/kubeconfig
NAME                                        STATUS    ROLES     AGE       VERSION
ip-x-x-x-x.us-west-2.compute.internal    Ready     <none>    35s       v1.x.x
ip-x-x-x-x.us-west-2.compute.internal   Ready     <none>    29s       v1.x.x
```
## Deploy Tiller
```
helm init --service-account tiller --kubeconfig=config/test/kubeconfig-xxx
kubectl create serviceaccount --namespace kube-system tiller --kubeconfig config/test/kubeconfig-xxx
kubectl create clusterrolebinding tiller-cluster-rule --clusterrole=cluster-admin --serviceaccount=kube-system:tiller --kubeconfig config/test/kubeconfigxxxxx
kubectl patch deploy --namespace kube-system tiller-deploy -p '{"spec":{"template":{"spec":{"serviceAccount":"tiller"}}}}' --kubeconfig config/test/kubeconfigxxxxx
```

## Check Tiller
```kubectl get pods --namespace kube-system --kubeconfig=config/test/kubeconfig```

## Create Auto Scaler
```helm install cluster-autoscaler --name autoscaler --set "autoscalingGroups[0].name=DA-Kubes-NodeGroup-ZM3BO90YD8QN,autoscalingGroups[0].maxSize=20,autoscalingGroups[0].minSize=3" --kubeconfig=config/test/kubeconfig```

Note: Get the value of this "autoscalingGroups[0].name" from CloudFormation from AWS Console -> Search for 'da-kube' or your given cluster name

## Check if autoscaler is running
```helm status autoscaler  --kubeconfig=config/test/kubeconfig```

## Get SSL Certificate
ServiceNow and request for "SSL Certificate"  (internal, give your domain name like "*.hub.yourcompany.com")
Once you get the certificate, follow this process to get the cert and key AWS Certificates
Now you have .pem and .key files, convert them to base64 and update the placeholders below (content of files within "" | base64 > file.txt, note: dont use -n option)
```
apiVersion: v1
kind: Secret
metadata:
  name: traefik-cert
  namespace: kube-system
type: Opaque
data:
  tls.crt: 
  tls.key: 
  ```

Place cert and key in this file in corresponding places and Install Traefik

## Install Traefik
Update crt and key in traefik/values.yaml with tls cert and key
```openssl req -x509 -newkey rsa:4096 -keyout server.key -out server.crt -days 365 -subj '/CN=localhost' -nodes```

```helm install traefik --kubeconfig=config/test/kubeconfig```

- check if traefik is deployed correctly
```kubectl get service --all-namespaces --kubeconfig=config/test/kubeconfig```
```kubectl describe service traefik-ingress-service -n kube-system  --kubeconfig=config/test/kubeconfig```
if you see "Error creating load balancer (will retry): failed to ensure load balancer for service kube-system/traefik-ingress-service: could not find any suitable subnets for creating the ELB". Go to AWS console, VPC -> select the VPC your EKS is running in, select the all DMZ subnets and check if all of them have a tag called "kubernetes.io/cluster/SelKube" with value "shared", DA-Kube is the name of the EKS cluster. If not, add it and check the status of the deployment again.

## Install Dashboard
```kubectl apply -f kubernetes-ingress/kubernetes-dashboard.yaml
kubectl create -f kubernetes-ingress/ingress.yaml --kubeconfig config/test/kubeconfig-expweb-test
```

## Install Jenkins
```helm install --set username=<base64 of username> --set password=<base64 of password> jenkins-master --name jenkins --kubeconfig=config/test/kubeconfig
```


## Create Service Account
This is so that jenkins jobs running in kube can create pods
```
kubectl describe serviceAccounts jenkins --kubeconfig=config/test/kubeconfig --namespace jenkins
kubectl get pods/jenkins-0 -o yaml --kubeconfig=config/test/kubeconfig --namespace jenkins
kubectl describe secrets jenkins-token-56qrq --kubeconfig=config/test/kubeconfig --namespace jenkins
kubectl config view --flatten --minify --kubeconfig=config/test/kubeconfig --namespace jenkins > cluster-cert.txt
kubectl config --kubeconfig=config/test/sa-config set-context svcs-acct-context
Reference: http://docs.shippable.com/deploy/tutorial/create-kubeconfig-for-self-hosted-kubernetes-cluster/
```

Create a config file <name>-config.yaml from the output of above steps - https://github.expedia.biz/Brand-Expedia/da-kubes/blob/master/config/test/sa-config
  
  
## Update Load Balancer to use DMZ subnet
Under VPC,  choose the VPC where Kubernetes cluster is running and search for 'DMZ' subnets.
Add a tag with key as "KubernetesCluster" and value as "DA-Kube", this is the name of the cluster. If some other cluster is also using this, then value will be "shared"
Also make sure there is a tag with just the name "kubernetes.io/role/elb" without any value(empty value).


##  Add IP address to the security group
Add corporate subnet to the security group of the worker nodes (if necessary)

## Changing reserved IP per Worker Node
```kubectl apply -f amazon-vpc-cni-k8s/aws-k8s-cni.yaml --kubeconfig=config/test/kubeconfig```

Ref: https://github.com/aws/amazon-vpc-cni-k8s/blob/master/config/v1.3/aws-k8s-cni.yaml

Why we need this: https://aws.amazon.com/blogs/opensource/vpc-cni-plugin-v1-1-available/

## Validating the setup
```kubectl get pods --all-namespaces --kubeconfig=config/test/kubeconfig```

```kubectl logs <get the traefik-ingress pod name> --namespace=kube-system --kubeconfig=config/test/kubeconfig```

## Get Load Balancer URL
```kubectl get pods,deployments,services --all-namespaces --kubeconfig=config/test/kubeconfig```
Update the LB url in the Rout53 record created in the first step.

## Contact
For any questions, concerns, or problems, please feel free to contact the author Ragavan Ambighananthan at eelam.ragavan@gmail.com / https://twitter.com/ragsambi

## License
This project has been released under the GPL license. Please see the license.txt file for more details.
