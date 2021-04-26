---
title: Infrastructure EKS
date: 2021-03-04T08:30:00
excerpt: Découvrez notre EKS.
authors:
  - koguchi
description: "Venez découvrir EKS chez eTF1"
---
## Avant propos

Cet article a pour objectif de présenter le cluster EKS qui héberge les applications de e-TF1.  
Avec un focus sur le mécanisme de blue/green de cluster et la scalabilité chez e-TF1. 

## EKS

EKS est le service Kubernetes géré par AWS.  
AWS fournit le controle plane (master) et également de workload (worker).

Nous utilisons actuellement uniquement la partie controle plane.  
Car lors de notre début sur EKS, il n'y avait pas de gestion de la partie worker.


![Architecture EKS](images/EKS.png#darkmode "Architecture EKS")

## Le Déploiement & les outils déployés

Nous déployons le contrôle plane EKS et les nodes via terraform.

Les outils déployés dans notre cluster EKS:  

* aws-auth (aws-iam-authenticator)
* External Secrets
* Cilium
* LinkerD
* Node-Local DNS
* ALB Ingress Controller 
* ExternalDNS
* AWS Node Termination Handler
* Cluster Autoscaler
* Descheduler
* Metrics Server
* FluentD

## Focus ExternalDNS et Scaling.
Nous allons vous partager notre utilisation d'ExternalDNS.

### Contexte eTF1
Nous voulions être capables de faire du déploiment blue/green de cluster EKS.

### La solution adoptée

Nous avons choisi d'utiliser quatre ExternalDNS.

Pourquoi 4? Explication:

* 1 ExternalDNS pour le domaine privé du cluster. (*.eks-blue.devinfra.local)
* 1 ExternalDNS pour le domaine publique du cluster. (*.eks-blue.devinfra.fr)
* 1 ExternalDNS pour le domaine privé en production. (*.devinfra.local)
* 1 ExternalDNS pour le domaine publique en production. (*.devinfra.fr)

![Ingress eks](images/ingress.png#darkmode "Ingress EKS")

Le fichier deploy de l'ExternalDNS privé
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: kube-system
  name: external-dns-private
spec:
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: external-dns-private
  template:
    metadata:
      labels:
        app: external-dns-private
    spec:
      priorityClassName: "system-cluster-critical"
      serviceAccountName: external-dns
      containers:
      - name: external-dns
        image: eu.gcr.io/k8s-artifacts-prod/external-dns/external-dns:v0.7.6
        livenessProbe:
          initialDelaySeconds: 10
          httpGet:
            path: /healthz
            port: 7979
        ports:
        - containerPort: 7979
          protocol: TCP
        args:
        - --source=service
        - --source=ingress
        - --domain-filter=devinfra.local
        - --fqdn-template={{ .Annotations.PrivateDNSName }}.eks-blue.devinfra.local
        - --annotation-filter=PrivateDNSName
        - --ignore-hostname-annotation
        - --provider=aws
        - --policy=upsert-only # would prevent ExternalDNS from deleting any records, omit to enable full synchronization
        - --registry=txt
        - --txt-owner-id=ingress-eks-blue.devinfra.fr
      securityContext:
        fsGroup: 65534
```  

Le fichier Deploy de l'ExternalDNS publique
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
namespace: kube-system
name: external-dns-public
spec:
strategy:
type: Recreate
selector:
matchLabels:
  app: external-dns-public
template:
metadata:
  labels:
    app: external-dns-public
spec:
  priorityClassName: "system-cluster-critical"
  serviceAccountName: external-dns
  containers:
  - name: external-dns
    image: eu.gcr.io/k8s-artifacts-prod/external-dns/external-dns:v0.7.6
    livenessProbe:
      initialDelaySeconds: 10
      httpGet:
        path: /healthz
        port: 7979
    ports:
    - containerPort: 7979
      protocol: TCP
    args:
    - --source=service
    - --source=ingress
    - --domain-filter=devinfra.fr
    - --fqdn-template={{ .Annotations.PublicDNSName }}.eks-blue.devinfra.fr
    - --annotation-filter=PublicDNSName
    - --ignore-hostname-annotation
    - --provider=aws
    - --policy=upsert-only # would prevent ExternalDNS from deleting any records, omit to enable full synchronization
    - --registry=txt
    - --txt-owner-id=ingress-eks-blue.devinfra.fr
  securityContext:
    fsGroup: 65534
```

Le fichier deploy de l'ExternalDNS privé de production
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: kube-system
  name: live-external-dns-private
spec:
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: live-external-dns-private
  template:
    metadata:
      labels:
        app: live-external-dns-private
    spec:
      priorityClassName: "system-cluster-critical"
      serviceAccountName: external-dns
      containers:
      - name: external-dns
        image: eu.gcr.io/k8s-artifacts-prod/external-dns/external-dns:v0.7.6
        livenessProbe:
          initialDelaySeconds: 10
          httpGet:
            path: /healthz
            port: 7979
        ports:
        - containerPort: 7979
          protocol: TCP
        args:
        - --source=service
        - --source=ingress
        - --domain-filter=devinfra.local
        - --fqdn-template={{ .Annotations.PrivateDNSName }}.devinfra.local
        - --annotation-filter=PrivateDNSName
        - --ignore-hostname-annotation
        - --provider=aws
        - --policy=upsert-only # would prevent ExternalDNS from deleting any records, omit to enable full synchronization
        - --registry=txt
        - --txt-owner-id=ingress.devinfra.local
      securityContext:
        fsGroup: 65534
```  

Le fichier Deploy de l'ExternalDNS publique de production
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
namespace: kube-system
name: live-external-dns-public
spec:
strategy:
type: Recreate
selector:
matchLabels:
  app: live-external-dns-public
template:
metadata:
  labels:
    app: live-external-dns-public
spec:
  priorityClassName: "system-cluster-critical"
  serviceAccountName: external-dns
  containers:
  - name: external-dns
    image: eu.gcr.io/k8s-artifacts-prod/external-dns/external-dns:v0.7.6
    livenessProbe:
      initialDelaySeconds: 10
      httpGet:
        path: /healthz
        port: 7979
    ports:
    - containerPort: 7979
      protocol: TCP
    args:
    - --source=service
    - --source=ingress
    - --domain-filter=devinfra.fr
    - --fqdn-template={{ .Annotations.PublicDNSName }}.devinfra.fr
    - --annotation-filter=PublicDNSName
    - --ignore-hostname-annotation
    - --provider=aws
    - --policy=upsert-only # would prevent ExternalDNS from deleting any records, omit to enable full synchronization
    - --registry=txt
    - --txt-owner-id=ingress.devinfra.fr
  securityContext:
    fsGroup: 65534
```
### Bascule blue/green eks

Exemple:  
eks-blue est actuellement le cluster actif.  

![Ingress eks-blue](images/ingress-eks-blue.png#darkmode "Ingress EKS blue")

On stop les ExternalDNS de production sur eks-blue afin d'arrêter les mises à jour des entrées DNS de production.  

![Stop ingress eks-blue](images/stop-ingress-eks-blue.png#darkmode "Stop Ingress EKS blue")


On start les ExternalDNS de production sur eks-green afin de mettre à jours toutes les entrées DNS
![Start ingress eks-blue](images/start-ingress-eks-green.png#darkmode "Start Ingress EKS green")

Il ne reste plus qu'à vérifier que les anciens ingress ne reçoivent plus de trafic avant de les supprimer.

#### Pour la partie kubeconfig.  
Les entrées DNS pour l'API et le champs TXT sont mis à jours via le pipeline CI/CD.

Le champs TXT est utilisé pour connaître la partie cluster name pour aws-authenticator de façon transparente avec un wrapper comme ci-dessous.

#### Le wrapper
```bash
#!/bin/bash
target_cluster=$(dig +short -t TXT "$3" | tr -d '"')
if [ -z "$target_cluster" ]; then target_cluster="$3"; fi
set -- "${@:1:2}" "$target_cluster" "${@:4}"
aws-iam-authenticator "$@"
```

#### Le kubeconfig
```yaml
apiVersion: v1
clusters:
- cluster:
    insecure-skip-tls-verify: true
    server: https://api.devinfra.info
  name: eks.devinfra.info
contexts:
- context:
    cluster: eks.devinfra.info
    user: eks.devinfra.info
  name: eks
kind: Config
preferences:
  colors: true
users:
- name : eks.devinfra.info
  user:
    exec:
      apiVersion: client.authentication.k8s.io/v1alpha1
      command: aws-iam-authenticator-wrapper
      args:
      - token
      - -i
      - eks.devinfra.info
      - -r
      - arn:aws:iam::XXXXXXXXXXXXXXX:role/k8s
      command: aws-iam-authenticator-wrapper
      env:
      - name: AWS_PROFILE
        value: awsaccess
```
### Le scaling

Nous utilisons les HPA + cluster autoscaler + overprovisioning 

Nous utilisons actuellement les metrics cpu/ram pour les HPA.  
L'évolution d'un deployment dans le temps.
![Hpa action](images/hpa-action.png#darkmode "Hpa en action")

L'évolution du nombre de node dans le temps.
![cluster autoscaler action](images/k8s-nodes.png#darkmode "cluster autoscaler en action")


Actuellement nous avons configuré le cluster-proportional-autoscaler en mode linear
```yaml
 linear: "{ \n  \"coresPerReplica\": 20,\n  \"nodesPerReplica\": 6,\n  \"min\": 6\n}"
```

Le deployment du pod overprovisionning est le suivant:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: overprovisioning
  namespace: overprovisioning
spec:
  replicas: 3
  selector:
    matchLabels:
      app: overprovisioning
  template:
    metadata:
      labels:
        app: overprovisioning
    spec:
      affinity:
        nodeAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - preference:
              matchExpressions:
              - key: node-role.kubernetes.io/spot
                operator: In
                values:
                - spot
            weight: 1
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - overprovisioning
              topologyKey: failure-domain.beta.kubernetes.io/zone
            weight: 100
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - overprovisioning
            topologyKey: kubernetes.io/hostname
      containers:
      - image: k8s.gcr.io/pause
        name: reserve-resources
        resources:
          limits:
            cpu: "1"
            memory: 2Gi
          requests:
            cpu: "1"
            memory: 2Gi
      nodeSelector:
        node-role.kubernetes.io/spot: spot
      priorityClassName: overprovisioning
      tolerations:
      - effect: NoSchedule
        key: spot
        operator: Exists

```

Avec la configuration de cluster-proportional-autoscaler + la configuration du deploy overprovisoning.  
On peu conclure que nous déployons 1 pod 1cpu/2Go de ram sur un node tous les 20 cores ou 6 nodes. 


## En conclusion
Nous sommes satisfait de EKS.  
Mais nous sommes toujours à la recherche de plus de résilience et d'une scalabilité plus intelligente qu'un simple monitoring des consommations cpu/ram et ainsi réduire les coûts AWS.  
