---
title: Scaling et Optimation des coûts Infrastructure
date: 2024-12-16T08:00:00
hero: /post/2024/iops/cost-optimisation/images/hero.jpg
excerpt: "Découvrez comment nous optimisons les coût infrastructure tout en tenant des pic de charges importants."
authors:
  - mbugeia
description: "Découvrez comment nous optimisons les coût infrastructure tout en tenant des pic de charges importants."
---

## La plateforme eTF1

Chez TF1, la plateforme eTF1 rassemble les marques clés TF1+, TF1 Info, et TfouMax. Pour répondre aux besoins de scalabilité rapides, notamment lors de grands événements sportifs et de pics d'audience, nous avons entrepris une démarche globale visant à rendre notre infrastructure plus dynamique, plus résiliente, et plus éco-responsable.

### Une infrastructure moderne hébergée principalement sur le cloud
Notre plateforme repose en grande partie sur des clusters Kubernetes sur lesquels tous nos applicatifs sont déployés. A cela sont associés plusieurs services managés autour de la data, tels que RDS, Elasticache, S3, MSK et OpenSearch.

L'infrastructure est déployée as-code via Terraform associé à [Terragrunt](https://terragrunt.gruntwork.io/) pour la factorisation de code et [Atlantis](https://www.runatlantis.io/) pour la CI/CD.

Toutes les ressources Kubernetes de nos cluster sont managées en GitOps via [ArgoCD](https://argo-cd.readthedocs.io). Nous sommes également utilisateurs d'un certains nombre d'opérateur kubernetes de la communauté, parmis eux : [prometheus operator](https://prometheus-operator.dev), [cert-manager](https://cert-manager.io/), [external-dns](https://kubernetes-sigs.github.io/external-dns), [external-secrets](https://external-secrets.io/), [kyverno](https://kyverno.io/), [reloader](https://docs.stakater.com/reloader/), ...

L'essentiel de cette infrastructure est hébergé sur le cloud AWS, bien que nous ayons aussi une partie on-premise, notamment avec un CDN interne pour la diffusion video.

## Une plateforme de production élastique

### Un trafic qui évolue en fonction des programmes, et de l'actualité

L'audience de eTF1 varie au cours de la journée, avec des pics le soir et un trafic réduit la nuit. Des programmes phares tels que `Koh Lanta` ou `The Voice` génèrent d'importants volumes d’utilisateurs, tout comme les événements sportifs majeurs.

Par exemple, la Finale de coupe du monde de foot 2022​ en chiffres​
​- ​2.4 millions d'utilisateurs (source Mediametrie)​
- 3.6Tbps en pic​
- 500 000 créations de compte​
- 576 000 rps (mesure CDN)​
- 30 000 rps (services backend)

De plus selon les composants applicatifs, les patterns de trafic peuvent être réellement différents.
![Profil de traffic match de foot](images/profil_traffic_foot.png#darkmode "Illustration des profils de traffic lors d'un match de foot")

Ces pics de trafic, parfois prévisibles comme lors des matchs de foot ou de l'annonces de résultats électoraux, peuvent aussi survenir de manière inattendue.

Exemple, sur le graphe suivant du 9 juin 2024, le jour des élections européennes, un pic important s’est ajouté à celui attendu de l'annonce des résultats : l'annonce de la dissolution de l’Assemblée Nationale.

![Pic de traffic dissolution](images/pic-dissolution.png#darkmode "Illustration d'un pic de traffic le jour de la dissolution de l'assemblée nationale le 9 juin 2024")

La plateforme doit pouvoir répondre avec un niveau de service satisfaisant​ en toutes circonstances, mais nous devons faire cela en maitrisant nos coûts. C'est pourquoi nous avons du mettre un oeuvre un certain nombre de stratégies de scaling de la plateforme.

### Scaling des clusters avec Karpenter

![karpenter](images/karpenter.png#darkmode "Logo karpenter")

Pour répondre à ces variations de demande au niveau des cluster EKS, nous avons déployé [Karpenter](https://karpenter.sh/), un outil d'autoscaling qui permet de provisionner automatiquement des nœuds Kubernetes en fonction des besoins des pods. Celui ci remplace cluster-autoscaler.

#### Les atouts de Karpenter :
- **Évaluation automatique des contraintes** : Karpenter détecte les pods "Unschedulable", et répartit les pods en respectant les anti-affinités et les besoins de ressources.
- **Configuration native Kubernetes** : Basée sur des CRDs, Karpenter permet d’ajuster les instances en combinant SPOT et OnDemand pour une meilleure gestion des coûts et des interruptions.
- **Visibilité améliorée** : Grâce à Karpenter, nous avons une vision claire de la répartition des nœuds et de la diversité des instances.
- **Consolidation des noeuds** : Karpenter consolide en permanence le cluster pour assurer une utilisation optimale des noeuds en fonction des requests (CPU/RAM) des pods déployés ainsi que du prix des instances EC2.

L'image suivante créé avec eks-node-viewer est un extrait de notre cluster de production. Elle illustre bien :
- la diversité des types d'instance
- l'utilisation d'instances spot
- la consolidation que fait karpenter pour remplir les noeuds

![EKS-node-viewer](images/eks-node-viewer.png#darkmode "Illustration du remplissage et de la diversité des noeud par kapenter")

Voici une exemple proche de la configuration des CRD Karpenter que nous utilisons. On peut voir :
- Une ressource `EC2NodeClass` qui définit les caractéristiques des instances des noeuds
- Une ressource `NodePool` qui utilise l'EC2NodeClass et définit les contraintes du nodepool en termes de limties, de consolidation ainsi que de choix d'instance type.

```yaml
---
apiVersion: karpenter.k8s.aws/v1beta1
kind: EC2NodeClass
metadata:
  name: default
spec:
  amiFamily: AL2
  # un disque root de 100Gi pour stocker images et logs
  blockDeviceMappings:
  - deviceName: /dev/xvda
    ebs:
      encrypted: true
      volumeSize: 100Gi
      volumeType: gp3
  role: Karpenter-xxxxx # role créé automatiquement par le module terraform
  securityGroupSelectorTerms:
  - tags:
      karpenter.sh/discovery: ${cluster} # on choisit les SG via le nom du cluster
  subnetSelectorTerms:
  - tags:
      karpenter.sh/discovery: ${env} # on choisit les subnet via le nom de l'environnement

---
apiVersion: karpenter.sh/v1beta1
kind: NodePool
metadata:
  name: default
spec:
  # stratégie de consolidation des noeuds
  disruption:
    budgets:
    - nodes: 10%
    consolidationPolicy: WhenUnderutilized
    expireAfter: 720h
  # limite de taille du cluster afin de ne pas exploser les coût
  # attention, atteindre la limite empêche tout autoscaling
  limits:
    cpu: 1200
    memory: 1800Gi
  template:
    spec:
      nodeClassRef:
        apiVersion: karpenter.k8s.aws/v1beta1
        kind: EC2NodeClass
        name: default
      requirements:
      # Des instances spot dans le nodepool par défaut !
      - key: karpenter.sh/capacity-type
        operator: In
        values:
        - spot
        - on-demand
      - key: kubernetes.io/arch
        operator: In
        values:
        - amd64
      - key: karpenter.k8s.aws/instance-hypervisor
        operator: In
        values:
        - nitro
      # Des limites de taille d'instances pour éviter un blast radius trop important
      - key: karpenter.k8s.aws/instance-cpu
        operator: Lt
        values:
        - "49"
      - key: karpenter.k8s.aws/instance-memory
        operator: Lt
        values:
        - "100001"
      - key: karpenter.k8s.aws/instance-category
        operator: In
        values:
        - c
        - m
        - r
```

Si une chose est à retenir avec l'utilisation de Karpenter : le `sizing des requests des containers est essentiel`. Le choix des noeuds et la compaction du cluster nécessite que les workloads soient dimensionnées au plus juste de leur utilisation réelle.

L'utilisation des `TopologySpreadConstraints` des `AntiAffinity` ainsi que des `PodDisruptionBudget` est également essentielle pour garantir la `Haute Disponibilité` des applicatifs dans un contexte où Karpenter va continuellement consolider le cluster et donc rescheduler des pods et des noeuds.

Voici par exemple un extrait de configuration que nous mettons sur chacun de nos deployment :
```yaml
  # avec l'antiaffinity on s'assure que les pods ne soit pas tous sur le même host
  affinity:
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - podAffinityTerm:
          labelSelector:
            matchExpressions:
            - key: app.kubernetes.io/name
              operator: In
              values:
              - deployment-name
          topologyKey: kubernetes.io/hostname
        weight: 100
  # avec la topologySpreadConstraints on s'assure que les pods ne soient pas tous sur la même AZ
  topologySpreadConstraints:
  - labelSelector:
      matchLabels:
        app.kubernetes.io/name: deployment-name
    maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: ScheduleAnyway
```


### Scaling des applications avec KEDA

![KEDA](images/keda.png#darkmode "Logo KEDA (Kubernetes Event-driven Autoscaling)")

Pendant plusieurs années, notre plateforme utilisait le Horizontal Pod Autoscaler (HPA) pour gérer l’autoscaling en fonction de la charge CPU et de la mémoire. Avec l’arrivée de [KEDA (Kubernetes Event-driven Autoscaling)](https://keda.sh), nous avons franchi un cap en introduisant des métriques métier comme levier pour le scaling, permettant ainsi des ajustements plus fins et une réactivité accrue.

KEDA nous permet de configurer des "triggers" variés (Prometheus, SQS, Cron) pour scalabiliser nos applications en fonction des besoins métier, comme l'augmentation du nombre d'utilisateurs ou la consommation de ressources spécifiques. Cette approche réduit les délais de montée en charge et optimise la réponse aux variations de trafic, améliorant l’expérience utilisateur et l'efficacité des ressources allouées.

KEDA utilise une CRD `ScaledObject` pour scaler les deployment. Voici un exemple de configuration utilisée :

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  labels:
    app.kubernetes.io/name: middle-catalog-graphql
    scaledobject.keda.sh/name: middle-catalog-graphql
  name: middle-catalog-graphql
  namespace: platform
spec:
  advanced:
    horizontalPodAutoscalerConfig:
      behavior:
        scaleDown:
          policies:
          - periodSeconds: 180
            type: Pods
            value: 50
          stabilizationWindowSeconds: 100
  cooldownPeriod: 100
  fallback:
    failureThreshold: 3
    replicas: 50
  maxReplicaCount: 300
  minReplicaCount: 10
  pollingInterval: 10
  scaleTargetRef:
    name: middle-catalog-graphql
  triggers:
  - metadata:
      type: Utilization
      value: "70"
    type: cpu
  - metadata:
      type: Utilization
      value: "90"
    type: memory
  - metadata:
      metricName: http_total_requests_by_second
      query: sum(irate(http_request_duration_seconds_count{app="middle-catalog-graphql",
        identifier="graphql", method!="OPTIONS"}[2m]))
      serverAddress: http://kube-prometheus-stack-prometheus.observability:9090
      threshold: "45"
    type: prometheus

```

Dans cet exemple nous utilisons 3 triggers pour scaler l'applicatif :
- Le CPU
- La mémoire
- Une métrique prometheus sur l'augmentation du nombre de requêtes par secondes

C'est principalement ce dernier trigger qui nous permet d'être plus réactif sur le scaling. Pour valider le dynamisme et trouver les bon seuils, cela s'est fait de manière expérimentale via des tests de charge.

KEDA permets d'aller beaucoup plus loin que cet exemple, il est capable de scaler des workload Kubernetes sur une multitude de triggers (queue SQS, topic Kafka, cron, ...) à utiliser selon le besoin métier.

### La mise en œuvre
#### Les pratiques mises en place
Pour garantir un autoscaling dynamique et réactif, plusieurs pratiques ont été adoptées :

- Tests de performance : Les tests de charge ont permis de valider l’efficacité de l’autoscaling avec KEDA et Karpenter et mettre en évidence le dynamisme de l'autoscaling.​
- Overprovisioning de 5 % : Un léger surplus sur le cluster garantit la disponibilité immédiate en cas de pic imprévu. Pour cela nous utilisons le chart [Cluster Overprovisioner](https://github.com/codecentric/cluster-overprovisioner).
- Diversification des instances AWS : En utilisant des types d'instances variés, nous améliorons la disponibilité globale et limitons la dépendance à une seule catégorie d’instances.
- Optimisation des coûts : Ces stratégies ont permis une réduction de 15 % de notre facture sur les EC2, sans compromis sur la qualité de service.

Gestion des `NodePool` Karpenter
- Un Managed Node Group est dédié à Karpenter avec des taints spécifiques
- 1 nodepool Karpenter à usage généraliste
- 1 nodepool Karpenter "io" pour les workloads faisant une utilisation intensive des disques

#### Accompagnement des développeurs
Pour tirer pleinement parti de l’architecture, un soutien actif aux équipes de développement est essentiel :

- Consolidation et vigilance : La consolidation de Karpenter nécessite de bien configurer les applications avec des contraintes de topologie, des budgets de perturbation (PDB), et des jobs adaptés pour minimiser les risques. (TopologySpreadConstraints, PDB, jobs...)​
- Documentation : nous avons produit des guides de déploiement clairs pour aider les développeurs à suivre les bonnes pratiques.
- Conformité : l’application des policies [Kyverno](https://kyverno.io) valide la conformité des configurations et renforce la sécurité de notre environnement.
- Helm chart universel : nous fournissons un chart Helm standardisée aux développeurs qui intègre nos bonnes pratiques par défaut.

## L'optimisation des coûts hors prod

Les environnements de développement et de test peuvent rapidement devenir des sources de dépenses importantes s’ils ne sont pas gérés efficacement. Voici quelques stratégies clés pour limiter ces coûts.

### Éviter le gaspillage
- **Dimensionnement ajusté** : Limiter la taille des instances et des clusters pour coller au plus près des besoins réels. Dans le cas de nos cluster EKS c'est Karpenter qui joue ce rôle. Pour les services managés cela est manuel.
- **Services managés optimisés** : Utiliser des configurations mono-AZ pour RDS et Elasticache là où la haute disponibilité n’est pas nécessaire.
- **Surveillance des POCs oubliés** : Nettoyer régulièrement les ressources inutilisées ou les projets pilotes abandonnés.

#### Gestion des données sur S3 :
- Désactiver le versioning des buckets non critiques pour économiser du stockage.
- Penser aux lifecycle policies
#### Instances ARM Graviton sur AWS :
- Les instances Graviton (repérées par un "g" dans leur nom) sont plus économiques et performantes : environ 10 % de coût en moins pour 10 % de performance en plus.
- Elles consomment jusqu’à 60 % d’énergie en moins que les instances classiques AMD64, contribuant ainsi à réduire l’empreinte environnementale.
- Pour les services managés, la seule opération à effectuer est de changer le type d'instance !


### Mesurer et corriger l'efficience des ressources Kubernetes

![Kubecost](images/kubecost.png#darkmode "Logo Kubecost")

Chez tf1 nous utilisons [Kubecost](https://www.kubecost.com/) dont une licence à rétention limitée (15j) ets gratuite avec EKS.

![Kubecost efficiency](images/kubecost_eff.png#darkmode "Illustration de l'efficience dans kubecost")

Kubecost nous permet de donner de la visibiltié aux développeurs sur les ressources effectivement consommées sur les clusters en prod et en hors-prod afin de sizer au mieux leurs containers.

### Extinction des ressources non utilisées

Pour optimiser les coûts, nous avons mis en place l'extinction automatique des environnements hors-production pendant les périodes creuses, comme la nuit et les week-ends. Ces environnements, souvent dédiés au développement et principalement utilisés pendant les heures ouvrées, génèrent des coûts surtout dus aux instances EC2.

#### Approche par Cluster EKS

Nous avons décidé d'adopter une approche où chaque cluster est autonome dans son cycle de vie. Cette stratégie repose sur un opérateur installé sur chaque cluster hors-production qui éteint les ressources non utilisées en heures non ouvrées.

### Solutions envisagées

Nous avons évalué plusieurs solutions open source pour automatiser cette optimisation :

#### Kube-green
  Site officiel : [kube-green.dev](https://kube-green.dev/)
  Kube-green est une solution open source qui permet de gérer l’extinction des ressources inutilisées via des Custom Resource Definitions (CRD). Il agit principalement sur les replicas de *Deployments* et les *CronJobs*, en stockant leur état précédent dans un *secret* pour une reprise sans interruption.

  **Inconvénients** :
  - Limité aux ressources gérées par l’opérateur : ne couvre pas toutes les ressources comme les statefulsets
  - Peut entrer en conflit avec les outils GitOps, compliquant le processus de déploiement continu.

```yaml
apiVersion: kube-green.com/v1alpha1​
kind: SleepInfo​
metadata:
  name: working-hours​
spec:
  weekdays: "1-5"​
  sleepAt: "20:00"​
  wakeUpAt: "08:00"​
  timeZone: "Europe/Rome"​
  suspendCronJobs: true​
  excludeRef:
  - apiVersion: "apps/v1"​
    kind: Deployment​
    name: my-deployment
```

#### Kubecost Cluster Turndown

  Site officiel : [kubecost.com](https://kubecost.com)
  Kubecost propose également une solution open source qui agit directement sur le scaling des nœuds via des CRD. Bien que non compatible avec Karpenter, il peut être utilisé pour les *node pools* managés d’EKS ou de GKE.

  **Inconvénient** :
  - Non compatible avec Karpenter, ce qui limite son utilisation à des clusters utilisant des pools de nœuds managés. Ce qui n'est pas notre cas.

```yaml
apiVersion: kubecost.com/v1alpha1​
kind: TurndownSchedule​
metadata:
  name: example-schedule​
  finalizers:
  - "finalizer.kubecost.com"​
spec:
  start: 2020-03-12T00:00:00Z​
  end: 2020-03-12T12:00:00Z​
  repeat: daily
```

### Développement d’un Outil Interne pour l'Extinction Automatisée

Les outils existants n'étant pas satisfaisants pour répondre aux besoins spécifiques de notre infrastructure, nous avons donc développé un outil interne permettant une gestion des ressources Karpenter. Cet outil prend en charge plusieurs opérations d'extinction et de reprise pour les `NodePool` Karpenter :

- **Sauvegarde et destruction des node pools Karpenter** : Avant chaque extinction, l’outil sauvegarde l’état des `NodePool`, puis les détruit via un delete de la CRD.
- **Désactivation temporaire des alertes** : L’outil désactive les alertes dans *Alertmanager* et crée des silences temporaires avant chaque extinction, minimisant ainsi les notifications superflues pendant les périodes de fermeture.

```yaml
schedulers:
- name: "daily-turndown-without-weekend"​
  karpenterEnabled: true​
  sleepAt: "0 21 * * 1-4" # All days of the week at 21:00​
  wakeUpAt: "50 6 * * 2-5" # All days of the week at 06:50 the next day​
  timezone: "Europe/Paris"​
- name: "weekly-turndown-weekend"​
  karpenterEnabled: true​
  sleepAt: "0 21 * * 5" # All Fridays at 21:00​
  WakeUpAt: "50 6 * * 1" # All Mondays at 06:50​
  timezone: "Europe/Paris"​
alertManagers:
- url: "http://internal-alertmanager.observability.svc.cluster.local:9093"​
  filters:
  - name: "cluster"​
    value: mycluster​
- url: "https://external-alertmanager.exemple.com"​
  filters:
  - name: "cluster"​
    value: mycluster
```

A la destruction du nodepool Karpenter, les instances EC2 s'éteignent automatiquement. A sa re création, Karpenter rallume des noeuds pour instancier tous les pods.

#### Inconvénients

Notre outil interne présente toutefois certains défis :

- **Drift temporaire avec Terraform** : Étant donné que les *node pools* Karpenter sont créés et gérés via Terraform, il se crée un *drift* (décalage) temporaire entre l’état réel des ressources et celui prévu dans notre configuration Terraform.
- **Non open-source** : Le choix de développer un outil interne implique un investissement en maintenance et une dépendance vis-à-vis de nos propres équipes. Nous envisageons d’étudier l’intégration de ces fonctionnalités dans un outil open-source pour réduire cette charge de maintenance et bénéficier de mises à jour de la communauté.

### Aller plus loin : Extinction des ressources au-delà de Kubernetes

Pour optimiser encore davantage les coûts, nous explorons l’extinction de ressources situées en dehors de Kubernetes. Ces ressources, souvent associées à des données ou à des services périphériques, nécessitent une orchestration spécifique. Toutefois, cette stratégie s’applique uniquement aux services payants en cas d’inutilisation, afin d’éviter toute manipulation superflue des ressources gratuites.

Exemple : extinction des clusters RDS ou opensearch

#### Gestion des ressources cloud créées via Kubernetes

Certaines ressources cloud, telles que les load balancers créés automatiquement (par les ingress ou services de type *LoadBalancer*) et les éléments provisionnés via des solutions comme Crossplane, peuvent représenter des coûts non négligeables. La gestion de leur cycle de vie est complexe, car ces ressources dépendent directement de Kubernetes et requièrent des stratégies d’extinction spécifiques, l'extinction des noeuds ne suffit pas.

Une approche possible serait d'étendre notre outil d"extinction de nodepool pour supporter la destruction de plus de types de ressources, et laisser les outils de gitops faire pour la reconstruction au réveil des noeuds.

#### Approche par *Terraform Destroy*

Une autre approche possible consisterait à détruire tout ou partie de l'infrastructure via des *terraform destroy* et de reconstruire via des *terraform apply*. Cette méthode pourrait permettre d’atteindre un niveau d’extinction plus avancé, en contrepartie d'un temps de reconstruction souvent plus important. Par exemple on sait qu'un cluster EKS mets aujourd'hui plus de 20min à être construit, idem pour une instance RDS, là où un cluster OpenSearch peut mettre jusqu'à 45min pour s'initialiser. Il convient aussi souvent de ne pas détruire certains composant clé (comme le réseau) pour éviter une complexité trop importante dans le cycle de reconstruction.

## Conclusion

La plateforme eTF1 a su évoluer pour répondre à des besoins critiques de scalabilité, en particulier lors des pics de trafic générés par des événements majeurs. En intégrant des outils comme Karpenter pour le scaling des clusters Kubernetes et KEDA pour des métriques métier précises, nous avons considérablement amélioré notre capacité à absorber des variations de charge tout en maîtrisant nos coûts.

Au-delà des clusters de production, l’optimisation des environnements hors production est devenue un levier clé. Grâce à des pratiques comme l’extinction automatique des ressources non utilisées et la diversification des types d’instances, nous avons réduit significativement les dépenses inutiles.

Cette démarche s’inscrit dans une logique pragmatique : fournir une plateforme performante et résiliente tout en limitant son empreinte financière et écologique. Ce travail n’est pas une fin en soi : il ouvre la voie à d’autres initiatives pour améliorer l’efficience globale de nos infrastructures.
