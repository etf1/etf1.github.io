---
title: "Coupe du Monde : dans les coulisses de MYTF1"
date: 2023-03-17T09:00:00
hero: https://photos.tf1.fr/1280/0/cover-hp-ott-135c2c-26558e-0@1x.png
excerpt: De l’architecture applicative aux CDN en passant par la sécurité, la Coupe du Monde a mis à l’épreuve tous les pans de l’infrastructure de MYTF1.
description: De l’architecture applicative aux CDN en passant par la sécurité, la Coupe du Monde a mis à l’épreuve tous les pans de l’infrastructure de MYTF1. Et a requis une mobilisation de tous, partenaires compris. Retour sur un événement exceptionnel.
---

De l’architecture applicative aux CDN en passant par la sécurité, la Coupe du Monde a mis à l’épreuve tous les pans de l’infrastructure de MYTF1. Et a requis une mobilisation de tous, partenaires compris. Retour sur un événement exceptionnel. 

Jusqu’à 29,4 millions de téléspectateurs durant la séquence de tirs au but de la finale France-Argentine. Un [record historique](https://groupe-tf1.fr/fr/communiques/excellentes-performances-pour-la-coupe-du-monde-de-la-fifa-2022tm-sur-tf1) pour un programme TV qui s’est également accompagné d’un autre record pour MYTF1 : 3 millions de visites cumulées durant la finale. Autant dire que pour relever le défi d’un tel événement e-TF1 n’avait pas d’autres choix que de se fixer un scénario très ambitieux. 
C’est donc dans cet esprit que les équipes ont travaillé pour assurer une diffusion en live et sans heurts des matchs de la Coupe du Monde via l’infrastructure de MYTF1. Un événement pour lequel TF1 doit tenir les engagements pris auprès de la FIFA, sous le regard de tous et durant lequel le moindre incident peut susciter un mécontentement à la hauteur des attentes. Plus encore pour les abonnés de l’offre payante [MYTF1 Max](https://www.tf1.fr/offre-max) lancée fin 2021.
À l’appui de l’expérience de la Coupe du Monde 2018, un objectif a été dimensionné : être en mesure de servir 1,5 million de personnes qui se connectent dans un intervalle de temps très réduit. Pour faire face à un tel tsunami, deux questions devaient notamment être tranchées :
* Faut-il opérer des changements majeurs sur l’infrastructure ou optimiser l’existant ?
* Est-il préférable de se reposer sur les mécanismes d’autoscaling ou de réserver les capacités requises en amont ?

### Pas de Big Bang, mais une préparation minutieuse
« Nous avons fait le choix d’une part de ne pas mener de changements majeurs sur l’infrastructure et de procéder plutôt à des optimisations ciblées, d’autre part de réserver les capacités requises par un tel événement, résume Ali Oubaziz, Head of Digital Infrastructure du groupe TF1. Dans ces contextes, avec de tels pics, les mécanismes d’autoscaling ne sont pas assez rapides et ne peuvent aider qu’à la marge ». Pas de Big Bang donc, mais une préparation minutieuse dont la clé de voûte est la mobilisation de toutes les parties prenantes, internes comme externes.
« Les équipes éditoriales nous ont aidés à évaluer les niveaux de criticité selon les matchs, détaille Djamel Arichi, Head of Service Management & Support. Et selon ce niveau de criticité, jusqu’à 20 personnes pouvaient être mobilisées le jour J ». Objectif, rassembler sur le plateau les représentants de chaque maillon : infrastructure, back end, player, CDN (Content Delivery Network), régie publicitaire, réseaux sociaux... Sans oublier les partenaires, tout particulièrement pour AWS (Amazon Web Service) sur lequel repose l’architecture applicative. Un customer success manager AWS était donc présent sur site, en relation avec les équipes Amazon.

### Des objectifs partagés avec l’ensemble des partenaires
La communication en amont avec les partenaires a été soignée : « nous leur avons partagé nos hypothèses d’audience pour débattre avec eux aussi bien des capacités à réserver que des optimisations à mener. Sur la base de ces discussions, les instances AWS ont été réservées, les CDN ont été dimensionnés en conséquence et les appels à des services clés ont été optimisés pour éviter de surcharger les API », résume Smaine Kahlouch, Team Leader SRE-DevOps-Cloud. « Cette planification en bonne intelligence avec l’ensemble de nos fournisseurs a représenté une bonne partie du travail de préparation et s’est avérée décisive pour le bon déroulé des opérations », souligne Ali Oubaziz.

### Une cartographie exhaustive de chaque maillon applicatif
Outre cette large communication, chaque pan de l’infrastructure a été passé au crible pour identifier les zones à risque. La partie applicative qui combine entre autres des containers orchestrés via [Kubernetes](https://tech.tf1.fr/post/2021/architecture/eks/), un back end en Go, [des API en GraphQL](https://tech.tf1.fr/post/2020/architecture/graphql-and-persisted-queries/), une architecture événementielle adossée à [Kafka](https://tech.tf1.fr/post/2021/architecture/migration-vers-kafka/), a été largement testée. « Nous avons veillé à ce que notre pré-prod soit dimensionnée de manière iso avec notre production. Objectif : coder des scénarios pour simuler la charge au regard de nos objectifs de nombre d’utilisateurs par seconde en obtenant des évaluations précises », précise Rémy Pinsonneau, architecte.

![Kafka transformer / projecteur](../../2021/architecture/migration-vers-kafka/images/archi-indexeur.svg#darkmode "Kafka transformer / projecteur")

Afin d’identifier toutes les applications externes impliquées dans la chaîne de performance, une cartographie a été dressée. « Pour la Home par exemple, nous avions listé de manière exhaustive les end points appelés afin de les mettre chacun à l’épreuve et d’analyser les effets en cascade en cas de défaillance du service. Ce travail a permis de définir le nombre de pods Kubernetes requis, mais aussi de procéder à des optimisations telles que la mise en place d’un cache in-memory ou encore la réduction des appels sur un service comme Gigya (fourni par SAP pour assurer l’authentification des utilisateurs) pour éviter de surcharger l’API ».

![Infrastructure des tests de performances](images/bench-k6.png#darkmode "Infrastructure des tests de performances")

### Des règles fines pour aiguiller sur le bon CDN
Côté streaming, ce sont 3 CDN qui œuvrent en coulisses, un CDN maison et deux CDN tiers. À cela s’ajoute le recours à une technologie capable d’exploiter les players des utilisateurs comme un réseau maillé pour orchestrer une diffusion peer-to-peer. « Cet ensemble de solutions a été pensé à la fois pour maîtriser les coûts et être résilient, analyse Simon Laroque, Team Leader Infrastructure Vidéo. On peut ainsi se permettre de perdre une source sans faire trembler l’édifice. »
Un travail très fin a également été mené pour faire face au pic de bande passante atteint durant la finale – 3,6 Terabits par seconde... 

![Traffic CDN](images/traffic-cdn.png#darkmode "Traffic CDN")

« Nous avons logé dans la partie applicative une brique qui aiguille sur le bon CDN selon de multiples paramètres et notamment le fournisseur d’accès Internet (FAI) de l’internaute. Selon les accords de peering des FAI, les capacités d’interconnexion avec nos CDN diffèrent, d’où l’intérêt d’aiguiller sur le CDN le mieux à même de servir l’internaute, explique Simon Laroque. Nous avons aussi ajouté des règles pour, en cas de saturation, basculer d’un CDN à l’autre. Des opérations automatiques, immédiates, quasiment sans latence ».

![Architecture diffusion vidéo](images/archi-cdn.png#darkmode "Architecture diffusion vidéo")

Outre ce travail d’optimisation des chemins de diffusion, une large concertation a été menée avec les partenaires. « N’oublions pas que nos lives sont aussi diffusés sur Molotov ou MyCanal, rappelle Simon Laroque. Une planification s’imposait avec nos fournisseurs pour éviter des embouteillages. Par exemple pour éviter que la mise à jour d’un jeu vidéo populaire ne soit lancée en pleine diffusion d’un match clé ».

### Un cockpit spécial « Coupe du Monde »
Pouvoir suivre en temps réel le comportement de l’infrastructure face à la charge est bien entendu un impératif. « Nous avions déjà pas mal de tableaux de bord. Nous en avons toutefois ajouté quelques-uns, précise Simon Laroque, notamment pour visualiser par FAI comment le trafic se ventile sur nos CDN. Surtout, nous avons regroupé ces tableaux de bord sous la forme d’un cockpit unique pour être en mesure de visualiser en un coup d’œil durant les matchs toutes les données clés. Celles en provenance des CDN comme celles produites par 3 solutions que nous utilisons pour mesurer la performance du point de vue de l’utilisateur final ».

![Tableau de bord “Coupe du Monde”](images/tdb.png#darkmode "Tableau de bord “Coupe du Monde”")

### Sécurité : pas d’outil magique
Sans surprise, un tel événement attire les hackers en quête d’exploits à forte visibilité. La sécurité a donc donné lieu, elle aussi, à un travail de préparation et à une surveillance minutieuse les jours de match. « Nous avons bien entendu revu en détail la sécurisation des frontaux, mais le vrai et gros sujet reste les attaques par déni de service (DOS, Denial of Service) », confirme Antoine Martin, responsable sécurité. Avec une difficulté spécifique...
« Pour un média comme TF1, d’un jour à l’autre les patterns de trafic ne sont pas les mêmes. Il est donc difficile de mettre en place des contre-mesures pérennes. Et soyons francs, face à une telle complexité, il n’existe pas d’outils magiques. La solution consiste d’une part à adapter en temps réel la sensibilité à des signaux faibles, d’autre part à analyser les logs à chaque match pour ajouter de nouveaux moyens de blocage. C’est ainsi, par itération, que l’on parvient à bloquer une bonne part du trafic frauduleux ».

![Exemple de pic de requêtes bloquées lors d’une attaque DDoS lors de la Coupe du Monde](images/ddos.png#darkmode "Exemple de pic de requêtes bloquées lors d’une attaque DDoS lors de la Coupe du Monde")

### Une démarche « d’hypercare »
Sécuriser un tel événement c’est aussi imaginer tous les scénarios. « Nous avons défini en amont tous les éléments pour faire face à une gestion de crise, souligne Djamel Arichi. Les canaux d’alertes ont été définis, les séquences de messages pour informer nos utilisateurs en cas de soucis ont été rédigés et validés pour l’ensemble des canaux – SMS , emails, réseaux sociaux... Plus globalement, nous nous sommes mis dans une démarche d’hypercare avec un support utilisateur renforcé durant les matchs ».

### Des enseignements pour l’avenir
Quels enseignements tirer de cet événement exceptionnel ? « Tout d’abord que la proximité avec nos partenaires clés, et en premier lieu AWS, a été très efficace, estime Smaine Kahlouch, Team Leader SRE-DevOps-Cloud. La présence de notre Customer Success Manager sur site a apporté une réactivité très précieuse. Pour la suite, l’idée n’est pas d’attendre les prochains grands événements pour rester à jour. Retour d’expérience de la Coupe du Monde à l’appui, nous allons régulariser les tests de charge pour nourrir un travail d’optimisation continu. » Objectif : tendre vers une infrastructure “futur proof”, donc toujours prête pour l’exceptionnel.
