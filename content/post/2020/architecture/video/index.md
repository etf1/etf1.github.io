---
title: La vidéo
date: 2020-10-22T16:00:00
hero: /post/2020/architecture/video/images/hero.jpg
excerpt: Le fonctionnement de la plateforme vidéo de MYTF1
authors:
  - dlecorfec
---

La vidéo est un domaine assez large, avec pas mal d'acronymes et de formats exotiques. Nous allons y aller progressivement 😉

## Types de vidéos

Nous distinguons 2 types de vidéos:

- les flux live, c'est à dire le direct des chaines TF1, TMC, TFX, TF1 Séries Films, LCI et lives évenementiels. Ils proviennent d'un _encodage_ en temps réel d'un flux vidéo "broadcast" (télévision) vers un format de diffusion vidéo "informatique". Nous appelerons cette partie "live". A noter que les mondes broadcast et informatique, longtemps séparés, tendent à se rapprocher ...
- les replays, programmes [AVOD](https://fr.wikipedia.org/wiki/Vid%C3%A9o_%C3%A0_la_demande) (MYTF1) ou SVOD (TFOU MAX), extraits, spots publicitaires et bonus digitaux que nous regrouperons ici sous l'appelation "replay", et qui subissent des _transcodages_ vers différents formats pour les différents écrans de diffusion (dont les capacités varient)

## Modes de diffusion

Les contenus MYTF1 sont mis à disposition des internautes de plusieurs manières différentes:

- en [OTT](https://fr.wikipedia.org/wiki/Service_par_contournement) (_over-the-top_) via notre infrastructure (origines et caches, liens réseau) et des services de cache tiers ([CDN](https://fr.wikipedia.org/wiki/R%C3%A9seau_de_diffusion_de_contenu) - _Content Delivery Networks_). Ici notre enjeu est d'offrir la meilleure expérience au plus grand nombre, en terme de qualité visuelle, de latence et d'accessibilité, tout en minimisant nos coûts de diffusion (ce qui est primordial pour une activité rémunérée principalement par la publicité), sans oublier la protection des contenus des ayants droit. Dans le cas classique de l'OTT, nous contrôlons
la partie serveur (origine vidéo) et la partie cliente (player). C'est le cas pour nos sites [MYTF1](https://www.tf1.fr/), [LCI](https://www.lci.fr/) et [TFOU MAX](https://www.tfoumax.fr/), ainsi que pour les apps iOS et Android.
- via des partenaires qui assurent la diffusion: normalement, nous ne contrôlons pas le serveur ni le player, nous nous contentons de livrer le média.

Parmi ces partenaires, on retrouve:

- les [FAI](https://fr.wikipedia.org/wiki/Fournisseur_d%27acc%C3%A8s_%C3%A0_Internet), avec le portail MYTF1 présent sur leurs [box](https://fr.wikipedia.org/wiki/Box_(Internet)).
- SALTO, pour les replays de nos différentes chaînes ainsi que TFOU MAX
- Amazon Prime Video, pour TFOU MAX

A noter que selon les box et produits des FAI, tout n'est pas si simple. Notamment avec les box Android, la diffusion revient vers un modèle OTT (nous contrôlons le serveur et le player et nous passons par notre infrastructure de diffusion, via Internet), parfois hybride: nous contrôlons
le serveur tandis que le FAI gère le player (ce qui est un peu risqué en terme d'assurance-qualité, il faut que les 2 côtés restent compatibles, donc nous préférons éviter ce genre de situation), ou bien le FAI met son CDN à notre disposition: notre player va chercher la vidéo sur le CDN du FAI, qui à son tour va chercher la vidéo (s'il ne l'a pas déjà) sur nos serveurs, via Internet.

## Description d'un fichier vidéo

Avant de parler de la manière dont nous diffusons sur le net, voyons quelques notions sur les fichiers vidéo et leurs formats.

### Propriétés d'une vidéo

Une vidéo a une dimension spatiale et une dimension temporelle.
Au niveau spatial, elle a une _résolution_ exprimée en pixels horizontaux et pixels verticaux (par exemple, 1280x720).
Au niveau temporel, elle a une durée et un nombre constant d'images par seconde (ips, ou en anglais, _frames per second_, fps), 25 dans notre cas. Au niveau audio, on parle d'échantillons par seconde, par exemple 48000, ou de fréquence d'échantillonage, soit ici 48 kHz.

Pour quantifier la capacité réseau nécessaire à la lecture d'une vidéo, le terme _bitrate_ est employé et désigne la quantité de données nécessaire pour diffuser 1 seconde de vidéo.
Ce terme est généralement en kilobits ou megabits par seconde (1000 kbps = 1 mbps, et 8 bits = 1 octet). Une vidéo a 500 kbps prendra 10 fois moins de débit qu'une vidéo à 5 mbps, mais aura une qualité et/ou une résolution inférieure.

Théoriquement, une image en 1280x720 pixels avec 3 octets par pixels prendrait 2.7 Mo. Ce qui donnerait une vidéo à 550 mbps (soit environ 70 Mo/s). De la compression est donc nécessaire.

### Compression

Les pistes audio et vidéo sont compressées avec perte, ce qui permet de réduire énormément leur taille, et de les diffuser sur des réseaux avec des capacités limitées.
Il y a donc un compromis à faire entre taille et qualité.

Pour rendre la perte la moins significative possible, les algorithmes de compression usuels se basent notamment sur des modèles psychoacoustiques (pour l'audio) et psychovisuels (pour la vidéo): certaines informations sont plus importantes que d'autres pour nos oreilles, nos yeux et notre cerveau, qui interprète ces données. A l'inverse, certaines informations sont peu utiles et peuvent être ignorées.

Par exemple, dans le domaine audio, les plus hautes fréquences sont inaudibles par l'oreille humaine,
tandis que dans le domaine visuel, l'oeil est plus réceptif aux changements de lumière qu'aux changements
de teinte, ou plus réceptif à l'aspect général d'une image, plutôt qu'à ses détails les plus fins.

Dans le cas de la vidéo, la compression interviendra au niveau d'une image, qui sera découpée en petits blocs, chaque bloc étant compressé individuellement. C'est pour ça qu'on peut parfois voir apparaître
des blocs dans une vidéo, si la compression est trop aggressive.

Souvent, il y a peu de différences entre 2 images successives. On va donc stocker une image et les différences avec les images suivantes plutôt qu'une série d'images. Mais on ne peut pas stocker des
différences indéfiniment, que ce soit lors d'un changement de plan ou pour sauter plus loin dans la vidéo.

On aura donc des groupes d'images, composés d'une image complète (image de référence, appelée _I-frame_, ou keyframe) et des différences (prédictives - _P-frame_ - ou bi-prédictives - _B-frame_) nécessaires pour
reconstruire les images suivantes.

![Dépendances des images P et B](images/ibp.svg#darkmode "Dépendances des images P et B")

Généralement ces groupes ne constituent pas plus de 2 secondes de vidéo.
Ces groupes sont appelés des _GOP_ (group of pictures).

![Un GOP de 15 images](images/gop15t.svg#darkmode "Un GOP de 15 images")

2 remarques:

- vu que les images B dépendent d'images qui les suivent, l'ordre de transmission des images n'est pas forcément l'ordre d'affichage des images.
- dans cet exemple de GOP de 15 images, la dernière est une B, qui dépend donc de l'image I qui commence le GOP suivant: on parle de GOP ouvert dans ce cas. On parle de GOP fermé lorsqu'il ne dépend pas des GOPs voisins (il ne commence ni se termine par une image B), ce qui facilite par exemple le _seeking_ (déplacement dans le temps), ou le passage d'un bitrate à un autre en cours de lecture.

La compression/décompression, aspect essentiel et très pointu de la diffusion vidéo, est implémentée par des _codecs_ (mot-valise anglais pour _coder-decoder_): il faut non seulement qu'ils soient connus par le système qui encode les vidéos, mais surtout par les sytèmes de ceux qui les lisent. Avec le développement de la mobilité, un codec n'est
utilisable en pratique que s'il dispose d'une version accélérée matériellement sur la plupart des smartphones, tablettes et ordinateurs, afin de garantir de bonnes performances et une consommation énergétique raisonnable. Mais ça explique pourquoi les codecs les plus répandus ont toujours au moins 10 ans d'existence, le temps que les implémentations matérielles se généralisent.

Parmi les codecs que nous utilisons actuellement, on peut citer [H.264](https://fr.wikipedia.org/wiki/H.264) pour la vidéo, et [AAC](https://fr.wikipedia.org/wiki/Advanced_Audio_Coding) pour l'audio.

### Formats de fichiers

Dans un fichier vidéo, on peut la plupart du temps distinguer le contenant du contenu: un fichier vidéo sera le plus souvent un conteneur organisant des données de types variés (notamment celles produites par les codecs audio/vidéo), et permettant de les retrouver facilement.
Ce conteneur aura le plus souvent au moins une piste vidéo et une piste audio, parfois plusieurs, et parfois d'autres types de pistes comme les sous-titres.

Par exemple, on peut avoir un fichier MP4 qui contient une piste vidéo provenant d'un codec H.264, une piste audio provenant d'un codec AAC et une piste de sous-titres au format [WebVTT](https://fr.wikipedia.org/wiki/WebVTT).

Le fait de contenir à la fois de la video et de l'audio est appelé multiplexage. Ce multiplexage peut être fait de différentes manières selon les fichiers. Par exemple, on peut avoir toutes les données vidéo, puis toutes les données audio. Ou à l'inverse, entremêler les différents types de données: un peu de vidéo suivi d'un peu d'audio, et ainsi de suite.

Les standards de formats de fichiers que nous gérons sont principalement issus de 2 organisations:

- [MPEG](https://fr.wikipedia.org/wiki/Moving_Picture_Experts_Group) (Moving Picture Experts Group)
- [SMPTE](https://fr.wikipedia.org/wiki/Society_of_Motion_Picture_and_Television_Engineers) (Society of Motion Picture and Television Engineers)

Parmi les formats de fichiers les plus courants, on a:

- [MPEG TS](https://fr.wikipedia.org/wiki/MPEG_Transport_Stream) (transport stream, 1994), avec des codes correcteurs d'erreurs, adapté pour la diffusion via un réseau
- [MPEG PS](https://fr.wikipedia.org/wiki/MPEG_program_stream) (program stream, 1994), sans codes correcteurs d'erreurs, pour les médias "fiables", utilisé sur les DVD par exemple
- [MP4](https://fr.wikipedia.org/wiki/MPEG-4_Part_14) (MPEG-4 Part 14, 2003), relativement répandu dans le monde informatique
- [MXF](https://fr.wikipedia.org/wiki/Material_Exchange_Format) (Material eXchange Format, issu de la SMPTE, 2004), relativement répandu dans le monde du broadcast

A noter que le format MPEG TS permet de simplement concaténer plusieurs fichiers pour obtenir un nouveau fichier MPEG TS valide!

## Formats de diffusion OTT

Au niveau des formats de diffusion OTT (c'est à dire, via Internet), nous supportons les 2 formats suivants:

- [HLS](https://fr.wikipedia.org/wiki/HTTP_Live_Streaming) ("HTTP Live Streaming", sur apps iOS et Safari Mobile), originaire d'Apple
- [DASH](https://fr.wikipedia.org/wiki/Dynamic_Adaptive_Streaming_over_HTTP) ("Dynamic Adaptive Streaming over HTTP", sur le reste), originaire de l'organisation MPEG

De plus, nous diffusons les spots de pubs via de simples fichiers MP4.

HLS et DASH sont des formats développés pour la diffusion sur Internet via HTTP: la vidéo est transcodée en différentes qualités et segmentée en petits fichiers vidéos indépendants (_chunks_) de quelques secondes, ce qui permet au player de s'adapter en cours de visionnage en téléchargeant la qualité la plus appropriée à sa capacité actuelle de téléchargement: on parle d'ABR, _Adaptive Bit Rate_.

De plus, la segmentation en petits fichiers permet de faciliter la mise en cache et la robustesse en cas d'erreur (il suffit de redemander le chunk posant problème).

Le player va d'abord récupérer un fichier contenant du texte:

- _main playlist_ (au format [M3U8](https://fr.wikipedia.org/wiki/M3U)) pour le HLS, contenant des métadonnées ainsi que des liens vers des _sous playlists_ M3U8 pour les différents bitrates,
- _manifest_ (au format [XML](https://fr.wikipedia.org/wiki/Extensible_Markup_Language)) pour le DASH, contenant des métadonnées et des liens vers les chunks des différentes pistes et différents bitates

![Exemple de segmentation HLS](images/hls.svg#darkmode "Exemple de segmentation HLS")

Pour la protection, nous avons mis en place de la [DRM](https://fr.wikipedia.org/wiki/Gestion_des_droits_num%C3%A9riques) sur certains contenus.

Nous utilisons sur les replays en DASH les DRM [Widevine](https://www.widevine.com/) (DRM Google: players sous Chrome, Firefox, Android ...) et [Playready](https://www.microsoft.com/playready/) (DRM Microsoft, donc players sous Edge) et sur les replays en HLS la DRM [Fairplay](https://developer.apple.com/streaming/fps/) (Apple)

Les manifests et chunks pour les différents formats (et chiffrements, pour la DRM) possibles pour une vidéo ne sont pas stockés de manière permanente, ils sont générés à la demande et mis en cache.

## Plusieurs sources pour les vidéos sur MYTF1

Les vidéos peuvent provenir de différentes sources:

- le MAM TF1 (media assets manager), notre bibliothèque de programmes prêts à être diffusés (séries, émissions enregistrées ...), avec éventuellement plusieurs pistes audio (VO et/ou audiodescription) et sous-titres (pour VOST et/ou pour sourds et malentendants). Les contenus sont réceptionnés quelques heures avant la diffusion, ce qui généralement nous permet une mise en ligne dès la fin de la diffusion antenne.
- le DVR (digital video recorder): nous enregistrons en HLS le direct des chaines et le gardons quelques jours. C'est à partir de là que nous livrons les replays des émissions en direct (JT notamment). Nous commandons une vidéo d'un intervalle de temps (à l'image près) d'une chaine. Ce système va identifier les chunks vidéos nécessaires, puis les recoller (facile avec les chunks en MPEG-TS) et rogner les bouts superflus afin de livrer un MP4 correspondant parfaitement à l'intervalle demandé.
- le FTP: on peut livrer une vidéo par simple transfert de fichier. C'est utilisé pour les contenus uniquement digitaux (AVOD, SVOD, extraits, pubs, ...)

## Plusieurs activités dans la gestion de la vidéo

La vidéo chez MYTF1 peut se décomposer en 2 grandes parties:

### Gestion des métadonnées live et replay (titre, résumé, dates de diffusion antenne et/ou de disponibilité sur MYTF1, ...) et des mises en ligne

- un backoffice éditorial de commande de "replay" (développé en interne)
- mise à disposition de ces informations aux autres services de MYTF1 et aux partenaires via différentes API et files de messages
- services pour les players (récupération des métadonnées et de l'URL de diffusion, protection de certains contenus via DRM - Digital Rights Management)

### Gestion des données vidéo live et replay

- ingestion (encodage/transcodage, gestion des sous-titres éventuels, packaging - préparation à la diffusion OTT)
- envoi aux partenaires, pour les vidéos (le live IPTV est géré par TF1)
- diffusion OTT (génération des formats HLS/DASH/MP4 éventuellement DRM-isés, caches, transit entre notre datacenter et les FAI, CDN)

La partie cache et transit est primordiale pour nos maîtrise des coûts de diffusion, afin d'utiliser le moins possible les services de CDN.
C'est pour cela qu'il doit être rapide de basculer la diffusion vidéo d'un point vers un autre, en fonction des besoins.

## Technos utilisées dans la vidéo

Une grande partie de nos services est développée en interne grâce à des [logiciels libres](https://fr.wikipedia.org/wiki/Logiciel_libre) ou des projets [Open source](https://fr.wikipedia.org/wiki/Open_source), mais nous avons recours à des systèmes propriétaires pour certains aspects très techniques (encodage/transcodage, packaging et génération à la volée des différents formats).

### Dans la partie métadonnées

Le service MOVE ("Outil Vidéo Multi-Ecrans"), qui est notre backoffice de commande de replays, de découpe d'extraits et de livraison aux partenaires, est écrit en [PHP](https://www.php.net/)/[Symfony](https://symfony.com/) avec du [MySQL](https://www.mysql.com/) derrière.

Le service de réferentiel vidéo (videoref), qui regroupe toutes les métadonnées des vidéos, a une API écrite en [NodeJS](https://nodejs.org/) et une autre en [Go](https://golang.org/). Son stockage primaire est une base [PostgreSQL](https://www.postgresql.org/) (avec utilisation de champs JSON).
Le système de notifications de changement de métadonnées est architecturé autour de [RabbitMQ](https://www.rabbitmq.com/).
Les services de mises à jour des métadonnées vidéo coté publicité sont écrits en Go.

Le service de métadonnées vidéo (mediainfo) appelé par les players est écrit en Go, ainsi que sa partie delivery, qui donne l'URL de la vidéo (en fonction du client et de la configuration, oriente vers le format et le CDN approprié, vérifie le géoblocage, etc ...)

Au niveau DRM, nous avons le service Widevine et le service Fairplay qui sont écrits en Go, et le service Playready qui est écrit en [C#](https://fr.wikipedia.org/wiki/C_sharp) (car SDK [.NET](https://dotnet.microsoft.com/))

### Dans la partie vidéo proprement dite

Le pilotage des transcodages est effectué par un outil (videoworkflow), écrit en Go et s'appuyant sur RabbitMQ pour la communication entre ses différentes étapes, et [ffmpeg](https://ffmpeg.org/) pour certaines opérations (récupération des métadonnées de la vidéo, génération de la petite vidéo muette de preview, conversion de MPEG TS en MPEG PS pour certains opérateurs).

Les transcodeurs sont des [Elemental Server](https://aws.amazon.com/elemental-server/). Ce sont des serveurs propriétaires avec des GPU pour accélérer les traitements. Ils disposent d'un backoffice web et d'une API REST, par lesquels on peut créer des profils d'encodage et soumettre des jobs.

Le système de génération à la demande des différents formats vidéo, avec gestion des DRM et des sous-titres, est également propriétaire, de chez [Unified Streaming](https://www.unified-streaming.com/). Initialement le produit s'appelait Unified Streaming Platform, nous continuons donc d'appeler ce système par l'acronyme "USP".

Nos caches sont basés sur l'excellent serveur Web [nginx](http://nginx.org/), avec des serveurs physiques gavés de RAM et de disques SSD.
Le sytème de commande DVR s'appuie sur ffmpeg et est écrit en Go.

![Architecture vidéo](images/archi-video.svg#darkmode "Architecture vidéo")