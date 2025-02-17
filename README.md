# GKE-Wordpress-Cluster

This repo will build off my previous project for a local network cluster by expanding the scope to make the website available over the internet using GKE

# For Information Regarding The Helm Charts and YAMLs used for the Actual Wordpress Install See the Link Below

https://github.com/MaxShaw5/k3s-wordpress-home-lab

This repo will be a bit of a "sister repo" that handles the GKE aspect of the project.

# Laying The Ground Work For the GKE Cluster

First things first, I created a cluster in the GCP console by simply using the GUI to make a 2 node cluster with 1 node in each zone (us-central1-b and us-central1-c) with a machine type of e2 small to keep costs down.

I then started a cloud shell in the cluster so that I could clone my original Git Repo (linked above) into the cluster so that I could install my Helm chart onto the nodes.

## ArgoCD Setup in GKE

Before I did a ```helm install wordpress``` I wanted to make sure I got ArgoCD up and running so that I could manage the continuous deployment of cluster resources on my GKE cluster. I installed ArgoCD with the command available on the Argo docs ```kubectl apply -k https://github.com/argoproj/argo-cd/manifests/crds\?ref\=stable```

After getting it all installed onto my nodes, I needed to change the argocd-server service over to a LoadBalancer so I could make it available externally. This was done by editing the service through the CLI with ```kubectl edit service -n argocd argocd-server```

```
apiVersion: v1
kind: Service
metadata:
  annotations:
    cloud.google.com/neg: '{"ingress":true}'
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","kind":"Service","metadata":{"annotations":{},"labels":{"app.kubernetes.io/component":"server","app.kubernetes.io/name":"argocd-server","app.kubernetes.io/part-of":"argocd"},"name":"argocd-server","namespace":"argocd"},"spec":{"ports":[{"name":"http","port":80,"protocol":"TCP","targetPort":8080},{"name":"https","port":443,"protocol":"TCP","targetPort":8080}],"selector":{"app.kubernetes.io/name":"argocd-server"}}}
  creationTimestamp: "2025-02-14T17:16:47Z"
  finalizers:
  - service.kubernetes.io/load-balancer-cleanup
  labels:
    app.kubernetes.io/component: server
    app.kubernetes.io/name: argocd-server
    app.kubernetes.io/part-of: argocd
  name: argocd-server
  namespace: argocd
  resourceVersion: "111647"
  uid: 29328df0-a2cc-463f-bff3-a57ef900c95c
spec:
  allocateLoadBalancerNodePorts: true
  clusterIP: redacted
  clusterIPs:
  - redacted
  externalTrafficPolicy: Cluster
  internalTrafficPolicy: Cluster
  ipFamilies:
  - IPv4
  ipFamilyPolicy: SingleStack
  ports:
  - name: http
    nodePort: 31770
    port: 80
    protocol: TCP
    targetPort: 8080
  - name: https
    nodePort: 31388
    port: 443
    protocol: TCP
    targetPort: 8080
  selector:
    app.kubernetes.io/name: argocd-server
  sessionAffinity: None
  type: LoadBalancer  ## This was changed to LoadBalancer - thats it!
status:
  loadBalancer:
    ingress:
    - ip: 34.172.27.223
      ipMode: VIP
```

After changing the service to a LoadBalancer, GKE assigned it an External IP automatically.

To make the webUI accessible from a domain name, I went over to CloudFlare (my domain owner) and made a new A record for my domain (maxshaw.online) with the host set to argocd (the subdomain) and the destination set to the external IP of the load balancer.

I also had to change the SSL/TLS settings in CloudFlare to "Full" in order for my traffic to come through correctly.

This allowed traffic to flow into the LoadBalancer and for it to serve up the ArgoCD UI through my domain (argocd.maxshaw.online). 

Heres a screenshot:

![image](https://github.com/user-attachments/assets/8bc42e8f-ae70-4527-8020-b83c5d972478)
