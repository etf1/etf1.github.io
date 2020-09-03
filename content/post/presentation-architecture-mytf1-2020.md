---
title: Présentation architecture MYTF1
date: 2020-09-01
hero: /images/hero-2.jpg
excerpt: Une présentation technique de la stack MYTF1
authors:
  - Rémy PINSONNEAU

---

# MY TF1 quésaco ?

MY TF1 est le service de replay qui permet à nos utilisateurs de voir ou revoir en streaming les programmes des chaînes du groupe: TF1, TMC, TFX, TF1 Séries Films, LCI. Il est disponible sur la plupart des écrans: Web, Mobile (iOS, Android) et sur l'IPTV (box des opérateurs). Ce service est gratuit et est rémunéré par la publicité.

Dans son ensemble ce service répond à un spectre varié de sujets:
* le streaming vidéo et l'encodage des contenus
* le ciblage de la publicité
* la recommendation de contenu (la data)
* l'animation des contenus (l'édito)

# Les techno utilisées

MY TF1 existe depuis 2011, il a été depuis plusieurs fois refondu from scratch.

  Période          | Technos
-------------------|--------------
2011 - 2015        | PHP, MySQL
2015 - 2019        | NodeJS, PostgreSQL
2019 - Aujourd'hui | Go, GraphQL, gRPC, MongoDB, Kafka, Redis, Kubernetes


## Backend

Aujourd'hui le backend est constitué d'un ensemble de micro services en **Go**. Après la période NodeJS, nous avons décidé de retourné à un language fortement typé. La façon dont Go gère la concurrence (go routine) permet de tenir le fort traffic de MY TF1 et est particulièrement adapté a un écosysteme **kubernetes**:
* emprunte mémoire faible
* démarage rapide (binaire compilé)
* taille des images docker réduite
* idéal pour des services HTTP / GRPC

C'est également un language rapide à apprendre.

### GraphQL

GraphQL est une brique centrale sur MY TF1. Nous l'utilisons comme API pour remonter les données sur les différents fronts. Les avantages sont les suivants:
* pas de service spécifique par écran, chaque front peut requêter ce dont il a besoin uniquement
* le graphQL joue un rôle d'API gateway, c'est lui qui rassemble les données des micro services sous jacent
* contrat d'interface auto docummenté entre le back et les fronts

### Les bases de données

Nous utilisons **MongoDB** et **PostgreSQL** sur les bases de données de référence.
Ces données sont ensuite dénormalisées dans des cluster **Redis**. Nous avons adopté une architecture "event driven" en nous appuyant sur **Kafka** pour maintenir une syncrhonisation entre ces bases de données.

## Web

Le site web est aujourd'hui une SPA en **React**.

## APP

Les application mobiles sont natives et codés en **Swift** (iOS) et **Kotlin** (Android).

## IPTV

Les techno utilisées sur l'iptv sont très variées et dépendent du modèle de box, globalement on retrouve trois familles:
* HTML/JS, principalement SFR et Orange
* QT/QML, très utilisé par Free
* Android, notemment sur Bouygues et Free

## Le player

Nous developpons notre propre player sur trois langages : JS (web), Swift et Kotlin pour le mobile et certaines box opérateur.

TODO

## Le CMS

Nous développons également un CMS maison, qui permet à l'équipe édito d'animer le contenu de MY TF1. Il est développé en VueJS.

# L'architecture backend

## Event driven

TODO un schéma de l'archi globale + explication event driven

# La performance et la QOS

La performance est un sujet critique pour MY TF1. Lors de grand évènements tel que la coupe du monde de football ou des programmes fédérateurs comme The Voice ou Koh-Lanta, le service doit tenir face à plusieurs centaines de milliers d'utilisateurs qui peuvent se connecter à seulement quelques minutes d'intervalle pour voir le live.

## La gestion du cache

### Les niveaux de cache

TODO Apollo, CDN, Redis

### GraphQL et les persisted queries

TODO persited queries

### Le cache de données privées

TODO séparation des requêtes publique / charger les données user dans Redis / token JWT

## Le monitoring

TODO Grafana + prometheus + graylog

# La vidéo

TODO a voir avec DLC

# Le cloud et le devops

Tous nos services sont déployés sur des clusters kubernetes. Nous utilisons le cloud AWS pour héberger ces clusters (service EKS).

## L'infra as code

TODO AWS + Terraform

## L'auto scaling

TODO fonctionnement + explication linkerD

## La CI/CD

TODO explication jenkins + tests non reg / replayer + spinnaker