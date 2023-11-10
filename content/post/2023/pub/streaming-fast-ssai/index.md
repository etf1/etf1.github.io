---
title: Gestion du Server-Side Ad Insertion (SSAI) sur nos chaînes FAST
date: 2023-11-06T09:00:00
hero: /post/2023/pub/streaming-fast-ssai/images/hero.jpg
excerpt: "Découvrez comment nous avons mis en place nos chaînes FAST et notre solution SSAI pour y insérer la pub."
authors:
  - rpinsonneau
description: "Découvrez comment nous avons mis en place nos chaînes FAST et notre solution SSAI pour y insérer la pub."
---

## Chaînes FAST

FAST est l'acronyme pour `Free Ad-supported Streaming Television`. Il s'agit de chaînes en streaming gratuites avec de la publicité. Le principe est de reprendre des contenus du catalogue TF1, typiquement en AVOD (Advertising Video on Demand) et de l'assembler pour constituer une grille de programmation qui alimentera une chaîne live. C'est le principe de Stream sur [MYTF1](https://www.tf1.fr/tf1/direct). On parle alors de re-linéarisation des contenus.

### Mise en oeuvre des flux origin

eTF1 assure le delivery des contenus VOD en [HLS](https://developer.apple.com/streaming/) (HTTP Live Streaming) et [DASH](https://dashif.org/) (Dynamic Adaptive Streaming over HTTP). Si vous ne l'avez pas déjà lu, nous avons écrit [un article sur notre architecture vidéo](https://tech.tf1.fr/post/2020/architecture/video/).

Afin de déployer nos chaînes FAST nous avons développé une brique maison "vod2live" qui permet de générer des flux live à partir de contenus VOD.

DASH et HLS sont deux formats de streaming vidéo qui permettent à un player de connaître les segments vidéo/audio (chunks) à télécharger pour lire un contenu. Le découpage en segment permet la lecture en streaming, le fait qu'il existe différents variants (bitrate / résolution) permet au player de faire de l'adaptation de la qualité à la bande passante réseau disponible (adaptative bitrate).

Dans un contexte VOD, l'ensemble des segments est connu dès l'initialisation du player. Dans un contexte live, le player recharge régulièrement la playlist HLS ou le manifest DASH pour avoir connaissance des segments à venir : il s'agit d'une fenêtre glissante, les segments trop anciens disparaissent et les nouveaux segments sont ajoutés. 

La fenêtre DVR (Digital Video Recorder) est l'intervalle de temps dans lequel le player pourra positionner sa tête de lecture pour jouer le live, soit au plus proche du direct, soit avec un décalage (en timeshifting). Un délais est toujours présent pour garantir un buffer minimal.

![DVR](images/dvr.drawio.svg#darkmode "time window")

Le rôle de la brique vod2live est donc à partir d'une grille de programme de déterminer en fonction de l'heure courante la liste des segments à présenter au player. 

Une grille de programme peut être générée de plusieurs façons :
* manuellement : chaque programme est positionné sur une grille
* automatiquement : une liste de programme est déterminée et boucle à l'infini à partir du démarrage de la chaîne

Cette seconde option est la plus simple à gérer car elle demande moins d'effort sur la programmation des chaînes. À partir d'une liste de vidéos ordonnées, de leur durée et de la date de démarrage de la boucle on peut déterminer à chaque instant (à l'aide de modulo) :
* l'index de la boucle courante
* l'index du programme courant
* l'index du segment correspondant

![vod2live DVR](images/vod2live-dvr.drawio.svg#darkmode "vod2live loops")

Dans l'exemple ci-dessus, on est sur la première boucle de la chaîne. Le premier segment de la fenêtre tombe sur le deuxième segment de la première vidéo et la fin de la fenêtre tombe sur le premier segment de la deuxième vidéo.

### Architecture de la brique VOD2LIVE

![architecture](images/archi-vod2live.drawio.svg#darkmode "schema")

Afin de générer les flux DASH / HLS de nos chaînes FAST, vod2live est interfacé à plusieurs systèmes :
* l'`enhancer` est une brique interne qui permet de construire une vue consolidée des chaînes, sa date de démarrage, la liste ordonnée des vidéos qui la compose et leurs durées, ce qui permet de calculer la grille à tout instant
* l'`[USP](https://www.unified-streaming.com/products/unified-packager)`` est la solution de packaging vidéo que nous utilisons et permet de générer à la volée les manifests DASH et playlist HLS des VOD sources
* les briques `delivery` et `mediainfo` permettent au player de récupérer une URL sécurisée de chaque flux

![sequence diagram](images/vod2live-seq-diag.drawio.svg#darkmode "schema")

Pour générer un manifest DASH ou une playlist HLS, la brique vod2live effectue les tâches suivantes :

1. récupération de la définition de la chaîne et calcul de la grille
2. calcul du ou des programmes inclus dans la fenêtre DVR actuelle
3. chargement des manifests DASH / playlist HLS des vidéos concernées
4. calcul des segments à inclure dans la fenêtre DVR et génération d'un manifest DASH ou playlist HLS

### Manipulation des manifests DASH

La génération d'un flux live DASH à partir de manifest provenant de VOD est relativement facile si l'on s'appuie sur le concept de [periods](https://dashif-documents.azurewebsites.net/DASH-IF-IOP/master/DASH-IF-IOP.html#periods).

En effet, un manifest DASH ou fichier MPD (Media Presentation Description) est constitué d'une ou plusieurs périodes.

![PeriodsMakeTheMpd](images/PeriodsMakeTheMpd.png#darkmode "MPD periods")

Dans un MPD, chaque période de la [timeline](https://dashif-documents.azurewebsites.net/Guidelines-TimingModel/master/Guidelines-TimingModel.html#mpd-general-timeline) correspond à une source (encodage) et contient elle même les éléments qui encapsulent les segments vidéo et audio pour les différents variants. On peut donc lire séquentiellement plusieurs VOD dans un manifest.

![BasicMpdElements](images/BasicMpdElements.png#darkmode "MPD elements")

Ci-dessous, un exemple très allégé de MPD live constitué de deux périodes :

```xml
<MPD xmlns="urn:mpeg:dash:schema:mpd:2011" type="dynamic"
  availabilityStartTime="2017-12-02T09:35:00Z" timeShiftBufferDepth="PT400S">
  <Period>
    ...
  </Period>
  <Period start="PT300S" duration="PT300S">
    ...
  </Period>
</MPD>
```

Le flux live commence à la date indiquée par l'attribut "availabilityStartTime". Les différentes périodes qui le composent sont positionnées de façon relative par rapport à cette date absolue grâce à leur attribut "start". La première période commence donc à cette date (start="PT0S" par défaut), la deuxième période commence 300s après la première période, qui à donc une durée de 300s. La deuxième période a également une durée de 300s.

La valeur de l'attribut "timeShiftBufferDepth" indique une fenêtre de DVR de 400s.
Si la date courante est "2017-12-02T09:45:00Z" soit 600s (la durée des deux périodes) de plus que "availabilityStartTime" alors notre fenêtre DVR commence à 200s de la première période.

Vous l'avez compris, pour générer le flux live, il faut recopier dans un nouveau MPD les périodes des VOD correspondantes à la DVR courante en adaptant la timeline DASH.

Il est également nécessaire d'adapter les SegmentTimeline dans le cas d'[adressage explicite](https://dashif-documents.azurewebsites.net/Guidelines-TimingModel/master/Guidelines-TimingModel.html#addressing-explicit).

```xml
<MPD xmlns="urn:mpeg:dash:schema:mpd:2011">
  <Period duration="PT900S">
    <AdaptationSet>
      <Representation>
        <SegmentTemplate timescale="1000" presentationTimeOffset="902"
            media="video/$Time$.m4s" initialization="video/init.mp4">
          <SegmentTimeline>
            <S t="900" d="4001" r="224" />
            <S t="900225" d="1775" />
          </SegmentTimeline>
        </SegmentTemplate>
      </Representation>
    </AdaptationSet>
  </Period>
</MPD>
```

* timescale > nombre d'unité de temps pour une seconde
* `S.t` : offset du segment exprimé en unité de timescale (soit 0,9s)
* `S.d` : durée du segment exprimé en unité de timescale
* `S.r` : nombre de segment similaire consécutifs

L'url des segments est déduite par le player à partir du template "video/$Time$.m4s", ici "$Time$" est remplacé par l'offset "S.t" du segment correspondant.

Afin d'inclure les bon segments dans notre fenêtre DVR, il faut adapter les "S.t", "S.r" du "SegmentTimeLine".


### Manipulation des playlist HLS

La génération d'un flux HLS est beaucoup plus simple qu'en DASH. En HLS, il existe une playlist pour chaque variant audio/vidéo. Une playlist est un fichier [m3u8](https://fr.wikipedia.org/wiki/M3U) qui contient la liste des segments de la DVR.

```shell
#EXTM3U
#EXT-X-VERSION:6
#EXT-X-MEDIA-SEQUENCE:206756
#EXT-X-TARGETDURATION:8
#EXT-X-DISCONTINUITY-SEQUENCE:1737
#EXTINF:8.000, no desc
../videoA-126.ts
#EXTINF:8.000, no desc
../videoA-127.ts
#EXT-X-DISCONTINUITY
#EXTINF:8.000, no desc
../videoB-1.ts
#EXTINF:8.000, no desc
../videoB-2.ts
```

L'attribut "EXTINF" indique la durée des segments (ici 8s). Pour enchaîner deux VOD distinctes, l'attribut "EXT-X-DISCONTINUITY" permet d'indiquer au player une discontinuité dans l'encodage de deux segments successifs.

Il suffit alors de lister explicitement les segments des VOD en plaçant les tags #EXT-X-DISCONTINUITY entre les VOD.

Il est également nécessaire de tenir à jour les valeurs des attributs #EXT-X-MEDIA-SEQUENCE et #EXT-X-DISCONTINUITY-SEQUENCE.
* `#EXT-X-MEDIA-SEQUENCE` : doit être incrémenté à chaque fois qu'un segment disparait de la playlist
* `#EXT-X-DISCONTINUITY-SEQUENCE` : doit être incrémenté à chaque fois qu'un segment en discontinuité disparait de la playlist

Il y a cependant un prérequis fort sur HLS, les contenus doivent tous avoir exactement le même nombre de variants (une seule master playlist pour le flux). Sans cela, la jonction entre les VOD n'est pas possible si un variant n'existe plus sur le contenu suivant, le player plante et s'arrête.

## Insertion des publicités

Pour l'insertion des publicités dans le flux vidéo des chaînes FAST il est nécessaire d'ajouter des emplacements prévus à cet effet. La publicité n'étant pas la même pour chaque utilisateur (ciblage, état de l'inventaire disponible sur l'ad server), la durée des tunnels de publicité (ad break) peut être très variable. Cela n'est pas gênant en VOD, mais sur une chaîne live, cela peut induire un décalage trop important par rapport à la grille de programmation. Afin d'éviter ce décalage, des emplacements sont ajoutés dans la grille de programme. Ces emplacements correspondent à des vidéos (ad slate) qui seront remplacées par la publicité.

![adslates](images/adslates.drawio.svg#darkmode "ad slates")

Dans l'exemple ci-dessus, le premier ad slate est remplacé par trois spots publicitaire, la durée totale des spots est égale à celui de l'ad slate.

Le deuxième ad slate lui n'est remplacé que par un seul spot (manque d'inventaire sur l'ad server). Dans ce cas, pour éviter de créer un décalage dans le flux, une partie de l'ad slate vient compléter le spot pour que la durée totale soit équivalente à celle de l'ad slate initial.

Dans le flux vidéo d'origine, des marqueurs [SCTE35](https://www.scte.org/standards/library/catalog/scte-35-digital-program-insertion-cueing-message/) sont ajoutés pour indiquer les emplacements où des opportunités publicitaire sont possibles ainsi que leur durée.

À noter également, pour insérer de la publicité au milieu d'un contenu, il est nécessaire de le couper en deux parties. Cela est possible en DASH ou HLS à condition que le point de jonction corresponde au début d'un segment. Afin que le packager puisse créer un segment à une frame précise, il est nécessaire de forcer un GOP (Group Of Picture) au niveau de l'encoder afin de tomber sur une frame I. En général, ce comportement est assuré par la présence de marqueur SCTE35 dans la source VOD. À default d'avoir correctement préparer les contenus, la publicité ne pourra être insérée qu'à la jonction des deux segments les plus proches de la frame désirée.

### Architecture de la brique adback

Pour insérer les publicités dans nos flux FAST, nous avons développé une solution SSAI (Server Side Ad Insertion) maison, nommée "adback".

![architecture](images/archi-adback.drawio.svg#darkmode "schema")

Adback est interfacé à plusieurs systèmes :
* `vod2live` : qui permet de récupérer le flux préparé avec des ad slates et marqueur SCTE35
* `wizads` : une brique interne qui proxifie les appels avec notre ad server (Freewheel)
* `C3PO` : un référentiel de publicités connues de notre delivery vidéo
* `video wf` : notre workflow d'encodage & packaging, il est sollicité par C3PO pour mettre à disposition les spots publicitaires
* l'`[USP](https://www.unified-streaming.com/products/unified-packager)` : la solution de packaging vidéo que nous utilisons et permet de générer à la volée les manifests DASH et playlist HLS des spots publicitaires (c'est la même brique qui est utilisée par vod2live)

![sequence diagram](images/adback-seq-diag.drawio.svg#darkmode "schema")

Pour insérer les spots publicitaire, la brique adback effectue les tâches suivantes :
1. `Adback` : récupération du manifest DASH ou playlist HLS du contenu, préparé avec les ad slates & marqueur SCTE35 par vod2live
2. `Wizads` : sur présence de marqueur SCTE35, appel à l'ad server Freewheel pour récupérer la liste de publicités du tunnel (format VAST, cf. ci-dessous), la durée du tunnel est extraite du marqueur SCTE35 et indiqué à l'ad server
3. `C3PO` : les publicités qui ne sont pas déjà connues font l'objet d'une demande d'encodage asynchrone à notre workflow vidéo, les publicités déjà encodées sont conservées dans le VAST retourné par Wizads
4. `Adback` : les manifests DASH ou playlist HLS du contenu et des pubs permettent après re-manipulation de générer le flux live, le tunnel est persisté en session pour chaque utilisateur

### Template VAST

Ci-dessous, un exemple de template VAST avec un spot publicitaire :

[VAST](https://www.iab.com/guidelines/vast/) est le format utilisé pour récupérer de l'ad server la liste des spots publicitaires pour un tunnel.

```xml
<VAST version="4.2" xmlns:xs="http://www.w3.org/2001/XMLSchema" xmlns="http://www.iab.com/VAST">
  <Ad id="20001" >
    <InLine>
      <AdSystem version="1">iabtechlab</AdSystem>
      <Error><![CDATA[https://example.com/error]]></Error>
      <Impression id="Impression-ID"><![CDATA[https://example.com/track/impression]]></Impression>
      <AdServingId>a532d16d-4d7f-4440-bd29-2ec05553fc80</AdServingId>
      <AdTitle>Inline Simple Ad</AdTitle>
      <AdVerifications></AdVerifications>
      <Advertiser>IAB Sample Company</Advertiser>
      <Category authority="https://www.iabtechlab.com/categoryauthority">AD CONTENT description category</Category>
      <Creatives>
        <Creative id="5480" sequence="1" adId="2447226">
          <Linear>
            <TrackingEvents>
              <Tracking event="firstQuartile"><![CDATA[https://example.com/tracking/firstQuartile]]></Tracking>
              <Tracking event="midpoint"><![CDATA[https://example.com/tracking/midpoint]]></Tracking>
              <Tracking event="thirdQuartile"><![CDATA[https://example.com/tracking/thirdQuartile]]></Tracking>
              <Tracking event="complete"><![CDATA[https://example.com/tracking/complete]]></Tracking>
            </TrackingEvents>
            <Duration>00:00:16</Duration>
            <MediaFiles>
              <MediaFile id="5241" delivery="progressive" type="video/mp4" bitrate="2000" width="1280" height="720" minBitrate="1500" maxBitrate="2500" scalable="1" maintainAspectRatio="1" codec="H.264">
                <![CDATA[https://iab-publicfiles.s3.amazonaws.com/vast/VAST-4.0-Short-Intro.mp4]]>
              </MediaFile>
            </MediaFiles>
            <VideoClicks>
              <ClickThrough id="blog">
                <![CDATA[https://iabtechlab.com]]>
              </ClickThrough>
            </VideoClicks>
          </Linear>
          <UniversalAdId idRegistry="Ad-ID">8465</UniversalAdId>
        </Creative>
      </Creatives>
    </InLine>
  </Ad>
</VAST>
```

On retrouve les différents trackings à exécuter :
* le tracking d'impression, si affichage de la PUB
* le tracking d'erreur, si la PUB n'a pu être insérée
* "firstQuartile", "midpoint", "thirdQuartile", "complete" : selon l'avancement de lecture du spot

On trouve aussi l'url du spot de PUB dans la balise "MediaFile" au format MP4 dans le retour Freewheel, réécrit ensuite en DASH et HLS suite à la réécriture de Wizads / C3PO.

### Manipulation des manifests DASH

Pour réaliser l'insertion des spots en DASH, il est nécessaire de tronquer la période d'ad slate qui sert de filler si la durée de la publicité est inférieure à la durée de l'ad slate initial.

Pour ce faire, il faut modifier la valeur du champ "presentationTimeOffset" du SegmentTemplate comme expliqué [ici](https://dashif-documents.azurewebsites.net/DASH-IF-IOP/master/DASH-IF-IOP.html#addressing-explicit-startpoint).

La durée de l'ad slate tronqué doit être égale à : "durée ad slate initiale" - "durée tunnel de PUB". La durée du tunnel de publicité est inférieure à l'ad slate, l'ad server reçoit un paramètre pour garantir ce comportement. Autrement, les spots en trop sont ignorés.

Il y a cependant une contrainte forte, le redimensionnement consiste à supprimer un certain nombre de segments de l'ad slate, mais on ne pourra pas toujours avoir la durée voulue exacte. On ne peut que prendre la jonction de deux segments la plus proche.

![adslates-cut](images/adslate-cut.drawio.svg#darkmode "ad slates cut")

Cette contrainte amène une deuxième contrainte, si le remplacement de l'adslate initial par la PUB + l'ad slate tronqué ne fait pas la même durée, cela va décaler les "start" de toutes les périodes suivantes. Il est donc nécessaire de garder une session pour chaque utilisateur pour tenir compte de ce décalage.

Une troisième contrainte vient s'ajouter, si dans une session de lecture, un grand nombre de tunnel de publicité induit un décalage trop important, la tête de lecture du player peut être amenée à sortir de la fenêtre DVR. Pour pallier cette dernière contrainte, il est important d'avoir une fenêtre DVR suffisamment grande, et de limiter les décalages au minimum en ayant des segments plus petits sur l'ad slate.

### Manipulation des playlist HLS

Il n'y a pas de complexité particulière sur HLS, il suffit de remplacer les segments de l'ad slate par ceux des spots de PUB en respectant bien la déclaration des discontinuités. Les mêmes contraintes qu'en DASH s'appliquent sur le découpage au niveau des segment et la fenêtre DVR.

### Tracking des publicités

![trackings](images/adback-trackings.drawio.svg#darkmode "trackings")

Le tracking des publicités se fait en server side, lors de la génération du flux live. Les URLs des segments de spot publicitaire sont remplacées par une URL qui pointera sur la brique Adback. 

Dans cette URL sont inclus :
* l'id de session de l'utilisateur
* l'id du tunnel publicitaire (identifié à l'aide du marqueur SCTE35)
* l'id du spot concerné
* l'index du segment

L'index du segment permet à la brique adback de déterminer la progression dans le spot et de déterminer les quartiles à déclencher ("firstQuartile", "midpoint", "thirdQuartile", "complete").

Adback pourra alors envoyer le ou les trackings adéquats en asynchrone et renvoyer une redirection (status code 302) vers le contenu du segment au player qui pourra télécharger celui-ci.

Ce mode de tracking crée une imprécision sur les quartiles car les trackings sont exécutés au téléchargement des segments et non à la lecture réelle du player. Selon la taille du buffer du player un écart plus ou moins important peut exister. 

Il donne également moins de possibilités de mesures pour les annonceurs qui souhaiteraient contrôler plus finement leur campagne (cf. [Open Measurement SDK](https://iabtechlab.com/standards/open-measurement-sdk/)).

L'avantage d'un tracking server side est qu'il ne necéssite pas de développement dans le player pour être mis en place et facilite la mise à disposition des flux FAST à des partenaires pour lesquels le player n'est pas maîtrisé.

## Conclusion

Il existe une multitude de solutions SSAI sur le marché. Plusieurs acteurs permettent également de gérer des chaînes FAST. eTF1 opère sa propre plateforme vidéo, dans notre contexte, il était pertinent de gérer nous même la delivery de ces chaînes et leur planification.

Les manipulations de manifest DASH ou playlist HLS nécessaires à la mise en place des chaînes FAST ne sont pas si complexes.

L'insertion publicitaire est plus complexe, elle exige de maintenir une session pour chaque utilisateur :
* pour gérer le décalage des périodes DASH
* pour conserver les publicités d'un utilisateur pour chaque tunnel (le manifest est rechargé régulièrement par le player en live)
* pour gérer les trackings sur les segments des spots publicitaires

Nous opérons actuellement une soixantaine de flux FAST. Notre solution "maison" permet en quelque clics dans notre [CMS](https://fr.wikipedia.org/wiki/Syst%C3%A8me_de_gestion_de_contenu) la publication d'une nouvelle chaîne avec la possibilité de gérer la pression publicitaire (durée et fréquence des ad slates).