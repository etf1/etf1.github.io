---
title: Infrastructure EKS
date: 2021-09-29T08:30:00
hero: /post/2021/architecture/eks/images/hero.jpg
excerpt: Découvrez notre EKS.
authors:
  - koguchi
description: "Venez découvrir EKS chez eTF1"
---
## Avant propos

Cet article présente le cluster EKS qui héberge les applications d'e-TF1:  

* Le site de streaming de [TF1](https://www.tf1.fr)
* Le site d'info [LCI](https://www.lci.fr)
* Le site jeunesse [TFOUMAX](https://www.tfoumax.fr)   

Et plus précisément notre gestion de droit au sein du cluster eks ainsi que le modèle de déploiement blue/green du cluster kubernetes managé par AWS (EKS) que nous utilisons. 

## Brève description de kubernetes à la sauce EKS

Kubernetes est un orchestrateur de container.  
Pour simplifier, il s'agit d'un logiciel qui s'occupe de gérer la vie d'un ensemble de containers docker.   
Un peu de vocabulaire pour commencer:

* Un cluster kubernetes est un ensemble de machines (node) qui permettent d'exécuter des containers.
* Un Node est une machine de travail (physique ou virtuelle) d'un cluster.
* Un Pod est le composant le plus petit qui gère directement un ou plusieurs containers qui partagent la même adresse IP.
* Un Deployment définit l'état cible des pods du cluster.
* Un Daemonset est un deployment qui déploie un pod par node.
* Un Ingress est une ressource kubernetes qui gère l'accès externe aux services dans un cluster.
* Un Namespace est une segmentation dans le nommage de ressources car les noms de ressources doivent être uniques dans un namespace.

Toutes les définitions des ressources kubernetes sont stockées dans API kubernets qui est dans notre cas EKS
![Bref EKS](images/bref-kubernetes.png#darkmode "Bref EKS").


## Architecture EKS e-TF1

Nous utilisons actuellement uniquement la partie control plane.  
Il est également possible de faire gérer la gestion des nodes par AWS via
les managed node groups. Pour plus d'explication je vous invite à lire https://docs.aws.amazon.com/eks/latest/userguide/managed-node-groups.html

Au sein de notre cluster, nous avons des nodes privés qui passent par la nat gateway pour l'accès internet et des nodes publics qui permettent aux pods l'accès internet sans passer par la nat gateway.  
Et nous avons des subnets dédiés pour les pod kubernetes afin de s'affranchir des problèmes de range IP dans les subnets privés ou publics. 
<br>
![Architecture EKS](images/EKS.png#darkmode "Architecture EKS")

## Le déploiement & les outils utilisés

Nous déployons le contrôle plane EKS et les nodes via terraform.  
Les outils additionnels déployés dans notre cluster EKS:
 
* [aws-auth](https://docs.aws.amazon.com/eks/latest/userguide/add-user-role.html) pour la gestion de droit 
* [External Secrets](https://github.com/external-secrets/kubernetes-external-secrets) pour récupérer les secrets stockés dans secrets manager AWS
* [AWS VPC CNI](https://github.com/aws/amazon-vpc-cni-k8s) pour le réseau kubernetes
* [LinkerD](https://linkerd.io/) comme load balancer interne pour GRPC
* [Node-Local DNS](https://kubernetes.io/docs/tasks/administer-cluster/nodelocaldns/) pour améliorer les performances DNS.
* [ALB Ingress Controller](https://github.com/kubernetes-sigs/aws-load-balancer-controller) pour créer les load balancers afin d'accéder aux services lancés dans kubernetes.
* [ExternalDNS](https://github.com/kubernetes-sigs/external-dns) pour lier un dns aux load blalancers créés
* [AWS Node Termination Handler](https://github.com/aws/aws-node-termination-handler) pour détecter les interruptions d'instance spot
* [Cluster Autoscaler](https://github.com/kubernetes/autoscaler/tree/master/cluster-autoscaler/cloudprovider/aws) pour permettre l'augmentation/réduction du nombre de node suivant la charge
* [Descheduler](https://github.com/kubernetes-sigs/descheduler) pour une meilleure répartition des pods au sein du cluster kubernetes
* [Metrics Server](https://github.com/kubernetes-sigs/metrics-server) pour récupérer les metrics des containers
* [FluentD](https://docs.fluentd.org/container-deployment/kubernetes) pour récupérer les logs des containers


## Gestion des droits au sein d'EKS

Notre besoin pour la gestion de droit est classique.  
Nous avons:

* Les Ops qui ont besoin d'avoir un contrôle total sur EKS.
* Les Leads Devs qui doivent pouvoir modifier leurs ressources EKS.
* Les Devs qui ont seulement besoin de consulter.

Nous avons donc défini 3 rôles IAM.

* Le rôle IAM etf1-crossaccount-k8sadmin pour les Ops.
* Le rôle IAM etf1-crossaccount-k8sreadwrite pour les Leads Devs.
* Le rôle IAM etf1-crossaccount-k8sreadonly pour les Devs.

On configure aws-auth pour associer le rôle IAM à un/des groupes kubernetes.  
Dans notre cas, nous utilisons les groupes kubernetes custom suivants:

* readwrite
* readonly
* debugmode

Voici la configmap aws-auth sur l'association rôle IAM et groupe kubernetes.
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  labels:
    k8s-app: aws-iam-authenticator
  name: aws-auth
  namespace: kube-system
data:
  mapRoles: |
    - rolearn: arn:aws:iam::XXXXXXXXX:role/etf1-crossaccount-k8sadmin
      username: kubernetes-admin
      groups:
        - system:masters
    - rolearn: arn:aws:iam::XXXXXXXXX:role/etf1-crossaccount-k8sreadwrite
      username: kubernetes-readwrite
      groups:
        - readwrite
        - readonly
        - debugmode
    - rolearn: arn:aws:iam::XXXXXXXXX:role/etf1-crossaccount-k8sreadonly
      username: kubernetes-readonly
      groups:
        - readonly
        - debugmode
```

Et la définition des droits des différents groupes (clusterrolebinding/clusterrole/role).
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  annotations:
  name: readonly-role-view
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: view
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: Group
  name: readonly

```
Le clusterrole view est builtin et ressemble à ceci:
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
    rbac.authorization.k8s.io/aggregate-to-edit: "true"
  name: view
rules:
- apiGroups:
  - ""
  resources:
  - configmaps
  - endpoints
  - persistentvolumeclaims
  - persistentvolumeclaims/status
  - pods
  - replicationcontrollers
  - replicationcontrollers/scale
  - serviceaccounts
  - services
  - services/status
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - ""
  resources:
  - bindings
  - events
  - limitranges
  - namespaces/status
  - pods/log
  - pods/status
  - replicationcontrollers/status
  - resourcequotas
  - resourcequotas/status
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - ""
  resources:
  - namespaces
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - apps
  resources:
  - controllerrevisions
  - daemonsets
  - daemonsets/status
  - deployments
  - deployments/scale
  - deployments/status
  - replicasets
  - replicasets/scale
  - replicasets/status
  - statefulsets
  - statefulsets/scale
  - statefulsets/status
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - autoscaling
  resources:
  - horizontalpodautoscalers
  - horizontalpodautoscalers/status
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - batch
  resources:
  - cronjobs
  - cronjobs/status
  - jobs
  - jobs/status
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - extensions
  resources:
  - daemonsets
  - daemonsets/status
  - deployments
  - deployments/scale
  - deployments/status
  - ingresses
  - ingresses/status
  - networkpolicies
  - replicasets
  - replicasets/scale
  - replicasets/status
  - replicationcontrollers/scale
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - policy
  resources:
  - poddisruptionbudgets
  - poddisruptionbudgets/status
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - networking.k8s.io
  resources:
  - ingresses
  - ingresses/status
  - networkpolicies
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - metrics.k8s.io
  resources:
  - pods
  verbs:
  - get
  - list
  - watch
```

Le groupe debugmode défini par namespace
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  annotations:
  name: debug-mode
  namespace: namespace-X
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: debug-mode
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: Group
  name: debugmode
  namespace: namespace-X

```
Le rôle debug-mode défini par namespace
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  annotations:
  name: debug-mode
  namespace: namespace-X
rules:
- apiGroups:
  - ""
  resources:
  - pods
  - pods/portforward
  verbs:
  - get
  - list
  - create
- apiGroups:
  - ""
  resources:
  - pods/exec
  verbs:
  - create
```

Le groupe readwrite est également défini par namespace et est associé à 2 rôles:
Le rôle edit qui est buildin et le rôle spécifique readwrite-secrets pour les external secrets 
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  annotations:
  name: readwrite
  namespace: namespace-X
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: edit
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: Group
  name: readwrite
  namespace: namespace-X
```

```yaml
kind: RoleBinding
metadata:
  annotations:
  name: readwrite-secrets
  namespace: namespace-X
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: readwrite-secrets
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: Group
  name: readwrite
  namespace: namespace-X

```
Le rôle edit builtin
```yaml
aggregationRule:
  clusterRoleSelectors:
  - matchLabels:
      rbac.authorization.k8s.io/aggregate-to-edit: "true"
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
    rbac.authorization.k8s.io/aggregate-to-admin: "true"
  name: edit
rules:
- apiGroups:
  - ""
  resources:
  - pods/attach
  - pods/exec
  - pods/portforward
  - pods/proxy
  - secrets
  - services/proxy
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - ""
  resources:
  - serviceaccounts
  verbs:
  - impersonate
- apiGroups:
  - ""
  resources:
  - pods
  - pods/attach
  - pods/exec
  - pods/portforward
  - pods/proxy
  verbs:
  - create
  - delete
  - deletecollection
  - patch
  - update
- apiGroups:
  - ""
  resources:
  - configmaps
  - endpoints
  - persistentvolumeclaims
  - replicationcontrollers
  - replicationcontrollers/scale
  - secrets
  - serviceaccounts
  - services
  - services/proxy
  verbs:
  - create
  - delete
  - deletecollection
  - patch
  - update
- apiGroups:
  - apps
  resources:
  - daemonsets
  - deployments
  - deployments/rollback
  - deployments/scale
  - replicasets
  - replicasets/scale
  - statefulsets
  - statefulsets/scale
  verbs:
  - create
  - delete
  - deletecollection
  - patch
  - update
- apiGroups:
  - autoscaling
  resources:
  - horizontalpodautoscalers
  verbs:
  - create
  - delete
  - deletecollection
  - patch
  - update
- apiGroups:
  - batch
  resources:
  - cronjobs
  - jobs
  verbs:
  - create
  - delete
  - deletecollection
  - patch
  - update
- apiGroups:
  - extensions
  resources:
  - daemonsets
  - deployments
  - deployments/rollback
  - deployments/scale
  - ingresses
  - networkpolicies
  - replicasets
  - replicasets/scale
  - replicationcontrollers/scale
  verbs:
  - create
  - delete
  - deletecollection
  - patch
  - update
- apiGroups:
  - policy
  resources:
  - poddisruptionbudgets
  verbs:
  - create
  - delete
  - deletecollection
  - patch
  - update
- apiGroups:
  - networking.k8s.io
  resources:
  - ingresses
  - networkpolicies
  verbs:
  - create
  - delete
  - deletecollection
  - patch
  - update
- apiGroups:
  - metrics.k8s.io
  resources:
  - pods
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - ""
  resources:
  - configmaps
  - endpoints
  - persistentvolumeclaims
  - persistentvolumeclaims/status
  - pods
  - replicationcontrollers
  - replicationcontrollers/scale
  - serviceaccounts
  - services
  - services/status
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - ""
  resources:
  - bindings
  - events
  - limitranges
  - namespaces/status
  - pods/log
  - pods/status
  - replicationcontrollers/status
  - resourcequotas
  - resourcequotas/status
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - ""
  resources:
  - namespaces
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - apps
  resources:
  - controllerrevisions
  - daemonsets
  - daemonsets/status
  - deployments
  - deployments/scale
  - deployments/status
  - replicasets
  - replicasets/scale
  - replicasets/status
  - statefulsets
  - statefulsets/scale
  - statefulsets/status
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - autoscaling
  resources:
  - horizontalpodautoscalers
  - horizontalpodautoscalers/status
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - batch
  resources:
  - cronjobs
  - cronjobs/status
  - jobs
  - jobs/status
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - extensions
  resources:
  - daemonsets
  - daemonsets/status
  - deployments
  - deployments/scale
  - deployments/status
  - ingresses
  - ingresses/status
  - networkpolicies
  - replicasets
  - replicasets/scale
  - replicasets/status
  - replicationcontrollers/scale
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - policy
  resources:
  - poddisruptionbudgets
  - poddisruptionbudgets/status
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - networking.k8s.io
  resources:
  - ingresses
  - ingresses/status
  - networkpolicies
  verbs:
  - get
  - list
  - watch
```
Le rôle readwrite-secrets 
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  annotations:
  name: readwrite-secrets
  namespace: namespace-X
rules:
- apiGroups:
  - kubernetes-client.io
  resources:
  - externalsecrets
  verbs:
  - get
  - watch
  - list
  - create
  - update
  - patch

```

# Le déploiement du cluster EKS en mode blue/green.

## Contexte eTF1
Nous voulons être capables d'apporter des changements dans de cluster EKS avec une capacité de rollback rapide en cas d'effet non désirable.  
Nous avons opté pour un déploiement en mode blue/green.  
C'est à dire avoir 2 infrastructures parallèles de cluster EKS et basculer le trafic entre le cluster N (blue) et le cluster N+1 (green) lors de nos changements majeurs (changement de CNI, upgrade de version, etc). 

## Illustration d'un blue/green  
   
![Blue green](images/aws-blue-green-simple.png#darkmode "Blue green simple")


## Implémentation

Notre solution se base sur l'utilisation du projet open source ExternalDNS qui permet de synchroniser les DNS avec les Load balancers.  
Nous avons pour chaque cluster EKS blue/green un sous-domaine privé et un sous-domaine public.  
Et nous avons un domaine privé et un domaine public partagé entre les clusters blue/green qui correspond aux domaines en production.

Schéma d'explication pour les domaines privés
![EKS blue green etf1](images/eks-blue-green-etf1.png#darkmode "EKS Blue green etf1")

Techniquement cela se matérialise par l'utilisation de quatre ExternalDNS:

* 1 ExternalDNS pour le domaine privé du cluster. (*.eks-blue.devinfra.local)
* 1 ExternalDNS pour le domaine public du cluster. (*.eks-blue.devinfra.fr)
* 1 ExternalDNS pour le domaine privé partagé. (*.devinfra.local)
* 1 ExternalDNS pour le domaine public partagé. (*.devinfra.fr)


Le fichier deploy de l'ExternalDNS privé du cluster
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

Le fichier Deploy de l'ExternalDNS public du cluster
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

Le fichier deploy de l'ExternalDNS privé partagé en production
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

Le fichier Deploy de l'ExternalDNS public partagé en production
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
### Exemple de bascule blue/green eks

Dans cet exemple simplifié, nous avons un cluster EKS eks-blue et un cluster EKS eks-green.  
Ils ont tous les deux un ingress privé hello et un ingress public world.  
Le domaine partagé est devinfra.local pour le domaine privé et devinfra.fr pour le domaine public.  
Nous avons une entrée DNS api.devinfra.local qui correspond à une API EKS en production.  
L'entrée DNS hello.devinfra.local vers une application interne et l'entrée DNS world.devinfra.fr vers une application accessible depuis internet.

Les entrées DNS de production pointent vers blue.  

![Ingress eks-blue](images/ingress-eks-blue.png#darkmode "Ingress EKS blue")

On arrête les ExternalDNS de production sur eks-blue afin d'arrêter les mises à jour des entrées DNS de production.  

![Stop ingress eks-blue](images/stop-ingress-eks-blue.png#darkmode "Stop Ingress EKS blue")

On met à jour l'entrée DNS de l'API EKS sur le domaine partagé pour pointer vers l'API EKS green.

![change api eks](images/change-api-eks.png#darkmode "Change API EKS")

On démarre les ExternalDNS de production sur eks-green afin de mettre à jour toutes les entrées DNS
![Start ingress eks-blue](images/start-ingress-eks-green.png#darkmode "Start Ingress EKS green")

Il ne reste plus qu'à vérifier que les anciens ingress ne reçoivent plus de trafic avant de les supprimer.

On a réalisé une bascule blue/green de cluster EKS. 
 
Mais en basculant de cluster, il y a un problème avec l'authentification kubectl car il est nécessaire de définir le cluster name du cluster EKS.


### Comment avoir un cluster name dynamique

Pour cela nous avons introduit une entrée DNS de type TXT qui a pour valeur le nom du cluster.  
Nous avons développé un petit script qui récupère la valeur du champ TXT et le remplace lors de l'appel à aws-iam-authenticator.

Le script entre kubectl et aws-iam-authenticator s'appelle aws-iam-authenticator-wrapper
```bash
#!/bin/bash
target_cluster=$(dig +short -t TXT "$3" | tr -d '"')
if [ -z "$target_cluster" ]; then target_cluster="$3"; fi
set -- "${@:1:2}" "$target_cluster" "${@:4}"
aws-iam-authenticator "$@"
```

exemple de kubeconfig (l'entrée DNS avec le champ TXT défini est eks.devinfra.info)
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
## Conclusion
Nous sommes globalement satisfaits de EKS. Il fournit un bon support des outils dont nous avons besoin et nous a permis la mise en place d'un processus de déploiement continu nécessaire pour nos applications. 
Notre prochain article décrira ce que nous avons mis en place pour gérer la scalabilité. Ceci est d’autant plus important car cela nous permet non seulement de réduire les coûts mais aussi l’empreinte carbone de notre activité.


