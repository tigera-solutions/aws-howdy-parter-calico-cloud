# AWS Howdy Partner | Calico Cloud
Howdy Partner is a weekly Twitch series highlighting AWS Partner Network (APN) Technology Partners that have built innovative solutions validated by AWS Solutions Architects.<br/>
https://www.imdb.com/title/tt13135054/

## Download and Install the EKSCTL utility
Download and extract the latest release of eksctl with the following command
```
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
``` 
Move the extracted binary to /usr/local/bin.
```
sudo mv /tmp/eksctl /usr/local/bin
``` 
Test that your installation was successful with the following command
```
eksctl version
``` 
First, create an Amazon EKS cluster without any nodes
```
eksctl create cluster  --name aws-howdy-partner  --with-oidc  --without-nodegroup
```
## Make a cluster EU-Compatible:
If necessary, replace the region code with Region your cluster's in
```
sed -i.bak -e 's/us-west-2/eu-west-1/' aws-k8s-cni.yaml
```
Replace <account> with Account from the EKS addon
```
sed -i.bak -e 's/602401143452/602401143452/' aws-k8s-cni.yaml
```
Address for Region that your cluster is in:
```
kubectl apply -f aws-k8s-cni.yaml
```
## Create a node group for the cluster
Verify VPC Networking and CNI plugin is used. Confirm the aws-node pod exists on each node
```
kubectl get pod -n kube-system -o wide
```
Finally, add nodes to your EKS cluster
```
eksctl create nodegroup --cluster aws-howdy-partner --node-type t3.xlarge --nodes=3 --nodes-min=0 --nodes-max=3 --max-pods-per-node 58
```
Confirm regions are configured correctly:
```
kubectl get node -o=jsonpath='{range .items[*]}{.metadata.name}{"\tProviderId: "}{.spec.providerID}{"\n"}{end}'
```  
## Configure Calico Cloud:
Get your Calico Cloud installation script from the Web UI - https://qq9psbdn-management.calicocloud.io/clusters/grid
```
curl https://installer.calicocloud.io/*****.*****-management_install.sh | bash
```
Check for cluster security group of cluster:
```
aws eks describe-cluster --name aws-howdy-partner --query cluster.resourcesVpcConfig.clusterSecurityGroupId
```
If your cluster does not have applications, you can use the following storefront application:
```
kubectl apply -f https://installer.calicocloud.io/storefront-demo.yaml
```
Create the Product Tier:
```
kubectl apply -f https://raw.githubusercontent.com/tigera-solutions/aws-howdy-parter-calico-cloud/main/policies/product.yaml
```  
## Zone-Based Architecture  
Create the DMZ Policy:
```
kubectl apply -f https://raw.githubusercontent.com/tigera-solutions/aws-howdy-parter-calico-cloud/main/policies/dmz.yaml
```
Create the Trusted Policy:
```
kubectl apply -f https://raw.githubusercontent.com/tigera-solutions/aws-howdy-parter-calico-cloud/main/policies/trusted.yaml
``` 
Create the Restricted Policy:
```
kubectl apply -f https://raw.githubusercontent.com/tigera-solutions/aws-howdy-parter-calico-cloud/main/policies/restricted.yaml
```

## Allow Kube-DNS Traffic: 
Create the 'Security' Tier:
``` 
kubectl apply -f https://raw.githubusercontent.com/tigera-solutions/aws-howdy-parter-calico-cloud/main/policies/security.yaml
```
Determine a DNS provider of your cluster (mine is 'coredns' by default)
```
kubectl get deployments -l k8s-app=kube-dns -n kube-system
```    
Allow traffic for Kube-DNS / CoreDNS:
```
kubectl apply -f https://raw.githubusercontent.com/tigera-solutions/aws-howdy-parter-calico-cloud/main/policies/allow-kubedns.yaml
```
  
## Increase the Sync Rate: 
``` 
kubectl patch felixconfiguration.p default -p '{"spec":{"flowLogsFlushInterval":"10s"}}'
kubectl patch felixconfiguration.p default -p '{"spec":{"dnsLogsFlushInterval":"10s"}}'
kubectl patch felixconfiguration.p default -p '{"spec":{"flowLogsFileAggregationKindForAllowed":1}}'
```
Introduce the Rogue Application:
```
kubectl apply -f https://installer.calicocloud.io/rogue-demo.yaml -n storefront
``` 
Quarantine the Rogue Application: 
```
kubectl apply -f https://raw.githubusercontent.com/tigera-solutions/aws-howdy-parter-calico-cloud/main/policies/quarantine.yaml
```
## Introduce Threat Feeds:
Create the FeodoTracker globalThreatFeed: 
``` 
kubectl apply -f https://raw.githubusercontent.com/tigera-solutions/aws-howdy-parter-calico-cloud/main/threatfeed/feodo-tracker.yaml
```
Verify the GlobalNetworkSet is configured correctly:
``` 
kubectl get globalnetworksets threatfeed.feodo-tracker -o yaml
``` 
Applies to anything that IS NOT listed with the namespace selector = 'acme' 
```
kubectl apply -f https://raw.githubusercontent.com/tigera-solutions/aws-howdy-parter-calico-cloud/main/threatfeed/block-feodo.yaml
```
Create a Default-Deny in the 'Default' namespace:
```
kubectl apply -f https://raw.githubusercontent.com/tigera-solutions/aws-howdy-parter-calico-cloud/main/policies/default-deny.yaml
```
## Anonymization Attacks:  
Create the threat feed for EJR-VPN: 
``` 
kubectl apply -f https://docs.tigera.io/manifests/threatdef/ejr-vpn.yaml
```
Create the threat feed for Tor Bulk Exit Nodes: 
``` 
kubectl apply -f https://docs.tigera.io/manifests/threatdef/tor-exit-feed.yaml
```
Additionally, feeds can be checked using following command:
``` 
kubectl get globalthreatfeeds 
```

  
## Configuring Honeypods

Create the Tigera-Internal namespace and alerts for the honeypod services:
```
kubectl apply -f https://docs.tigera.io/manifests/threatdef/honeypod/common.yaml
```

Get the Tigera Pull Secret from the Tigera Guardian namespace:
```
kubectl get secrets tigera-pull-secret -n tigera-guardian -o json
```

We need to get the Tigera pull secret output yaml, and put it into a 'pull-secret.yaml' file:
```
kubectl get secret tigera-pull-secret -n tigera-guardian -o json > pull-secret.json
```

Edit the below pull secret - removing all metadata from creationTimestamp down to the name of the tigera-pull-scret. Use Capital 'D' and lower-case 'd' while in insert mode of VI to remove this content

```
vi pull-secret.json
```

Apply changes to the below pull secret
```
kubectl apply -f pull-secret.json
```

Add tigera-pull-secret into the namespace tigera-internal
```
kubectl create secret generic tigera-pull-secret --from-file=.dockerconfigjson=pull-secret.json --type=kubernetes.io/dockerconfigjson -n tigera-internal
```

# IP Enumeration
Expose a empty pod that can only be reached via PodIP, we can see when the attacker is probing the pod network:
```
kubectl apply -f https://docs.tigera.io/manifests/threatdef/honeypod/ip-enum.yaml 
```

# Exposed service (nginx)
Expose a nginx service that serves a generic page. The pod can be discovered via ClusterIP or DNS lookup. 
An unreachable service tigera-dashboard-internal-service is created to entice the attacker to find and reach, tigera-dashboard-internal-debug:
```
kubectl apply -f https://docs.tigera.io/manifests/threatdef/honeypod/expose-svc.yaml 
```

# Vulnerable Service (MySQL)
Expose a SQL service that contains an empty database with easy access. 
The pod can be discovered via ClusterIP or DNS lookup:
```
kubectl apply -f https://docs.tigera.io/manifests/threatdef/honeypod/vuln-svc.yaml 
```

Verify the deployment - ensure that honeypods are running within the 'tigera-internal' namespace:
```
kubectl get pods -n tigera-internal -o wide
```

And verify that global alerts are set for honeypods:
```
kubectl get globalalerts
```

  
  
  
## Deploy the Boutique Store Application

```
kubectl apply -f https://raw.githubusercontent.com/GoogleCloudPlatform/microservices-demo/master/release/kubernetes-manifests.yaml
```  
We also offer a test application for Kubernetes-specific network policies:
```
kubectl apply -f https://raw.githubusercontent.com/tigera-solutions/aws-howdy-parter-calico-cloud/main/workloads/test.yaml
```

#### Block the test application
Deny the frontend pod traffic:
```
kubectl apply -f https://raw.githubusercontent.com/tigera-solutions/aws-howdy-parter-calico-cloud/main/policies/frontend-deny.yaml
```
Allow the frontend pod traffic:
```
kubectl delete -f https://raw.githubusercontent.com/tigera-solutions/aws-howdy-parter-calico-cloud/main/policies/frontend-deny.yaml
```

#### Introduce segmented policies
Deploy policies for the Boutique application:
```
kubectl apply -f https://raw.githubusercontent.com/tigera-solutions/aws-howdy-parter-calico-cloud/main/policies/boutique-policies.yaml
``` 
Deploy policies for the K8 test application:
```
kubectl apply -f https://raw.githubusercontent.com/tigera-solutions/aws-howdy-parter-calico-cloud/main/policies/test-app.yaml
```
## Alerting
```
kubectl apply -f https://raw.githubusercontent.com/tigera-solutions/aws-howdy-parter-calico-cloud/main/alerting/networksets.yaml
```
```
kubectl apply -f https://raw.githubusercontent.com/tigera-solutions/aws-howdy-parter-calico-cloud/main/alerting/dns-access.yaml
```
```
kubectl apply -f https://raw.githubusercontent.com/tigera-solutions/aws-howdy-parter-calico-cloud/main/alerting/lateral-access.yaml
``` 
## Compliance Reporting
```   
kubectl apply -f https://raw.githubusercontent.com/tigera-solutions/aws-howdy-parter-calico-cloud/main/reporting/cis-report.yaml
```
```  
kubectl apply -f https://raw.githubusercontent.com/tigera-solutions/aws-howdy-parter-calico-cloud/main/reporting/inventory.yaml
```
``` 
kubectl apply -f https://raw.githubusercontent.com/tigera-solutions/aws-howdy-parter-calico-cloud/main/reporting/network-access.yaml  
```
Run the below .YAML manifest if you had configured audit logs for your EKS cluster:<br/>
https://docs.tigera.io/compliance/compliance-reports/compliance-managed-cloud#enable-audit-logs-in-eks

```
kubectl apply -f https://raw.githubusercontent.com/tigera-solutions/aws-howdy-parter-calico-cloud/main/reporting/policy-audit.yaml  
```
