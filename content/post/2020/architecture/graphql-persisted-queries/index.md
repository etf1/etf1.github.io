---
title: GraphQL et persisted queries
date: 2020-11-07T10:00:00
hero: /post/2020/architecture/graphql-persisted-queries/images/hero.jpg
excerpt: La gestion du cache et l'utilisation des persisted queries avec graphQL
authors:
  - rpinsonneau
---

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

