# da-kubes
Selenium Grid based Distributed Automation using Kubernetes, Docker, Traefik and Helm

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
## Create CloudFormation stack
- Update cloudformation.yaml with at-least two subnets (choose which has more free IP addresses), VPC and AWS account number and AMI ids
- Create a new cloudformation stack using cloudformation/cloudformation.yaml file
- Wait for this task to complete

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
```kubectl create serviceaccount --namespace kube-system tiller --kubeconfig config/test/kubeconfig```

```kubectl create clusterrolebinding tiller-cluster-rule --clusterrole=cluster-admin --serviceaccount=kube-system:tiller --kubeconfig config/test/kubeconfig```

```kubectl patch deploy --namespace kube-system tiller-deploy -p '{"spec":{"template":{"spec":{"serviceAccount":"tiller"}}}}' --kubeconfig config/test/kubeconfig ```

## Check Tiller
```kubectl get pods --namespace kube-system --kubeconfig=config/test/kubeconfig```

## Create Auto Scaler
```helm install cluster-autoscaler --name autoscaler --set "autoscalingGroups[0].name=DA-Kubes-NodeGroup-ZM3BO90YD8QN,autoscalingGroups[0].maxSize=20,autoscalingGroups[0].minSize=3" --kubeconfig=config/test/kubeconfig```

Note: Get the value of this "autoscalingGroups[0].name" from CloudFormation from AWS Console -> Search for 'da-kube' or your given cluster name

## Check if autoscaler is running
```helm status autoscaler  --kubeconfig=config/test/kubeconfig```

## Update Load Balancer to use DMZ subnet
Under VPC,  choose the VPC where Kubernetes cluster is running and search for 'DMZ' subnets.
Add a tag with key as "KubernetesCluster" and value as "DA-Kube", this is the name of the cluster. If some other cluster is also using this, then value will be "shared"
Also make sure there is a tag with just the name "kubernetes.io/role/elb" without any value(empty value).

## Install Traefik
Update crt and key in traefik/values.yaml with tls cert and key
```openssl req -x509 -newkey rsa:4096 -keyout server.key -out server.crt -days 365 -subj '/CN=localhost' -nodes```

```helm install traefik --kubeconfig=config/test/kubeconfig```

##  Add IP address to the security group
Add corporate subnet to the security group of the worker nodes (if necessary)

## Changing reserved IP per Worker Node
```kubectl apply -f amazon-vpc-cni-k8s/aws-k8s-cni.yaml --kubeconfig=config/test/kubeconfig```

Ref: https://github.com/aws/amazon-vpc-cni-k8s/blob/master/config/v1.3/aws-k8s-cni.yaml

Why we need this: https://aws.amazon.com/blogs/opensource/vpc-cni-plugin-v1-1-available/

## Validating the setup
```kubectl get pods --all-namespaces --kubeconfig=config/test/kubeconfig```

```kubectl logs <get the traefik-ingress pod name> --namespace=kube-system --kubeconfig=config/test/kubeconfig```
