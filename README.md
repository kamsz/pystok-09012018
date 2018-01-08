# PyStok 09.01.2018 demo

## Prerequisites

* AWS account
* AWS CLI installed
* kubectl installed
* kops installed
* Helm installed
* Public domain managed by Route53

## Create KOPS cluster

Export AWS credentials:

```
export AWS_ACCESS_KEY_ID=x
export AWS_SECRET_ACCESS_KEY=x
export AWS_REGION=eu-west-1
```

Create S3 bucket to store kops state:

```
aws s3 mb s3://eu-west-1-pystok-kops-state
export KOPS_STATE_STORE=s3://eu-west-1-pystok-kops-state
```

Create Kubernetes cluster based on Debian Jessie (1 master, 2 nodes) in 3 availability zones:

```
kops create cluster --zones=eu-west-1a,eu-west-1b,eu-west-1c pystok.demo.lintops.tech
kops edit cluster
```

Add the following policy under `spec`:

```
  additionalPolicies:
    node: |
      [
        {
          "Effect": "Allow",
          "Action": [
            "route53:ChangeResourceRecordSets"
          ],
          "Resource": [
            "arn:aws:route53:::hostedzone/*"
          ]
        },
        {
          "Effect": "Allow",
          "Action": [
            "route53:ListHostedZones",
            "route53:ListResourceRecordSets"
          ],
          "Resource": [
            "*"
          ]
        }
      ]
```

Finally, create the actual cluster:

```
kops update cluster pystok.demo.lintops.tech --yes
```

After a few minutes validate if cluster is up and running:

```
kops validate cluster
```

## Deploy Kubernetes dashboard

Use `kubectl` to deploy Kubernetes dashboard:
```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/master/src/deploy/recommended/kubernetes-dashboard.yaml
```

Validate if Pod is running:

```
kubectl get pods -n kube-system |grep dashboard
```

Create a proxy to access the dashboard:

```
kubectl proxy
```

Go to http://localhost:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/ to see the dashboard.

## Deploy ingress-nginx

Use the following `kubectl` commands to deploy ingress-nginx:
```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/namespace.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/default-backend.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/configmap.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/tcp-services-configmap.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/udp-services-configmap.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/without-rbac.yaml
kubectl patch deployment -n ingress-nginx nginx-ingress-controller --type='json' \
  --patch="$(curl https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/publish-service-patch.yaml)"
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/provider/aws/service-l4.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/provider/aws/patch-configmap-l4.yaml
```

Validate if default-backend and nginx-ingress-controller Pods are running:

```
kubectl get pods -n ingress-nginx
```

## Deploy external-dns

Use `kubectl` to deploy external-dns:

```
kubectl apply -f deployments/external-dns.yaml
```

Validate if external-dns Pod is running:

```
kubectl get pods -n kube-system |grep external
```

## Deploy sample website

Use `kubectl` to deploy sample website Pod:

```
kubectl apply -f deployments/website.yaml
```

Validate if sample website Pod is running:

```
kubectl get pods
```

Visit http://pystok.demo.lintops.tech :)

## Deploy sample website as Helm chart

At first, remove the previously created Pod, Service and Ingress:

```
kubectl delete -f deployments/website.yaml
```

Initialize Helm:

```
helm init
```

Deploy sample website Helm chart:

```
helm install -n pystok-website charts/website
```

Verify that chart was successfully deployed:

```
helm ls
kubectl get pods
```

Package sample website chart as archive:

```
helm package charts/website -d charts
```

# Clean things up

```
kops delete cluster --name pystok.demo.lintops.tech --yes
```
