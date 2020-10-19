---
title: La video
date: 2020-09-01
hero: /post/2020/architecture/video/images/hero.jpg
excerpt: Le fonctionnement de la plateforme vidéo de MYTF1
authors:
  - dlecorfec
---

# La vidéo
La vidéo est un domaine assez large, avec pas mal d'acronymes et de formats exotiques. Nous allons y aller progressivement 😉
## Plusieurs types de vidéos et modes de diffusion
Nous distinguons 2 types de vidéos:
- les flux live (chaines TF1, TMC, TFX, TF1 Séries Films, LCI et lives évenementiels) qui proviennent d'un _encodage_ en temps réel d'un flux vidéo "broadcast" vers un format de diffusion vidéo "informatique". Nous appelerons cette partie "live"
- les replays, extraits, spots publicitaires et bonus digitaux que nous regrouperons ici sous l'appelation "replay", et qui subissent des _transcodages_ vers différents formats pour les différents écrans de diffusion (dont les capacités varient)
MYTF1 diffuse de la vidéo de 2 manières différentes:
- en OTT (_over-the-top_, terme consacré pour la diffusion via Internet) via notre infrastructure ou des services tiers que nous payons (CDN - Content Delivery Networks). Ici notre enjeu est d'offrir la meilleure expérience au plus grand nombre, en terme de qualité visuelle, de latence et d'accessibilité, tout en minimisant nos coûts de diffusion, sans oublier la protection des contenus des ayants droit.
- via des partenaires qui assurent l'éventuel transcodage et la diffusion (IPTV - portails des box - et Salto).
Au niveau des formats de diffusion OTT, nous supportons les formats suivants:
- HLS ("HTTP Live Streaming", sur apps iOS et Safari Mobile)
- DASH ("Dynamic Adaptive Streaming over HTTP", sur le reste)
- MP4 (pour les courts spots de pub des replays)
HLS et DASH sont des formats de diffusion adaptés à la diffusion sur Internet: la vidéo est transcodée en différentes qualités et segmentée en bouts de quelques secondes, ce qui permet au player de s'adapter en cours de visionnage en téléchargeant la qualité la plus appropriée à sa capacité actuelle de téléchargement.
Pour la protection contre la copie, nous utilisons sur les replays en DASH les DRM Widevine (DRM Google: players sous Chrome, Firefox, Android ...) et Playready (DRM Microsoft, donc players sous Edge) et sur les replays en HLS la DRM Fairplay (Apple)
Les différents formats possibles pour une vidéo ne sont pas stockés de manière permanente, ils sont générés à la demande et mis en cache.
Au niveau des formats de compression OTT, nous utilisons le codec H.264 pour la vidéo, et le codec AAC pour l'audio.
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
## Architecture
(insérer schéma high-level ici)
## Technos utilisées dans la vidéo
Une grande partie de nos services est développée en interne grâce à des projets OpenSource, mais nous avons recours à des systèmes propriétaires pour certains aspects très techniques (encodage/transcodage, packaging et génération à la volée des différents formats)
### Dans la partie métadonnées
Le service MOVE ("Outil Vidéo Multi-Ecrans"), qui est notre backoffice de commande de replays, de découpe d'extraits et de livraison aux partenaires, est écrit en PHP/Symfony avec du MySQL derrière (oui, il vit depuis quelques années).
Le service de réferentiel vidéo, qui regroupe toutes les métadonnées des vidéos, a une API écrite en NodeJS et une autre en Go. Son stockage primaire est une base Postgresql (avec utilisation de champs JSON)
Le système de notifications de changement de métadonnées est architecturé autour de RabbitMQ.
Les services de mises à jour des métadonnées vidéo coté publicité sont écrits en Go.
Le service de métadonnées vidéo (mediainfo) appelé par les players est écrit en Go.
Au niveau DRM, nous avons le service Widevine et le service Fairplay qui sont écrits en Go, et le service Playready qui est écrit en C# (car SDK .NET)
### Dans la partie vidéo proprement dite
Le pilotage des transcodages est effectué par un outil (videoworkflow), écrit en Go et s'appuyant sur RabbitMQ.
Les transcodeurs sont des Elemental Server. Ce sont des serveurs propriétaires avec des GPU pour accélérer les traitements. Ils disposent d'un backoffice web et d'une API REST, par lesquels on peut créer des profils d'encodage et soumettre des jobs.
Le système de génération à la demande des différents formats vidéo, avec gestion des DRM et des sous-titres, est également propriétaire, de chez Unified Streaming.
Nos caches sont basés sur l'excellent serveur Web nginx, avec des serveurs physiques gavés de RAM et de disque.