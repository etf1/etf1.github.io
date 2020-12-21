---
title: Notre utilisation du langage GraphQL pour nos UI BACKEND
date: 2020-12-21T09:00:00
hero: /post/2020/architecture/backend-graphql-ui/images/graphql.png
excerpt: Nos différentes manières d’ajouter GraphQL à nos applications Intranets
authors:
  - eferte
description: "Nos différentes manières d’ajouter GraphQL à nos applications Intranets côté BACKEND"
---

## Avant propos

Nous avons déjà eu l’occasion de montrer que les échanges de données entre le Front web MYTF1 et les Api back, s’effectuent en Graphql (voir: [GraphQL et les persisted queries](https://tech.tf1.fr/post/2020/architecture/graphql-and-persisted-queries/)). Plus encore, le langage Graphql est également majoritairement utilisé dans nos applications intranet (CMS du site Web MYTF1, Administration, Configurations, Gestion des droits). Nous proposons ici, de vous présenter nos différentes manières d’ajouter graphql à nos applications Intranets.


## Pourquoi GraphQL

![Logo GraphQL](images/graphql.png "GraphQL")

D’abord pour la souplesse apportée par GraphQL. GraphQL met de l’huile dans les rouages Back - Front. Côté back, nos développeurs mettent à disposition les données métier de manière suffisamment exhaustive. Côté Front, une fois l’api GraphQL disponible, les développeurs peuvent parcourir la documentation à l’aide de Graphiql et y piocher uniquement les champs dont ils ont besoin. 

Ensuite, GraphQL normalise la composition de requêtes/mutations à l’aide de Fragments. Ce qui évite les répétitions parfois sources de bugs.

> “Don’t repeat yourself, uses Graphql Fragments”
> Einstein

Un autre avantage de GraphQL est l’utilisation d’alias permettant plus de souplesse encore pour transformer un model serveur vers un modèle client. Cela permet, par exemple de découper un tableau en propriétés métiers côté client : 

```javascript
webProps: streamProperties(forScreen: WEB) {
    ...PrgStreamProps
 }
appProps: streamProperties(forScreen: APP) {
    ...PrgStreamProps
} 
iptvProps: streamProperties(forScreen: IPTV) {
    ...PrgStreamProps
}
```

Bien entendu, même si dans nos projets nous n’avons pas eu besoin de plus, le langage GraphQL est encore plus riche, on pourra s’en rendre compte ici: [2 More GraphQL Concepts](https://www.howtographql.com/advanced/2-more-graphql-concepts/).

La documentation Graphiql est également un vrai plus pour parcourir les modèles de requêtes et de mutations possibles. Comme ce composant est en React et en open source, nous avons pu le customiser un peu pour y ajouter un header d’authentification ou encore la prévisualisation pour certaines URL d’images.

Par chance, le langage GraphQL est relativement simple à implémenter côté back et encore plus simple côté front, avec des librairies riches et abouties déjà disponibles.

## Apollo

![Logo Apollo](images/apollo.png "Apollo")

Lorsqu’on parle de GraphQL, aujourd’hui, il n’est pas rare de lui associer la librairie Apollo tout simplement parce que cette implémentation GraphQL est complète, riche et adaptée à tous les frameworks (en ce qui nous concerne Vue et React) par ailleurs Apollo dispose déjà des définitions de types pour Typescript, la majorité de nos projets React sont en Typescript.

### La gestion du cache de données

L’un des nombreux avantages d’Apollo est la gestion du cache, cela simplifie la mise à jour des données. A bien des égards, le cache Apollo peut même rendre inutile l’ajout d’une librairie comme Redux (pour React). Si l’on y ajoute le “double-binding” en Vue, on obtient des composants quasiment purement déclaratifs. Il suffit de rattacher le modèle de requête Apollo directement au composant Vue et le composant est “auto-magiquement” mis à jour. On peut aussi regretter le côté non contrôlable du double-binding, mais rappelons, qu’il est toujours possible d’appeler directement des requêtes sans passer par un composant visuel (ce aussi bien pour React que pour Vue).

A noter que le comportement du cache est impacté par un paramètre “fetchPolicy” directement configurable pour chaque requête : 

```javascript
apolloClient.query({
        query: GET_FOLDER_INFOS,
        fetchPolicy: "no-cache",
        variables: { path },
 })
```

Nous avons eu à plusieurs reprises besoin d’ajouter ce paramètre “fetchPolicy” pour être sûr que notre composant se rafraichissait bien comme attendu.

### Les composants UI Apollo et les requêtes depuis le client Apollo 

Pour nos applications React, nous avons aussi bien utilisé les composants disponibles dans la librairie Apollo, que des appels directs en utilisant le client Apollo. Avec l’apparition des hooks, il serait toutefois assez logique que notre utilisation de composants pour représenter des requêtes GraphQL décline.

### Ajout des headers aux requêtes Apollo

En voulant sécuriser par tokens JWT, certaines de nos applications intranet, nous avons utilisé un peu du middleware Link d’Apollo. Dans la terminologie Apollo, un Link est une  fonction qui prend une opération et retourne un observable. 

![Apollo Link](images/apollo-link.png "Apollo Link")


Les Links peuvent être enchaînées. Nous avons utilisés en particulier le Link “createHttpLink” avec l’opération “setContext” pour ajouter, lors de la création du client Apollo, le header d'autorisation avec le token en quelques lignes:
```javascript
    …

    const httpLink = createHttpLink({
      uri: url,
    });

    const authLink = setContext((_, { headers }) => {
      return {
        headers: token ? {
          ...headers,
          authorization: `Bearer ${token}`,
        }: {
          ...headers
        },
      };
    });

    const client =  new ApolloClient({
      link: authLink.concat(httpLink),
      cache: new InMemoryCache(),
    }); 
    …
```

(A noter que l’on peut également passer un context spécifiquement pour une requête en particulier).


Pour en savoir plus sur le middleware Link d’Apollo : [Apollo Link Overview](https://www.apollographql.com/docs/link/overview/)

Bref, Apollo est une librairie à la fois riche et de haut niveau, tout en restant hautement configurable. Mais as-t-on toujours besoin d’Apollo pour faire du GraphQL ?

## Début d’implémentation d’un client GraphQL simple

Nous avons des applications pour lesquelles la liaison avec l’api GraphQL du serveur se résume à deux trois requêtes GraphQL simples. Faut-il ajouter une librairie riche comme Apollo pour de simples appels ? Pas forcément, et d’ailleurs, pour l’un de nos projets en React, nous avons fait le choix de l’éviter. Nous sommes parti d’un simple “fetch”, y avons ajouté un header application/json, et voici nos requêtes : 

```javascript
  const getMediaById = `query getMediaById($mediaID: String!) {
    getMediaById(mediaID: $mediaID) ${mediaFragment}
  }`;

  …

  const body = {
     query: getMediaById,
     variables: { mediaID: fileSlug },
   };

   const data = await postGraphql(body);
```

Cette implémentation est un peu trop simpliste, nous perdons notamment le cache Apollo. Mais a-t-on nécessairement besoin de plus ? Bon, en réalité, on a fini par implémenter un mini cache en quelques dizaines de lignes pour éviter de faire deux appels identiques en un temps très court… :-) On notera l’utilisation des “Templates strings” qui simulent l’ajout de Fragments GraphQL. 

D’accord, mais comment fait-on, par exemple, pour uploader un fichier en GraphQL ? Avec Apollo, la chose était aisée. Or, en y regardant de plus près, après quelques recherches sur le net, il se trouve que c’est tout aussi facile d’implémenter l’upload GraphQL sans Apollo. Il faut juste connaître un peu la norme utilisée par Apollo et, en quelques lignes, nous pouvons ajouter l’Upload pour un fichier :

```javascript
const body = {
     query: upload,
     variables: {
       file: null,
       ...autres champs metiers,
     },
   };
   const formData = new FormData();
   formData.append("operations", JSON.stringify(body));
   formData.append("map", '{ "0": ["variables.file"] }');
   formData.append("0", uploadInfos.file);

   const response = await fetch(ConfigService.getGraphUrl(), {
     method: "POST",
     body: formData,
   });
   const data = await response.json();
```

Pour les curieux, la norme est spécifiée ici : [Spécifications](https://github.com/jaydenseric/graphql-multipart-request-spec).

Et pour l’upload GraphQL côté Backend ? Il se trouve que nous avons des spécialistes du Go qui l’ont implémenté avec brio comme expliqué ici: [Uploader des fichiers via un middleware GraphQL](https://vincent.composieux.fr/article/uploader-des-fichiers-via-un-middleware-graphql)  ;-)

## Conclusion

Le choix initial d’utiliser GraphQL se révèle aujourd’hui encore un choix judicieux tant il apporte de fluidité dans l’interaction entre les développeurs back et front. Au final, côté client, l’effort pour passer d’une expérience REST classique au langage Graphql, aura été vraiment minime notamment grâce à des librairies comme Apollo, compatibles pour nos projets Vue, React, en Javascript comme en Typescript. Dans certains cas assez simples, on peut tout aussi bien se passer de librairies complémentaires.

