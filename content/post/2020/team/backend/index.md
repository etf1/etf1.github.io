---
title: L'équipe Backend
date: 2020-09-20
hero: /post/2020/architecture/presentation/images/hero.jpg
excerpt: Welcome to the other side
authors:
  - ldechoux
---

## Qui sommes nous ?

L’équipe backend a pour objectif de répondre aux problématiques suivantes :
- Gérer la mise en ligne et l’animation éditoriale de notre contenu
- Stocker et restituer les données utilisateurs (historique et progression de lecture, programme favoris, bookmakrs etc..)
- Exporter/partager notre catalogue de contenu avec nos partenaires (FAI, Salto, etc...)
- Fournir des API aux front pouvant supporter de forte charges

Elle est composée d’une dizaine de personnes ayant des profils (développeur, product owner, lead tech) et des niveaux d’expérience (débutant, expérimenté, stagiaire, alternant) différents. Depuis 2018 nous avons fait le choix d’investir fortement dans le langage Go qui représente aujourd’hui la quasi intégralité de notre base de code.

## Architecture et technologies

Nous avons fait le choix d’une architecture micro-services. Les différentes composantes métier sont réparties en différents services, qui communiquent principalement via GRPC.
Voici une liste non exhaustives de nos différentes briques métier :
- CMS API : dédiée à l’animation éditoriale de notre contenu
- Catalog API : dédiée à la gestion de notre catalogue de contenu
- User API : dédiée à la gestion des données utilisateur
- Reco API : dédiée à la recommandation de contenu
- SEO API : dédiée  aux problématiques SEO
- Auth API : dédiée à l’identification de nos utilisateurs

Pour les applications MYTF1, nous avons fait le choix d’exposer à travers une API GraphQL une vision consolidée de ces services.
En effet, l’API GraphQL agit comme une API Gateway et se charge d’exposer un modèle de données cohérent et unifié qui répond aux besoins exprimés par les équipes produit/métier.
Elle a été conçue et créée avec une vision multi-écran et doit être capable de fonctionner aussi bien pour nos applications Web, que pour les applications mobiles ou encore les box opérateurs.

Pour répondre aux différents challenges qui nous sont adressés, nous avons choisies les technologies suivantes : 
- Langages : Go, Java
- Base de données : MongoDB, Elasticsearch, DynamoDB, Redis
- Event/Message broker : RabbitMQ, Kafka
- Frameworks Fronts : Vue.js, React
- Formats des API : GRPC, GraphQL, Rest
- Infrastructure : AWS, K8S, Docker, Linkerd
- Monitoring : Prometheus, Grafana
- CI/CD : Jenkins

Cette liste, bien que fournies, peut-être amenée à évoluer en fonction des futurs besoins ou challenges qui se présenteront.
En effet, une des forces de l’équipe et de savoir se remettre en question et faire table rase du passée. C’est ce que nous allons voir dans le paragraphe suivant.

## Un peu d’histoire

### 2018 : Nouvelle expérience IPTV

Début 2018 MYTF1 se lance dans un projet radical de transformation de l’expérience utilisateur sur les box opérateurs (IPTV).
Un cahier des charges est définit et de nouveaux enjeux apparaissent :
- Permettre une navigation fluide du contenu
- Possibilités d’éditorialisation avancées
- Recommandation de contenu personnalisée
- Gestion de l’historique et de la reprise de lecture
- Mise en favoris des programmes

À ce stade, nous sentons bien que le socle technologique existant a atteint ses limites et qu’il faut envisager des changements radicaux.
Une petite équipe est montée pour relever ce défi. Elle deviendra plus tard l'équipe Backend.

Nous ferons alors plusieurs choix structurants :
- Mise en place d’une API GraphQL pour exposer les données au front
- Utilisation de cache in-memory au niveau de l'API GraphQL
- Découpe des différents besoins en micro-services GRPC dédiés
- Dénormalisation des données catalogue et éditoriales dans elasticsearch
- Gestion de la session utilisateur dans redis
- Ecriture asynchrone des données utilisateur
- Utilisation d’un token JWT pour identifier l’utilisateur

Dès le début, même si le projet est centré sur l’IPTV, nous avons la volonté de créer un nouveau socle technique qui sera capable d’adresser tous les écrans (Web et applications mobiles compris).
En effet, à date, il n’y a pas réellement de socle commun et chaque écran est traité séparément (avec ses propres solutions, technologies et choix techniques).
Nous avons toujours garder cette idée dans un coin de notre tête et elle a piloté beaucoup de nos décisions.

S’en suit alors une phase intensive de conception et développement, en effet les délais sont serrés (mise en production attendue pour Juin 2018) et nous avons beaucoup de travail à abattre.
L'application sera finalement lancée en Juillet 2018, pour les utilisateurs de freebox.

La première mouture de notre nouvelle architecture backend est enfin prête :

![2018 - Première version de la nouvelle architecture backend](images/archi_2018.svg "2018 - Première version de la nouvelle architecture backend")
