---
title: Mise en place de notre système d'abonnement multi-plateforme
date: 2022-03-14T08:00:00
hero: /post/2022/architecture/abonnement-multi-plateforme/images/hero.jpg
excerpt: Découvrez comment nous avons mis en place une gestion d'abonnements multi-platforme pour notre offre MYTF1 MAX.
authors:
  - vcomposieux
description: "Découvrez comment nous avons mis en place une gestion d'abonnements multi-platforme pour notre offre MYTF1 MAX"
---

## Présentation de l'architecture

Vous l'avez peut-être remarqué si vous êtes un utilisateur MYTF1, nous avons récemment sorti l'offre [MYTF1 MAX](https://www.tf1.fr/offre-max) : une offre payante vous permettant de bénéficier de plus de contenu et de fonctionnalités étendues.

Afin de mettre en place cette nouvelle fonctionnalité sur le site web mais aussi sur les applications mobiles MYTF1, nous avons mis en place une stack technique permettant de prendre des abonnements depuis plusieurs platformes :

* Abonnement sur le web, nous utilisons [Stripe](https://www.stripe.com) pour la facturation,
* Abonnement sur Android via le [Google Play Store](https://play.google.com/store),
* Abonnement sur iOS via l'[App Store](https://www.apple.com/fr/app-store/) d'Apple.

Pour les abonnements pris depuis le web, plusieurs solutions ont été envisagées. Notre choix s'est porté sur la solution Stripe : un des leaders dans le domaine du paiement en ligne permettant de gérer des abonnements. Aussi, Stripe dispose de solutions d'intégration très complètes (APIs, SDK front).

L'architecture technique se décompose comme suit :

![Architecture d'abonnement](images/architecture.svg#darkmode "Architecture d'abonnement")

Dans chacun des cas, nos applications web et mobiles contactent notre gateway GraphQL qui communique ensuite via le protocole [gRPC](https://grpc.io/) avec les différents micro-services.

gRPC nous permet d'avoir des appels serveur-serveur normés grâce à une définition des APIs et des messages qui seront échangés avec [Protocol Buffers](https://developers.google.com/protocol-buffers/).

Dans le cas des abonnements, il s'agit d'un micro-service nommé `purchase`. Il s'occupe de toute la gestion des abonnements MYTF1 MAX.

Cette brique a donc plusieurs rôles :

* Exposer une API permettant de souscrire à une offre d'abonnement sur le web (via Stripe),
* Exposer une API permettant de vérifier un abonnement souscrit depuis un store mobile (Apple, Google),
* Exposer un endpoint permettant de traiter les webhooks envoyés par les différents providers (Stripe, Apple, Google).

Avant toute chose, nous avons créé un **modèle unifié**. Pour cela, comme vu précédemment, nous utilisons [Protocol Buffers](https://developers.google.com/protocol-buffers/) pour décrire, à la fois nos APIs gRPC mais également le modèle de données unifié suivant :

```js
message Purchase {
    enum Status {
        INACTIVE = 0;
        TRIAL = 1;
        ACTIVE = 2;
        CANCELED = 3;
        EXPIRED = 4;
        ON_GRACE = 5;
    }

    enum Periodicity {
        UNDEFINED = 0;
        WEEKLY = 1;
        MONTHLY = 2;
        YEARLY = 3;
    }

    enum Type {
        UNDEFINED = 0;
        MAX = 1;
    }

    enum Provider {
        UNDEFINED = 0;
        STRIPE = 1;
        APPLE = 2;
        GOOGLE = 3;
    }

    Provider provider = 1;       // Fournisseur (Stripe, Apple, Google)
    Type type = 2;               // Type de l'abonnement (MYTF1 MAX)
    Status status = 3;           // Statut actuel de l'abonnement (actif, annulé, ...)
    Periodicity periodicity = 4; // Périodicité de l'abonnement (mensuel, annuel)
    float amount = 5;            // Montant actuel de l'abonnement ou de l'achat
    float upcomingAmount = 6;    // Montant du prochain prélèvement de l'abonnement
    string userId = 7;           // Identifiant de l'utilisateur MYTF1
    string providerId = 8;       // Identifiant de l'abonnement ou de l'achat chez le provider
    string providerPriceId = 9;  // Identifiant du prix chez le provider
    string label = 10;           // Libellé de l'abonnement ou de l'achat
    int64 expireAt = 11;         // Date d'expiration de l'abonnement
    int64 createdAt = 12;        // Date de création de l'abonnement ou de l'achat
    int64 updatedAt = 13;        // Date de mise à jour de l'abonnement ou de l'achat
    int64 lastEventAt = 14;      // Date du dernier événement traité pour cet abonnement ou achat
}
```

Ce modèle étant défini, il nous permet maintenant de stocker tous les abonnements ou achats sous le même format, quel que soit le fournisseur : Stripe pour le web, Apple ou encore Google.

Concrètement, voici les différents formats retournés par les trois fournisseurs pour un abonnement :

* Stripe : un JSON formatté comme défini ici : [https://stripe.com/docs/api/subscriptions/object](https://stripe.com/docs/api/subscriptions/object),
* Apple : un JSON formatté comme défini ici : [https://developer.apple.com/documentation/appstorereceipts/responsebody](https://developer.apple.com/documentation/appstorereceipts/responsebody),
* Google : un JSON encodé en base64 et formatté comme défini ici : [https://developers.google.com/android-publisher/api-ref/rest/v3/purchases.subscriptions](https://developers.google.com/android-publisher/api-ref/rest/v3/purchases.subscriptions).

Chacun ayant des formalismes différents, il était important de passer par cette première étape d'uniformisation.

D'ailleurs, vous remarquerez le champ `providerId` dans le modèle unifié. Celui-ci nous sert d'identifiant unique pour identifier un abonnement et contient la valeur suivante :

* Dans le cas de Stripe : il s'agit du champ `id` de l'objet [Subscription](https://stripe.com/docs/api/subscriptions/object),
* Dans le cas d'Apple, il s'agit du champ [original_transaction_id](https://developer.apple.com/documentation/appstorereceipts/original_transaction_id),
* Dans le cas de Google, il s'agit du champ [purchaseToken](https://developers.google.com/android-publisher/api-ref/rest/v3/purchases.subscriptions/get).

Nous avons ensuite commencé à éprouver ce modèle avec le parcours le plus complexe : celui de Stripe sur le web car nous gérons nous-même les workflows de création des abonnements, annulation et changement de périodicité, ce qui n'est pas le cas côté application mobile car les abonnements sont gérés par les stores.

## Prise d'un abonnement sur le web

Afin de prendre un abonnement via la plateforme Stripe, il y a deux actions à effectuer :

1. Préparer un objet [SetupIntent](https://stripe.com/docs/payments/setup-intents), permettant de configurer un moyen de paiement pour des paiements et renouvellements futurs (c'est le cas de nos abonnements, on appelle ça le **off-session**), contrairement à l'API [PaymentIntent](https://stripe.com/docs/payments/payment-intents) qui vous permet de réaliser des prélèvements directs (**on-session**),
2. Demander à Stripe la création d'un client ([Customer](https://stripe.com/docs/api/customers/object)) puis d'un abonnement ([Subscription](https://stripe.com/docs/api/subscriptions/object)) relié à l'objet précédemment créé.

Le workflow est le suivant :

![Diagramme de séquence - appels web](images/diagram-web.svg#darkmode "Diagramme de séquence - web")

1. L'utilisateur **saisit ses informations bancaires** sur le front MYTF1 via le SDK [PaymentElement](https://stripe.com/docs/payments/payment-element),
2. Le front récupère alors du SDK Stripe un identifiant de **méthode de paiement** (`paymentMethodID`) qu'il transfère dans la mutation GraphQL nommée `webSetupIntent`. Nous lui retournons si la création de celui-ci s'est bien déroulée et/ou si une authentification 3D Secure est nécessaire,
3. Si une authentification **3D Secure** est nécessaire, le front s'en charge via le SDK de Stripe,
4. Lorsque tout est prêt : le front web peut donc appeler une mutation GraphQL `webSubscriptionCreate` en passant l'**identifiant du tarif** demandé par l'utilisateur (prix mensuel ou annuel).
5. Lorsque le micro-service purchase reçoit l'appel API, il s'occupe donc de **créer le compte client** ainsi que l'**abonnement** chez Stripe et stocke les données d'abonnement dans une table [AWS DynamoDB](https://aws.amazon.com/fr/dynamodb/).

Une fois l'abonnement créé, le front web n'a plus qu'à authentifier à nouveau l'utilisateur afin que ses droits d'abonnement soient pris en compte.

Si vous souhaitez plus d'informations sur la prise d'abonnement via Stripe, je vous conseille le guide suivant : [https://stripe.com/docs/billing/subscriptions/overview](https://stripe.com/docs/billing/subscriptions/overview).

## Prise d'un abonnement sur les stores mobile

Côté mobile, que ce soit pour Apple ou Google : le workflow est globalement le même côté backend MYTF1.

![Diagramme de séquence - appels mobile](images/diagram-mobile.svg#darkmode "Diagramme de séquence - mobile")

Les étapes sont les suivantes :

1. L'application mobile va demander la création d'un **receipt** via le SDK du fournisseur : **StoreKit** pour Apple et le **SDK Android** pour Google.
2. Une fois ce receipt créé, il nous est envoyé côté backend dans une mutation GraphQL `mobileSubscriptionVerify` permettant d'effectuer la **vérification** de celui-ci (en mode server-to-server) et stockons les informations dans notre base de données en cas validation de celui-ci.
3. Uniquement une fois le receipt validé côté serveur et après s'être assuré que l'utilisateur ne bénéficie pas déjà d'un abonnement, les applications mobiles vont alors **confirmer la prise d'abonnement** via le SDK mobile.

Pour la vérification des receipts, nous utilisons les APIs suivantes :

* Android : [https://developers.google.com/android-publisher/api-ref/rest/v3/purchases.subscriptions/acknowledge](https://developers.google.com/android-publisher/api-ref/rest/v3/purchases.subscriptions/acknowledge)
* iOS : [https://developer.apple.com/documentation/appstorereceipts/verifyreceipt](https://developer.apple.com/documentation/appstorereceipts/verifyreceipt)

Comme pour le web, une fois toutes les étapes terminées, l'application mobile procède à une nouvelle authentification de l'utilisateur afin de lui attribuer ses droits d'abonnement.

## Gestion des événements

Une fois les abonnements actifs, il nous faut également surveiller les mises à jour de ceux-ci :

* **Mettre à jour** les informations sur l'objet `Purchase` lorsque l'abonnement est renouvelé (conciste globalement à étendre la date d'expiration),
* **Annuler l'abonnement** de l'utilisateur à la fin de la période courante lorsqu'un utilisateur demande une annulation depuis le web ou depuis son interface App Store ou Google Play Store,
* **Expirer** l'abonnement en cas de défaut de paiement.

Pour cela, nous avons mis en place des endpoints HTTP pour chaque fournisseur : `/webhook/<provider>` : pour récupérer les événements provenant d'un fournisseur donné.

Nous récupérons alors des données JSON, bien évidemment encore différentes pour chaque provider.

Pour plus d'informations, voici les documentations techniques de la gestion d'événements chez les différents fournisseurs :

* Apple : [https://developer.apple.com/documentation/appstoreservernotifications](https://developer.apple.com/documentation/appstoreservernotifications)
* Google : [https://developer.android.com/google/play/billing/rtdn-reference](https://developer.android.com/google/play/billing/rtdn-reference)
* Stripe : [https://stripe.com/docs/billing/subscriptions/webhooks](https://stripe.com/docs/billing/subscriptions/webhooks)

Si vous avez lu notre [article précédent](https://tech.tf1.fr/post/2021/architecture/migration-vers-kafka/), vous savez que nous utilisons désormais [Kafka](https://kafka.apache.org/) pour la gestion de nos événements backend. C'est aussi le cas pour les événements que l'on reçoit des fournisseurs.

![Architecture des événements](images/event.svg#darkmode "Architecture des événements")

En effet, dès qu'un événement arrive sur le endpoint HTTP, nous produisons simplement un message dans un topic Kafka. Cela est très rapide et nous permet d'éviter d'avoir un temps de traitement trop long au niveau des webhooks, pour ne pas surcharger la plateforme en cas de forte influence.

Afin de **garder un historique** des événements reçus, nous stockons également chaque message dans une table d'audit DynamoDB.

### Traitement des événements via Kafka

Le message `Event` produit, également défini sous le format Protocol Buffers, est le suivant :

```js
message Event {
    enum Provider {
        UNDEFINED = 0;
        STRIPE = 1;
        APPLE = 2;
        GOOGLE = 3;
    }

    Provider provider = 1;
    int64 timestamp = 2;
    bytes data = 3;
    string signature = 4;
}
```

Nous avons ici uniquement les informations nécessaires au traitement de ce message : le fournisseur, la date et l'heure à laquelle nous avons reçu le message, le message lui-même (format original) et une signature envoyée par le fournisseur afin de s'asurer qu'il est bien autorisé à nous envoyer des événements.

Ce message Kafka est ensuite traité par un consommateur, **de façon asynchrone**, et ce consommateur dispatche le message en fonction du fournisseur.

```go
switch event.Provider {
case provider.Apple:
  err = e.appleHandler.Handle(ctx, event, options...)

case provider.Google:
  err = e.googleHandler.Handle(ctx, event, options...)

case provider.Stripe:
  err = e.stripeHandler.Handle(ctx, event, options...)

default:
  logger.Error("Event consumer: Unknown provider", logging.Field("event", event))
  return
}
```

Ainsi, chaque fournisseur dispose de sa logique qui lui est propre et côté architecture logicielle, ils sont isolés les uns des autres.

Dans un souci de **robustesse** de traitement de ces événements, trois points sont importants à noter du côté de ces consommateurs afin de limiter tout type d'erreur :

1. Nous avons mis en place un **système de lock** avec [Redis](https://redis.io/), qui expire au bout d'une durée définie (TTL). Cela nous permet de ne nous assurer qu'aucuns événements ne puissent se chevaucher.

2. En cas d'erreur lors du traitement d'un événement, nous avons également un **système de retry** de message Kafka basé sur un algorithme [backoff exponentiel](https://en.wikipedia.org/wiki/Exponential_backoff) et au bout d'un nombre d'essai défini, le message termine dans un topic type "dead letter", ce qui nous permet d'éventuellement pouvoir les analyser et/ou les re-jouer.

3. Nous stockons également sur l'objet `Purchase`, défini plus haut, un champ `lastEventAt` nous permettant de connaitre la date du dernier événement traité, afin de **ne pas avoir un événement plus vieux** qui viendrait être traité après un plus récent.

### Particularités

#### Google Play Store

La première particularité concerne le Google Play Store : afin de recevoir les événements concernant les abonnements, il faut [mettre en place un topic dédié](https://developer.android.com/google/play/billing/getting-ready) sur le service [Google Cloub Pub/Sub](https://cloud.google.com/pubsub/) et autoriser le service account `google-play-developer-notifications@system.gserviceaccount.com` à publier des messages dans ce topic.

Le nom du topic fraichement créé doit ensuite être renseigné dans la console Google Play.

Nous avons ensuite pris le parti de créer un abonnement (au sens Pub/Sub) sur ce topic qui va consommer automatiquement les messages et les pousser sur notre endpoint HTTP.

De cette façon, nous pouvons les traiter comme les autres fournisseurs.

#### App Store

Une autre particularité concerne les notifications Apple : si vous vous lancez dans un système d'abonnement, étudiez bien le formalisme des messages en fonction des différents cas d'utilisation.

La documentation n'est en effet pas toujours bien à jour et certaines valeurs ne sont pas forcément renseignées correctement dans le message, en fonction des événements reçus : nous avons donc passé beaucoup de temps à tester et évaluer les données reçues.

## Conclusion

Nous avons passé beaucoup de temps à défricher toutes les informations chez les différents fournisseurs afin de pouvoir mettre en place un modèle commun ainsi qu'à comprendre les différents workflows mobiles mais nous sommes finalement parvenus à mettre en place un système qui répond à notre besoin.

Ce service est désormais actif en production depuis quelques mois et n'avons pour le moment pas rencontré de souci majeur depuis sa mise en place.

Nous allons très certainement être amenés à étoffer ce service avec de nouvelles fonctionnalités, nous pourrons donc vous tenir informés des évolutions par la suite.
