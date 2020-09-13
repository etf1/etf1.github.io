---
title: Présentation architecture MYTF1
date: 2020-09-01
hero: /post/2020/architecture/presentation/images/hero.jpg
excerpt: Une présentation technique de la stack MYTF1
authors:
  - rpinsonneau
---

# MYTF1 quésaco ?

[MYTF1](https://www.tf1.fr/) est le service de replay du [groupe TF1](https://www.groupe-tf1.fr). Il permet à nos utilisateurs de voir ou revoir en streaming les programmes des chaînes suivantes : [TF1](https://www.tf1.fr/tf1), [TMC](https://www.tf1.fr/tmc), [TFX](https://www.tf1.fr/tfx), [TF1 Séries Films](https://www.tf1.fr/tf1-series-films) et [LCI](https://www.lci.fr/). Il est disponible sur la plupart des écrans: Web, Mobile (iOS, Android) et sur les box des principaux opérateurs (IPTV). Ce service gratuit tire principalement ses revenus de la publicité.

MYTF1 englobe un large spectre de sujets :

- le streaming vidéo et l'encodage des contenus
- le ciblage de la publicité
- la recommendation de contenu (la data)
- l'animation des contenus (l'éditorialisation)
- la gestion des données utilisateurs (historique de lecture, mise en favoris, etc...)

# Les technos utilisées

MYTF1 existe depuis 2011 et a été depuis plusieurs fois refondu from scratch.

| Période            | Technos                                              |
| ------------------ | ---------------------------------------------------- |
| 2011 - 2015        | PHP, MySQL                                           |
| 2015 - 2019        | NodeJS, PostgreSQL                                   |
| 2019 - Aujourd'hui | Go, GraphQL, gRPC, MongoDB, Kafka, Redis, Kubernetes |

## Backend

Aujourd'hui le backend est constitué d'un ensemble de micro-services écrits en [Go](https://golang.org). Après la période [NodeJS](https://nodejs.org/), nous avons décidé de retourner à un language fortement typé. La façon dont Go gère la concurrence (go routine) permet de tenir le fort traffic de MYTF1 et est particulièrement adapté a un écosysteme **kubernetes** :

- emprunte mémoire faible
- démarrage rapide (binaire compilé)
- taille des images docker réduite
- idéal pour des services HTTP / [gRPC](https://grpc.io)

C'est également un language rapide à apprendre.

### GraphQL

L'API [GraphQL](https://graphql.org) est une brique centrale pour MYTF1. Nous l'utilisons comme source de données des différents fronts. Les avantages sont les suivants :

- pas de service spécifique par écran, chaque front peut requêter ce dont il a besoin uniquement
- le GraphQL joue le rôle d'API gateway, c'est lui qui rassemble les données des micro-services sous jacents
- contrat d'interface auto documenté entre le back et les fronts

### Les bases de données

Nous utilisons [MongoDB](https://www.mongodb.com) et [PostgreSQL](https://www.postgresql.org) pour les bases de données de référence.
Ces données sont ensuite dénormalisées dans des cluster [Redis](https://redis.io). Nous avons adopté une architecture "event driven" en nous appuyant sur [Kafka](https://kafka.apache.org) pour maintenir une synchronisation constante entre ces bases de données.

## Web

Le front MYTF1 repose sur une SPA en [Reactjs](https://fr.reactjs.org) et un serveur expressjs pour le SSR (server side rendering) pour assurer un bon référencement. Nous utilisons react car la partie SSR est éprouvée ainsi que le large écosystème open source.

### Applicatif front :

La stack est principalement axée sur les performances et le SEO. React via React-dom/server permet de généré du HTML cote serveur qui sera ensuite "hydrater" cote client pour assurer une bonne UX.

**TypeScript :**
Nous utilisons fortement [Typescript](https://www.typescriptlang.org) pour que l'appropriation du code soit rapide, de plus, et comme tests "statiques" pour s'assurer que l'on envoie bien le bon type de données aux fonctions

**GraphQL & Apollo :**
[Apollo](https://www.apollographql.com) nous permet de consommer l'API GraphQL et fonctionne aussi cote serveur.
Via GraphQL code generator, on genere toute nos composants/hooks Apollo via nos queries/mutations en typescript, ce qui permet encore une fois de s'assurer que ces requêtes sont valides.

**Helmet :**
[Helmet](https://github.com/staylor/react-helmet-async) est la librairie (react-helmet-async et non pas react-helmet) nous permet d'enrichir au fur et a mesure des composants rendu les balises metas qui aident a la compréhension des robots de moteurs de recherches du contenu de nos pages.

### Performance et Qualité :

**Webpack & Lazyloading :**
_brouillon : Nous utilisons [Webpack](https://webpack.js.org) pour nos ressources statiques…( modules, chunk...), (code splitting) splitter nos builds en plusieurs paquets, chargé à la demande (à compléter ....)_

**Jest / React Testing Library :**
Nous utilisons [Jest](https://jestjs.io) pour nos tests unitaires, ce qui nous permet de vérifier la non-régression du front MYTF1, tout au long du développement de nos features et de garantir la fonctionnalité de composants complexes.

## APP

Les application mobiles sont natives et codées en [Swift](https://swift.org) (iOS) et [Kotlin](https://kotlinlang.org) (Android).

TODO a voir avec Mohamed

## IPTV

Les techno utilisées sur l'iptv sont très variées et dépendent du modèle de box, globalement on retrouve trois familles:

- **HTML/JS**, principalement SFR et Orange
- **QT/QML**, très utilisé par Free
- **Android**, notamment sur Bouygues et Free

Dans le cas des box Android, un moteur QT/QML tourne dans l'application afin de réutiliser le code QT/QML.

## Le player

Nous developpons notre propre player pour différentes plateformes :

- le Web (JS)
- iOS (Swift)
- Android (Kotlin) pour les applications mobiles et certaines box opérateur

TODO a voir avec Guillaume

## Le CMS

Nous avons également développé un CMS maison qui permet à l'équipe édito d'animer le contenu de MYTF1. Il est développé en [Vue.js](https://vuejs.org).

# L'architecture backend

## Event driven

TODO un schéma de l'archi globale + explication event driven

# La performance et la QOS

La performance est un sujet critique pour MYTF1. Lors de grands évènements tels que la coupe du monde de football ou de la diffusions de programmes fédérateurs comme The Voice ou Koh-Lanta, le service doit tenir la charge face à plusieurs centaines de milliers d'utilisateurs simultanés qui peuvent se connecter à seulement quelques minutes d'intervalle pour, par exemple, suivre un live.

## La gestion du cache

### Les niveaux de cache

Pour tenir la charge, différents niveaux de caches sont utilisés:

- le cache apollo, permet d'avoir un cache cohérent au niveau d'un front pour un utilisateur (données privées et publiques)
- les CDN ([cloudfront](https://aws.amazon.com/cloudfront/)) permettent de mettre en cache les données publiques et le flux vidéo coté backend
- les base de données Redis permettent de mettre en place une session pour les données privées ou d'avoir un vision dénormalisée des données publiques

### GraphQL et les persisted queries

Historiquement, MYTF1 s'appuie beaucoup sur le cache des CDN. En effet, une grande partie des données sont publiques, notamment toutes les informations liées au catalogue vidéo. Avec GraphQL il n'est pas naturel d'utiliser ce type de cache. Par nature, la combinatoire des requêtes et le fait de pouvoir mélanger données privées et publiques ne permet pas de cacher.

Nous avons réutilisé la mécanique de "persisted queries" d'apollo que nous avons légèrement modifiée. Celle-ci consiste à sauvegarder le body de la requête dans une base de données. Le client apollo n'envoit qu'un ID (en général un hash du body de la requête) dans une requête GET pour intérroger le GraphQL. De cette façon il est plus simple de mettre en place du cache CDN. Ce fonctionnement est activé en PROD, sur les environnements de developpement il reste desactivé pour garder toute la souplesse de GraphQL. Les body des requêtes sont sauvegardés en base de données au moment du build par notre CI/CD.

![Diagramme explicatif du fonctionnement des persisted queries](images/persisted-queries.svg "Diagramme explicatif du fonctionnement des persisted queries")

Les avantages sont les suivants:

- le GraphQL est vérouillé en PROD on ne peut pas explorer le schema avec des requêtes non présentes dans le code
- le GraphQL n'est pas vérouillé sur les environnment hors PROD, on garde donc toute la souplesse de l'API pour les developpeurs
- les requêtes publiques sont connues à l'avance et associées à un TTL qui est transmis dans un header Cache-Control et donc exploités par les CDN

### Le cache de données privées

Il est necessaire, pour exploiter efficacement le cache, de séparer les call qui présentent des données privées. Une partie de l'intelligence est déportée sur le front. Par exemple, la liste des vidéo mises en favoris par un utilisateur, est récupérée en début de session. L'écran est construit à partir des données publiques, les favoris précédemments récupérés viennent enrichir l'écran, en affichant un coeur sur la vignette vidéo. Cette logique est plus complexe mais garanti que même lors d'une indisponibilité du backend, le front pourra afficher un écran dans un mode dégradé (à partir du stale cache CDN).

<!--more-->

Nous nous appuyons également sur des cache Redis, en début de sessions, les données d'un utilisateur sont remontées dans ce type de cache, les call ultérieurs sont alors moins couteux.

<!--more-->

Enfin, l'utilisations d'un token JWT, permet d'identifier un utilisateur sans sollicitation d'un service ou d'une base de données, le token est transmis du GraphQL aux micro services sous jacents qui peuvent alors vérifier eux même la validitée du token (par simple vérification de la signature).

## Le monitoring

Nous utilisons l'outil [Grafana](https://grafana.com), associé à [Prometheus](https://prometheus.io), pour faire le monitoring de nos applicatifs et de notre infrastructure. [Graylog](https://www.graylog.org) permet quand à lui de récupérer les logs d'execution. Nous enviseagons d'utiliser [Jaeger](https://www.jaegertracing.io) qui permettrait de restituer une vision cohérente des services en terme de monitoring et de logs.

![Exemple de dashboard Grafana](images/grafana-graphql.png "Exemple de dashboard Grafana")

# La vidéo

TODO a voir avec DLC

# Le cloud et le devops

Nos services sont déployés sur des clusters kubernetes. Nous utilisons le cloud [AWS](https://aws.amazon.com/) pour héberger ces clusters (service [EKS](https://aws.amazon.com/eks/)).

## L'infra as code

TODO AWS + Terraform

## L'auto scaling

TODO fonctionnement + explication linkerD

## La CI/CD

TODO explication jenkins + tests non reg / replayer + spinnaker
