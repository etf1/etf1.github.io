---
title: Workflow d'encodage et delivery vidéo avec Temporal
date: 2023-07-23T09:00:00
hero: /post/2023/architecture/workflow-video-avec-temporal/images/hero.jpg
excerpt: "Découvrez comment nous avons mis en place notre nouveau workflow d'encodage et de delivery vidéo avec Temporal."
authors:
  - vcomposieux
description: "Découvrez comment nous avons mis en place notre nouveau workflow d'encodage et de delivery vidéo avec Temporal."
---

## Contexte du nouveau workflow

Dans un objectif de faire évoluer et de rendre plus flexible notre workflow d'encodage et de mise à disposition de nos flux vidéo (ce que nous appelons le delivery, principalement aux formats Dash et HLS), nous avons souhaités effectuer une refonte applicative de cette partie de notre stack applicative.

Nous avons commencés par réfléchir à la façon dont nous pourrions découper les différentes tâches que nous effectuons pour l'encodage et la mise à disposition de nos vidéos.

Il était avant tout important de faire un état des lieux des tâches que nous effectuons. Elles sont les suivantes :

* Nous vérifions les fichiers fournis en entrée (fichier vidéo, fichier de sous-titres, ...),
* Nous déplaçons des fichiers sur des espaces de stockages,
* Nous générons des storyboards, une thumbnail et une petite preview de la vidéo,
* Nous effectuons des demandes d'encodage à des encodeurs, les interrogons pour obtenir le statut et attendons leur retour,
* Nous générons des clés de DRM auprès des différents fournisseurs (Fairplay, Playready, Widevine),
* Nous construisons un package (fichiers .ismv, .isma, .ism) afin qu'ils soient servis par USP Origin,
* Nous envoyons les fichiers produits sur un stockage AWS S3,
* Nous enrichissons une base de données de delivery avec les fichiers produits.

Toutes ces tâches ne sont pas systématiquement effectuées. Cela dépend en effet du type de vidéo que nous récupérons en entrée, il nous faut donc pouvoir gérer plusieurs workflows.

L'exemple le plus flagrant est celui des vidéos utilisées pour les publicités contre celles utilisées pour la VOD. En effet, les vidéos publicitaires n'ont pas de sous-titres, n'ont pas besoin de storyboards, thumbnail, preview et de DRM.

Nous avons donc choisis ce scope des vidéos publicitaires pour commencer notre travail sur ce nouveau workflow et pouvoir livrer rapidement un premier workflow en production.

Nous avons ensuite mis en place un feature toggle en production afin de servir une petite quantité de vidéo sur le nouveau workflow et monter progressivement en charge sur cette nouvelle version. Cela nous donne la possibilité de revenir rapidement sur l'ancien workflow si la moindre anomalie est détectée.

## Choix de l'outil : plusieurs solutions

Afin d'éviter de devoir développer nous-même un nième outil d'orchestration de workflow, nous souhaitions nous appuyer sur un outil fiable, éprouvé, open-source et disposant d'une communauté active.

Nous avons donc étudiés les outils suivants :

* [Conductor](https://conductor.netflix.com/) : Outil d'orchestration de workflow développé par Netflix basé sur de la définition,
* [Cadence](https://cadenceworkflow.io/) : Outil d'orchestration de workflow développé par Uber pour des systèmes distribués,
* [Temporal](https://temporal.io/) : Outil d'orchestration de workflow.

Nous avons assez rapidement écartés Conductor car cet outil se base sur une définition en JSON des tâches à éxécuter dans les workflows. De plus, la communauté maintenant le SDK Go est assez restreinte.

Cadence et Temporal sont deux outils assez similaires en terme d'utilisation.

Temporal nous semblait cependant beaucoup plus évolué au niveau de l'outillage (CLI, SDK, interface web) et il s'agit surtout d'un outil avec une société dédiée et une grosse communauté qui nous permet d'avoir du support si nécessaire.

## Architecture du cluster Temporal

Temporal apporte une architecture très intéressante et des briques indépendantes qui vous apporteront une résilience et vous donneront la possibilité d'adapter les ressources dont vous aurez besoin.

En effet, une fois le [cluster Temporal déployé](https://docs.temporal.io/cluster-deployment-guide) avec l'aide de notre équipe IOPS, celui-ci s'occupe de la gestion de nos workflows, sans avoir à avoir connaissance de nos applications (via des workers), qui sont déployées librement et de façon indépendantes.

![Architecture](images/architecture.svg#darkmode "Architecture")

Le cluster Temporal se compose des services suivants :
* Frontend : il s'agit du serveur gRPC auquel se connectent les clients (workers, CLI, interface web), s'occupe également du rate limiting du routing et de la gestion d'autorisations,
* History : il maintient les données des workflows et activités (entrées/sorties et états),
* Matching : il s'occupe du dispatch des activités et workflows via la notion de Task Queues,
* Worker : il est utilisé pour la gestion interne des exécutions de workflows,
* Une base de données : il est possible d'utiliser MySQL, PostgreSQL ou Cassandra. Vous pourrez aussi utiliser Elasticsearch comme stockage secondaire pour de la recherche.

## Architecture applicative mise en place

Nous avons fait le choix d'ajouter une API métier devant notre cluster Temporal. La responsabilité de cette API est de piloter l'exécution des workflows dans Temporal et également de maintenir un état condensé de l'exécution de nos workflows.

Ainsi, nos clients ne contactent jamais le cluster Temporal en direct mais passent toujours via notre API métier. Ceci nous permet d'avoir une zone de sécurité si nous devions effectuer des opérations ou changer d'outil par la suite.

Une fois l'API de pilotage et le cluster en place, il ne nous reste plus qu'à commencer à déployer nos workers. Un worker est une application qui permet d'exécuter une ou plusieurs activités ou un ou plusieurs workflows.

## Mise en place des workers

Cette philosophie de workers permet de pouvoir mettre à l'échelle efficacement nos applications, en fonction des besoins en ressource ou durée d'exécution des tâches qui sont exécutées dans ceux-ci.

![Workers](images/workers.svg#darkmode "Workers")

Dans cet exemple, nous avons quatre workers :
* Un worker "Check" : Ce worker permet l'exécution de deux activités qui nous permettent de vérifier le fichiers vidéo (pistes vidéos / audios, nombre de frames, ...) ainsi que les sous-titres associés (durée cohérente, langue utilisée, ...),
* Un worker "Video" : Ce worker a plus de besoins en ressources car il exécute des opérations gourmandes afin de générer les storyboards, thumbnails et preview de la vidéo,
* Un worker "DRM" : Ce worker génère les clés de DRM auprès des différents servides, il est très peu coûteux en ressources,
* Un worker "Workflows" : Ce worker permet de piloter l'exécution des workflows (l'ensemble des activités qui doivent être exécutées).

Bien évidemment, il est possible d'organiser les workers comme vous le souhaitez. De notre côté, nous avons fait le choix de les regrouper pour le moment par thématique mais nous pourrons revoir ce point plus tard si le besoin s'en fait sentir. C'est un point de souplesse important apporté par Temporal.

## Implémentation et fonctionnalités de Temporal

Temporal vient avec des SDKs bien fournis pour la plupart des langages. Dans l'équipe vidéo, nous utilisons Go et utilisons donc le [SDK Go](https://github.com/temporalio/sdk-go) associé.

Le SDK permet d'instancier un worker qui communique via [gRPC](https://grpc.io/) avec le frontend du cluster Temporal puis d'avoir tous les outils nécessaires à la mise en place des activités et workflows.

### Création d'une activité et d'un workflow

Un atout majeur de Temporal vient du fait que le code applicatif est presque complètement découplé du SDK Temporal.

En effet, nous pouvons écrire simplement l'activité suivante :

```go
func MyActivity(
  ctx context.Context,
  params *proto.MyActivityParams,
) (*proto.MyActivityResult, error) {
  // ...
  return nil
}
```

Comme on le voit dans cet exemple, l'activité est simplement une fonction qui sera exécutée par Temporal lorsqu'elle sera enregistrée au worker. Aucun lien avec le SDK Temporal, il serait donc facile de re-utiliser cette fonction ailleurs, comme par exemple dans un autre moteur d'orchestration de workflow.

Nous avons fait le choix d'utiliser [Protocol Buffers](https://protobuf.dev/) pour la gestion de nos paramètres d'entrée et sortie des activités et workflow, cela est bien géré par Temporal et nous permet d'avoir de réels contrats d'interface pour chaque activité et workflow.

C'est presque pareil pour le code d'un workflow, à la seule exception que l'argument `ctx` provient du package workflow du SDK (`workflow.Context`).

```go
func MyWorkflow(ctx workflow.Context) error {
  opts := workflow.ActivityOptions{
    // ...
    TaskQueue:              "my-task-queue",
    ScheduleToStartTimeout: 5 * time.Minute,
    ScheduleToCloseTimeout: 60 * time.Second,
    StartToCloseTimeout:    30 * time.Second,
    RetryPolicy: &temporal.RetryPolicy{
      InitialInterval:        10 * time.Millisecond,
      BackoffCoefficient:     1.2,
      MaximumInterval:        3 * time.Second,
      MaximumAttempts:        5,
      NonRetryableErrorTypes: nil,
    },
  }

  var result *proto.MyActivityResult

  activityCtx := workflow.WithActivityOptions(ctx, opts)
  err := workflow.
    ExecuteActivity(activityCtx, MyActivity, &proto.MyActivityParams{}).
    Get(ctx, &result)

  return err
}
```

Dans cet exemple, notre workflow exécute une seule activité : `MyActivity`.

Point important à souligner, dans notre implémentation, nous avons souhaités :

* Avoir peu de logique dans nos activités : une activité doit s'occuper de réaliser une seule et même action,
* Rendre le plus générique possible les activités afin de pouvoir les re-utiliser dans les différents workflows,
* Certaines activités peuvent être optionnelles dans notre workflow, dans ce cas, cela doit être spécifié par un paramètre en entrée afin de garantir le même comportement lors d'une re-exécution du workflow.

Dans le code du workflow ci-dessus, nous avons définis des timeouts pour l'exécution de notre activité. Ceux-ci peuvent être définis pour chaque activité, ce qui permet d'avoir un contrôle complet sur l'exécution. Il est également possible de définir une logique de retry personnalisée pour chaque activité.

### Association à un worker

Une fois l'activité et le workflow créé, il n'y a plus qu'à écrire le worker qui exécutera ces fonctions. Rien de plus simple :

```go
func main() {
  client, err := client.NewClient(client.Options{})
  if err != nil {
    log.Fatal("Failed to create Temporal client:", err)
  }
  defer client.Close()

  workerOptions := worker.Options{
    TaskQueue: "my-task-queue",
  }
  worker := worker.New(c, "my-namespace", workerOptions)
  defer worker.Stop()

  worker.RegisterActivity(MyActivity)
  worker.RegisterWorkflow(MyWorkflow)

  err = worker.Run(workerOptions)
  if err != nil {
    log.Fatal("Failed to start worker:", err)
  }
}
```

C'est le seul code dont votre worker a besoin pour exécuter cette activité dans ce workflow.

Comme nous l'avons vu précédemment, les workflows et activités à exécuter sont envoyés aux workers en fonction des Task Queues. 

### Déterminisme

Pour le bon déroulé des workflows, y compris lors de la re-exécution d'un workflow, et ainsi éviter des erreurs dûes à des logiques aléatoires et/ou datées dans le temps, Temporal vient avec un aspect [déterministe](https:/docs.temporal.io/workflows#deterministic-constraints).

Cela signifie que même si l'exécution du workflow échoue ou redémarre, le résultat doit rester le même.

Pour garantir cela, le SDK vient avec un ensemble de méthodes, comme par exemple ici, `workflow.Now` pour obtenir l'heure courante et `workflow.GetRandomValue` pour générer une valeur aléatoire :

```go
func MyWorkflow(ctx workflow.Context) error {
  // ...
  currentTime := workflow.Now(ctx)
  randomNumber := workflow.GetRandomValue(ctx)
  // ...

  return nil
}
```

## Notre workflow VOD

Notre workflow VOD, le plus complet à ce jour, exécute l'ensemble d'activités suivantes :

![Workflow VOD](images/workflow_vod.svg#darkmode "Workflow VOD")

Comme vous pouvez le voir, il est également possible de paralléliser l'exécution de certaines activités, ce qui permet de rendre l'exécution du workflow encore plus rapide. C'est le cas de nos activités de génération de storyoards et thumbnail.

Les paramètres d'entrée et sortie de notre workflow sont définis via Protobuf comme suit :

```go
message VodOttParams {
  string id = 1 [(validate.rules).string.uuid = true];
  source.InputContent source = 2 [(validate.rules).message.required = true];
  repeated AudioTrack audio_tracks = 3;
  repeated subtitle.InputFile subtitles = 4;
  bool is_drm_enabled = 5;
  caller.CallParams callback = 6;
}

message VodOttResult {
  repeated delivery.File delivery_files = 1;
  source.OutputContent thumbnail = 2;
  source.OutputContent preview = 3;
}
```

Lors de la demande d'exécution d'un workflow, nous avons besoin en entrée de l'API des informations suivantes :
* Identifiant de la vidéo
* Emplacement du fichier source (peut-être un fichier sur un système de fichier, HTTP ou encore sur AWS S3),
* Informations sur les pistes audios,
* Informations sur les fichiers de sous-titres associés,
* Savoir si les DRM doivent être activés ou non (cela conditionnera l'exécution de l'activité de DRM),
* Informations sur une éventuelle callback à effectuer en fin de workflow (cela conditionnera l'exécution de l'activité de callback).

En sortie du workflow, nous avons choisis de renvoyer les éléments suivants :
* Informations de fichiers de delivery qui ont été produits par le workflow,
* Emplacement du fichier de thumbnail JPG généré,
* Emplacement du fichier de preview vidéo généré.

En complément, Temporal permet également via des APIs de récupérer l'ensemble des paramètres d'entrée et sortie des activités du workflow qui ont été exécutées dans le workflow.

Nous stockons également ces informations dans notre API de workflow qui nous permet de restituer toutes ces informations au client. Cela permet d'apporter des informations supplémentaires lorsque l'on souhaite avoir un aperçu complet de ce qui a été exécuté dans notre workflow.

## Versioning et rétro-compatibilité

Lorsque nous faisons évoluer notre workflow ou nos activités, la question de rétro-compatibilité se pose, d'autant plus lorsque nous utilisons Protocol Buffers.

Pour ce qui est des workflow, Temporal vient avec un outillage permettant de conditionner l'exécution à l'intérieur du workflow en fonction de la version qui est en cours d'exécution.

Ainsi, il est possible d'ajouter un (ou plusieurs) tag de version dans le code du workflow via la ligne suivante :

```go
version := workflow.GetVersion(ctx, "Version-20230630-1", 1, 2)
```

Nous pouvons ensuite conditionner l'exécution d'une activité comme suit :

```go
	if version == 1 {
		mediaCheckResult, err := executeMediaCheck(ctx)
		if err != nil {
			return workflowResult, fmt.Errorf(
				"when running mcheck activity: %w", err,
			)
		}
	} else {
		// Nous ne voulons plus exécuter cette activité en version 2.
	}
```

Cette solution pouvant devenir rapidement compliquer à suivre, il peut aussi nous arriver de dupliquer le workflow en le suffixant `V2`. Ainsi, lorsque tous les workflow en version 1 seront terminés, nous pourrons simplement supprimer le code applicatif du workflow version 1.

Nous utilisons la même logique de duplication pour les changements qui ne seraient pas rétro-compatibles sur les activités.

## Métriques et observabilité

Le cluster Temporal et son SDK viennent également avec leur lot de métriques et d'outillage [Open-Telemetry](https://opentelemetry.io/).

Côté métriques, simplement en implémentant le SDK, vous pourrez alors ainsi récupérer beaucoup d'informations. Nous conseillons de suivre principalement les métriques suivantes :

* Activité : Tracking du temps d'attente du statut "schedule" à "start" (`activity_schedule_to_start_latency`),
* Activité : Temps d'exécution des activités : celle-ci est importante pour donner une idée du temps d'exécution et ainsi adapter les timeouts (`activity_execution_latency`),
* Activité : Exécutions en échec (`activity_execution_failed`),
* Workflow : Temps d'exécution total du workflow (`workflow_endtoend_latency`),
* Workflow : Exécutions en échec ou complétés avec succès (`workflow_completed` et `workflow_failed`),
* Worker : Slots disponibles pour l'exécution de tâches (`worker_task_slots_available`).

Nous utilisons Datadog comme plateforme d'observabilité pour notre infrastructure, nos logs et aussi nos traces applicatives.

Le SDK de Temporal permet également d'envoyer des traces pour chaque activité de notre workflow :

![Open-Telemetry Tracing](images/tracing.jpg#darkmode "Open-Telemetry Tracing")

## Conclusion

La mise en place de Temporal dans notre architecture applicative était un réel succès et nous a apporté un meilleur contrôle sur l'ensemble de nos tâches ainsi qu'une meilleure visibilité sur leur exécution.

Nous avons pu commencer notre étude en installant simplement le binaire [temporalite](https://github.com/temporalio/temporalite) qui permet de faire tourner un cluster allégé, ce qui est super pour commencer à tester l'outil.

La réflexion effectuée en amont et le fait d'avoir rendu les activités le plus générique possible nous a permis une très grande ré-utilisation sur l'ensemble de nos workflows et ainsi gagner en temps de développement sur la mise en place des workflows suivants.

## Crédits

L'image utilisée pour illustrer cet article est fournie par [Jamie Street](https://unsplash.com/fr/photos/XhzGiOXw8yE).