---
title: Migration de Jenkins vers Argo
date: 2024-09-02T09:00:00
hero: /post/2024/qa/migration-argo/images/argo-logo.svg
excerpt: "Comment nous avons changé d'orchestrateur de tests automatisés"
authors:
  - jbbernard
description: "Comment nous avons changé d'orchestrateur de tests automatisés"
---

La migration de Jenkins vers une solution basée sur Docker avec Argo Workflows et Argo CD est une démarche stratégique pour l'équipe QA visant à moderniser et optimiser les processus de test automatisés avec une possibilité d'intégration continue (CI) et de déploiement continu (CD).


## Contexte de la migration

Historiquement, l'équipe QA de la DT eTF1 utilisait Jenkins comme orchestrateur de lancement pour ses tests automatisés, via des pipelines d’exécution lancés manuellement ou de manière automatique. Les équipes de dev ont peu à peu cessé de l'utiliser au profit d'autres outils de CI, au point où la QA était le dernier utilisateur et où les équipes OPS ont commencé à évoquer la question d'un décomissionnement.

Deux choix s'offraient alors à nous :
* Rester sur Jenkins, et reprendre à notre charge l'administation et la maintenance de l'outil
* Migrer au profit d'un autre outil 

Jenkins, bien que puissant et largement adopté, peut parfois présenter des défis en termes de gestion de pipelines complexes, d'orchestration de conteneurs et de maintenance. De plus, cette situation fut une bonne opportunité pour challenger nos besoins pour l'avenir. Notre choix s'est donc porté sur une migration vers une stack Argo Workflow / Argo CD dans des environnements Kubernetes.

### Pourquoi choisir Argo ?

Nous avons besoin d'un outil assez complexe pour gérer de multiples environnements, pouvant êtré déployé/éxecuté à la demande mais aussi avec divers déclencheurs git ou programmés, tout en étant assez simple pour être configuré en partie par des ingénieurs automaticiens (non professionnels de la gestion d'infrastructure), avec des ajouts/suppressions/modifications fréquentes de workflows au fur et à mesure des développements à venir, qu'on sait ambitieux, de façon très fluide. Argo étant basé sur Kubernetes, il nous offrait à la fois cette flexibilité et cette scalabilité.\
Nous avions aussi besoin de garder un historique clair de chaque modification sur les workflows, les infrastructures ou les images, pour assurer la stabilité, la reproductibilité et la fiabilité de nos tests, et pouvoir rollback facilement au moindre souci, ceci à moindre coût. L'approche GitOps offerte par Argo répond le mieux à notre besoin, elle permet à nos automaticiens, de gérer leur infrastructure, le déploiement de leur application à tester, la compilation et le lancement de leur tests, ainsi que la génération et la communication des rapports, de la même façon qu'ils gèrent l'implémentation de leurs tests, donc avec une courbe d'apprentissage raisonnable.\
Enfin, cela permet d'uniformiser la stack technique de la QA avec le reste des applications métier, soit déjà en Argo, soit par exemple les interactions avec des workflows CI existant en Github Actions seront beaucoup plus naturelles (langage yaml dans les deux cas).



### Étapes de Migration

#### Analyse de l'environnement actuel :
La première étape consiste d'abord à cartographier notre existant et s'assurer d'avoir tous les pré-requis nécessaire à la migration. Nous avons donc analyser les pipelines existants dans Jenkins et ainsi identifier les dépendances et les configurations spécifiques.

#### Mise en place de Kubernetes :
Une fois cette étude effectuée, nous pouvons passer à la migration en tant que telle, en installant et configurant le cluster Kubernetes, ce qui nous permettra de commencer à déployer des outils nécessaires.  

#### Déploiement d'Argo Workflow :
L'étape suivante consiste à [installer Argo Workflow sur le cluster Kubernetes](https://argo-workflows.readthedocs.io/en/latest/installation/), puis de commencer à créer en dur les workflows en utilisant un template par taches à accomplir (voir plus bas pour la description d'un workflow complet)

#### Configuration d'Argo CD :
Une fois nos workflows définis en dur, nous pouvons passer à ArgoCD pour péréniser la déclaration de ces workflows templates en nous donnant la possibilité de les redéployer en cas de besoin. Nous avons donc d'abord [installé Argo CD sur le cluster Kubernetes](https://argo-cd.readthedocs.io/en/stable/getting_started/) afin de pouvoir y définir les applications et les environnements dans des manifestes Kubernetes, que nous avons versionné dans nos repositories Git pour la synchronisation des applications.

#### Migration des Pipelines Jenkins :
Tous les pré-requis étaient alors réunis pour commencer effectivement la migration.\
Par sécurité, nous avons d'abord commencé par sélectionner un premier pipeline, celui de tests automatisés du site Web de TFOU MAX. Nous avons traduit le pipeline Jenkins en workflow Argo en utilisant les briques définies précédemment, et nous l'avons testé dans un environnement de staging. Une fois celui-ci validé, nous avons pu le déployer dans l'environnement de prod, et après 2 semaines de double run, nous avons comparé les résultats entre Jenkins et Argo avant de décider de stopper le pipeline dans Jenkins.\
Puis nous avons décliné cette procédure de migration pipeline par pipeline, avec à nouveau un double run pour s'assurer du bon fonctionnement.\

#### Validation et Optimisation :
Une fois tous les pipelines Jenkins migrés et transformés en workflow Argo, nous avons pu valider la solution au global, qui donne des résultats identique, avec des temps d'éxécution comparables.\
Il ne restait alors plus que quelques affinages. Nous avons d'abord supervisé les performances pour ajuster les ressources Kubernetes en conséquence (augmentation ou diminution de la RAM et/ou du CPU des instances par exemple). Pour faciliter le débug, les logs ont été envoyés dans Datadog qui permet facilement de retrouver les informations pertinentes grâce à son système de requêtage. Et enfin des métriques ont été mis en place, ainsi que des alertes pour surveiller l'état des workflows et des déploiements.



## Intégration dans notre contexte

Les workflows dans Argo sont décomposés en plusieurs images ayant chacun une tache définie :
* Clone du repository Git
* Exécution du framework de test
* Génération du rapport de test
* Envoi des notifications (Slack et/ou mail)
* Stockage des artefacts dans AWS

![Workflow](images/workflow.svg "Workflow d'exécution d'un job")

La déclaration de chacune de ses images et des ressources se fait de manière simple dans un fichier YAML grâce à [Kustomize](https://kustomize.io/) (inclus nativement dans Kubernetes)
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: qa
resources:
- resources/config-map.yaml
- resources/role-binding.yaml
- resources/service-account.yaml
- templates/git-clone
- templates/runner-katalon
- templates/reporting-cucumber
- templates/email-sending
```

Et on peut ensuite facilement déclarer un workflow complet, avec ses paramètres de lancement custom, toujours grâce à des fichiers YAML, comme par exemple :
```yaml
apiVersion: argoproj.io/v1alpha1
kind: WorkflowTemplate
metadata:
  name: qa-tf1plus-web
spec:
  entrypoint: main
  onExit: on-exit

  arguments:
    parameters:
    - name: branch
      value: main
    - name: profile
      value: VAL
      enum:
      - INT
      - VAL
      - PROD
    - name: browser
      value: chrome
      enum:
      - chrome
      - edge
      - ff
      - safari
    - name: os
      value: mac
      enum:
      - mac
      - windows
    - name: tags
      value: |-
        @Account
        @HomePage
        @LivePage
        @Login
        @ProgramPage
        @SearchPage
        @ShowPage
        @VideoPage

  templates:
  - name: main
    steps:
    - - name: git-clone
        templateRef:
          name: qa-templates-git-clone
          template: git-clone
        arguments:
          parameters:
          - name: repository
            value: path-to-repository
          - name: branch
            value: "{{workflow.parameters.branch}}"
    - - name: run-katalon
        templateRef:
          name: qa-templates-runner-katalon
          template: run
        arguments:
          parameters:
          - name: browserstack-licence
            value: tf1plus-web
          - name: browser
            value: "{{workflow.parameters.os}}_{{workflow.parameters.browser}}"
          - name: profile
            value: "{{workflow.parameters.profile}}"
          - name: suite
            value: TSC_TF1PLUS_WEB_FRONT
          - name: tag
            value: "{{item}}"
        withParam: "{{=toJson(sprig.uniq(sprig.compact(sprig.regexSplit('(\\n|,)', workflow.parameters.tags, -1))))}}"

  - name: on-exit
    steps:
    - - name: reporting
        templateRef:
          name: qa-templates-reporting-cucumber
          template: report
        arguments:
          parameters:
          - name: project
            value: "TF1+ WEB"
          - name: environment
            value: "{{workflow.parameters.profile}}"
          - name: branch
            value: "{{workflow.parameters.branch}}"
          - name: platform
            value: "{{workflow.parameters.os}}"
          - name: browser
            value: "{{workflow.parameters.browser}}" 
```


## Conclusion

La migration de Jenkins vers une solution Docker avec Argo Workflow et Argo CD représente une avancée significative pour notre équipe. D'un outil risquant de correspondre de moins en moins à nos besoins, nous sommes passés à une solution aux nombreux bénéfices en termes de scalabilité, flexibilité, et sécurité.

En terme de perspectives d'avenir, cette migration nous ouvre un champ des possibles et nous avons déjà plein de projets dont certains sont déjà en cours de réalisation.\
On peut par exemple citer l'intégration des workflows de tests automatisés dans les différentes CI des équipes de développement. Il est en effet très simple de déclencher l'éxécution d'un workflow depuis un outil tiers, par exemple en utilisant le framework natif [Argo Events](https://argoproj.github.io/argo-events/sensors/triggers/argo-workflow/) ou depuis Github Actions qui propose ce type d'interface dans sa marketplace.\
Nous avons aussi dans l'idée de mettre en place une interface entre Argo et Jira/Xray, qui nous permettrait par exemple de déclencher un workflow à la création d'une campagne de test dans Xray, puis d'y insérer les résultats des tests automatisés une fois ceci joués, afin de centraliser dans le même référentiel nos resultats de tests manuels et automatisés.