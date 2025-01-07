---
title: Analyse d'un contenu vidéo
date: 2025-01-06T09:00:00
hero: /post/2025/video/computer-vision/images/hero.jpg
excerpt: "Comment analyser le contenu d'une vidéo, quels outils et usages ?"
authors:
  - rpinsonneau
description: "Comment analyser le contenu d'une vidéo, quels outils et usages ?"
---

## Contexte et cas d'usage

Aujourd'hui TF1 met à disposition des centaines de milliers de contenus vidéo sur sa plateforme TF1+. La plupart de ces contenus proviennent du catalogue TF1, depuis peu, certains proviennent de partenaires (TF1+ devient une plateforme d'aggrégation). Selon le type de contenu (AVOD, Quotidienne, JT ...) les meta données associées à ces contenus sont plus ou moins bien fournies. L'augmentation en volume de ce catalogue nécessite la mise en place de certaines automatisations afin de garantir la présence de certaines de ces méta données :

* Cue Point : pour déterminer les placements des coupures PUBs
* Thumbnail : pour afficher une vignette, représentative d'un épisode
* Casting : afin que le contenu soit correctement indexé dans le moteur de recherche, que les données de casting soient affichées en front
* Chapitrage : pour identifier les séquences du JT

Ces meta données peuvent être auto générées à partir du contenu vidéo.

## Selection automatique des vignettes vidéo

Dans cet article, nous allons étudier le cas d'usage de la selection automatique des vignettes vidéo (thumbnail). L'idée est de selectionner une frame de la vidéo, qui sera utilisée comme vignette de présentation (thumbnail) dans TF1+. Pour se faire, nous définissons des règles qui vont s'appuyer sur la présence d'un ou plusieurs personnages ainsi que l'émotion qu'ils dégagent.

Par exemple :
* personnage principal, calme
* personnage secondaire, souriant
* duo: personnage principal et secondaire, heureux

Pour implémenter ce type de filtrage il est nécessaire de mettre en place des outils pour :
* la detection des visages
* la reconnaissance des visages
* l'analyse des émotions

Par ailleur, afin de selectionner le meilleur visuel, il faut également déterminer les caractéristiques précises d'un visage : orientation, yeux ouverts / fermés ...


### AWS Rekognition

AWS Fournit le service [Rekognition](https://aws.amazon.com/fr/rekognition/), qui permet notemment :
* la detection / reconnaissance de visages
* detection d'objets
* detection de CUE Point (placement PUB)

Le service est fiable et donne un niveau de detail assez impressionnant, surtout sur la detection des visages :
* l'orientation du visage (pitch, roll, yaw)
* l'orientation des yeux
* yeux (ouverts / fermés)
* bouche (ouverte / fermée)
* le genre (homme / femme)
* une estimation de l'âge de la personne
* l'émotion de la personne 

La reconnaissance de visage permet d'affecter un identifiant à chaque personne présente dans une vidéo. Il est possible également de rechercher dans un bucket d'images indexées des visages d'une même personne.

Le tarif d'AWS rekognition est calculé à la minute de vidéo analysé, il est de 0,10 USD/min pour la detection de visage, soit 6$ pour l'analyse d'une heure de vidéo. Il est également possible d'utiliser le service pour l'analyse d'une seule image, dans ce cas, le tarif est dégressif et commence à 0,001$ pour la detection, l'indexation et la recherche de visages.

Ci dessous quelques exemples d'analyses d'images avec AWS rekognition :

* Frame extraite du programme "Demain nous appartient" - Camille :

![Camille](images/raw_group20_scene20_frame2830_time_1m53.2s.jpg "Camille - Demain nous appartient")

```json
{
  "FaceModelVersion": "7.0",
  "FaceRecords": [
    {
      "Face": {
        "BoundingBox": {
          "Height": 0.36938998,
          "Left": 0.4648164,
          "Top": 0.15837377,
          "Width": 0.14581785
        },
        "Confidence": 99.99991,
        "ExternalImageId": "group20_scene20_frame2830_time_1m53.2s",
        "FaceId": "ea9af6bd-f1b4-460a-86a0-fd32608a8db2",
        "ImageId": "a1bee274-2e8c-378f-9e6b-5ac92e25bdf6",
        "IndexFacesModelVersion": null,
        "UserId": null
      },
      "FaceDetail": {
        "AgeRange": {
          "High": 27,
          "Low": 21
        },
        "Beard": {
          "Confidence": 99.327,
          "Value": false
        },
        "BoundingBox": {
          "Height": 0.36938998,
          "Left": 0.4648164,
          "Top": 0.15837377,
          "Width": 0.14581785
        },
        "Confidence": 99.99991,
        "Emotions": [
          {
            "Confidence": 99.674484,
            "Type": "HAPPY"
          },
          {
            "Confidence": 0.029698014,
            "Type": "SURPRISED"
          },
          {
            "Confidence": 0.024700165,
            "Type": "CALM"
          },
          {
            "Confidence": 0.004152457,
            "Type": "CONFUSED"
          },
          {
            "Confidence": 0.0021755695,
            "Type": "SAD"
          },
          {
            "Confidence": 0.0009357929,
            "Type": "DISGUSTED"
          },
          {
            "Confidence": 0.00067055225,
            "Type": "FEAR"
          },
          {
            "Confidence": 0.00024437904,
            "Type": "ANGRY"
          }
        ],
        "EyeDirection": {
          "Confidence": 99.99743,
          "Pitch": -11.714278,
          "Yaw": 4.179036
        },
        "Eyeglasses": {
          "Confidence": 99.83311,
          "Value": false
        },
        "EyesOpen": {
          "Confidence": 99.94284,
          "Value": true
        },
        "FaceOccluded": {
          "Confidence": 99.72817,
          "Value": false
        },
        "Gender": {
          "Confidence": 99.88475,
          "Value": "Female"
        },
        "Landmarks": [
          {
            "Type": "eyeLeft",
            "X": 0.51439583,
            "Y": 0.3072095
          },
          {
            "Type": "eyeRight",
            "X": 0.58142936,
            "Y": 0.30300102
          },
          {
            "Type": "mouthLeft",
            "X": 0.52031505,
            "Y": 0.4340107
          },
          {
            "Type": "mouthRight",
            "X": 0.5762489,
            "Y": 0.42992982
          },
          {
            "Type": "nose",
            "X": 0.5613297,
            "Y": 0.37382948
          },
          {
            "Type": "leftEyeBrowLeft",
            "X": 0.48439318,
            "Y": 0.2788651
          },
          {
            "Type": "leftEyeBrowRight",
            "X": 0.5327073,
            "Y": 0.2687561
          },
          {
            "Type": "leftEyeBrowUp",
            "X": 0.5108063,
            "Y": 0.26297098
          },
          {
            "Type": "rightEyeBrowLeft",
            "X": 0.57106155,
            "Y": 0.2668129
          },
          {
            "Type": "rightEyeBrowRight",
            "X": 0.60086155,
            "Y": 0.27194983
          },
          {
            "Type": "rightEyeBrowUp",
            "X": 0.5880483,
            "Y": 0.2588775
          },
          {
            "Type": "leftEyeLeft",
            "X": 0.5006436,
            "Y": 0.30724815
          },
          {
            "Type": "leftEyeRight",
            "X": 0.5274707,
            "Y": 0.30748418
          },
          {
            "Type": "leftEyeUp",
            "X": 0.51479876,
            "Y": 0.30083188
          },
          {
            "Type": "leftEyeDown",
            "X": 0.51464856,
            "Y": 0.3128167
          },
          {
            "Type": "rightEyeLeft",
            "X": 0.5679361,
            "Y": 0.30504102
          },
          {
            "Type": "rightEyeRight",
            "X": 0.59146637,
            "Y": 0.30155528
          },
          {
            "Type": "rightEyeUp",
            "X": 0.58209264,
            "Y": 0.29671252
          },
          {
            "Type": "rightEyeDown",
            "X": 0.58103305,
            "Y": 0.3085822
          },
          {
            "Type": "noseLeft",
            "X": 0.54039043,
            "Y": 0.3878802
          },
          {
            "Type": "noseRight",
            "X": 0.5651958,
            "Y": 0.38635063
          },
          {
            "Type": "mouthUp",
            "X": 0.5540638,
            "Y": 0.41727102
          },
          {
            "Type": "mouthDown",
            "X": 0.552262,
            "Y": 0.45527056
          },
          {
            "Type": "leftPupil",
            "X": 0.51439583,
            "Y": 0.3072095
          },
          {
            "Type": "rightPupil",
            "X": 0.58142936,
            "Y": 0.30300102
          },
          {
            "Type": "upperJawlineLeft",
            "X": 0.45368963,
            "Y": 0.30953178
          },
          {
            "Type": "midJawlineLeft",
            "X": 0.47044832,
            "Y": 0.44600865
          },
          {
            "Type": "chinBottom",
            "X": 0.5466232,
            "Y": 0.5203043
          },
          {
            "Type": "midJawlineRight",
            "X": 0.5890297,
            "Y": 0.4391184
          },
          {
            "Type": "upperJawlineRight",
            "X": 0.5997778,
            "Y": 0.30139688
          }
        ],
        "MouthOpen": {
          "Confidence": 88.61521,
          "Value": true
        },
        "Mustache": {
          "Confidence": 99.89976,
          "Value": false
        },
        "Pose": {
          "Pitch": -3.9920487,
          "Roll": -0.19200581,
          "Yaw": 14.684415
        },
        "Quality": {
          "Brightness": 39.181137,
          "Sharpness": 78.6435
        },
        "Smile": {
          "Confidence": 98.989685,
          "Value": true
        },
        "Sunglasses": {
          "Confidence": 99.64001,
          "Value": false
        }
      }
    }
  ],
  "OrientationCorrection": "",
  "UnindexedFaces": [],
  "ResultMetadata": {}
}
```

* Frame extraite du programme "Demain nous appartient" - Simon :

![Simon](images/raw_group9_scene19_frame2757_time_1m50.28s.jpg "Simon - Demain nous appartient")

```json
{
  "FaceModelVersion": "7.0",
  "FaceRecords": [
    {
      "Face": {
        "BoundingBox": {
          "Height": 0.5073994,
          "Left": 0.57771593,
          "Top": 0.15709291,
          "Width": 0.1947689
        },
        "Confidence": 99.99969,
        "ExternalImageId": "group9_scene19_frame2757_time_1m50.28s",
        "FaceId": "6b903cf6-3d45-440d-a472-b2ec758b52ad",
        "ImageId": "e43f1ef7-1029-3021-8cb0-bc4a3dc6c50f",
        "IndexFacesModelVersion": null,
        "UserId": null
      },
      "FaceDetail": {
        "AgeRange": {
          "High": 28,
          "Low": 22
        },
        "Beard": {
          "Confidence": 99.83963,
          "Value": true
        },
        "BoundingBox": {
          "Height": 0.5073994,
          "Left": 0.57771593,
          "Top": 0.15709291,
          "Width": 0.1947689
        },
        "Confidence": 99.99969,
        "Emotions": [
          {
            "Confidence": 82.02637,
            "Type": "CALM"
          },
          {
            "Confidence": 3.7963867,
            "Type": "HAPPY"
          },
          {
            "Confidence": 0.8710225,
            "Type": "CONFUSED"
          },
          {
            "Confidence": 0.6752014,
            "Type": "SURPRISED"
          },
          {
            "Confidence": 0.18806458,
            "Type": "SAD"
          },
          {
            "Confidence": 0.09021759,
            "Type": "DISGUSTED"
          },
          {
            "Confidence": 0.03452301,
            "Type": "ANGRY"
          },
          {
            "Confidence": 0.0192523,
            "Type": "FEAR"
          }
        ],
        "EyeDirection": {
          "Confidence": 99.9961,
          "Pitch": -4.282108,
          "Yaw": -41.371216
        },
        "Eyeglasses": {
          "Confidence": 99.346664,
          "Value": false
        },
        "EyesOpen": {
          "Confidence": 99.8771,
          "Value": true
        },
        "FaceOccluded": {
          "Confidence": 99.911705,
          "Value": false
        },
        "Gender": {
          "Confidence": 99.69594,
          "Value": "Male"
        },
        "Landmarks": [
          {
            "Type": "eyeLeft",
            "X": 0.6221226,
            "Y": 0.35962352
          },
          {
            "Type": "eyeRight",
            "X": 0.7024969,
            "Y": 0.37882823
          },
          {
            "Type": "mouthLeft",
            "X": 0.62099516,
            "Y": 0.5224739
          },
          {
            "Type": "mouthRight",
            "X": 0.6880223,
            "Y": 0.53904825
          },
          {
            "Type": "nose",
            "X": 0.64596355,
            "Y": 0.46686035
          },
          {
            "Type": "leftEyeBrowLeft",
            "X": 0.5980547,
            "Y": 0.3112588
          },
          {
            "Type": "leftEyeBrowRight",
            "X": 0.63695276,
            "Y": 0.32009196
          },
          {
            "Type": "leftEyeBrowUp",
            "X": 0.6162148,
            "Y": 0.3041741
          },
          {
            "Type": "rightEyeBrowLeft",
            "X": 0.68303853,
            "Y": 0.3303982
          },
          {
            "Type": "rightEyeBrowRight",
            "X": 0.7381201,
            "Y": 0.34361362
          },
          {
            "Type": "rightEyeBrowUp",
            "X": 0.7091143,
            "Y": 0.3251264
          },
          {
            "Type": "leftEyeLeft",
            "X": 0.6092563,
            "Y": 0.3539567
          },
          {
            "Type": "leftEyeRight",
            "X": 0.63794935,
            "Y": 0.36476493
          },
          {
            "Type": "leftEyeUp",
            "X": 0.62157404,
            "Y": 0.35207558
          },
          {
            "Type": "leftEyeDown",
            "X": 0.62197393,
            "Y": 0.3667092
          },
          {
            "Type": "rightEyeLeft",
            "X": 0.68647975,
            "Y": 0.3762255
          },
          {
            "Type": "rightEyeRight",
            "X": 0.7183587,
            "Y": 0.37976536
          },
          {
            "Type": "rightEyeUp",
            "X": 0.70233184,
            "Y": 0.3711335
          },
          {
            "Type": "rightEyeDown",
            "X": 0.7016881,
            "Y": 0.3856617
          },
          {
            "Type": "noseLeft",
            "X": 0.6371298,
            "Y": 0.47295845
          },
          {
            "Type": "noseRight",
            "X": 0.6670698,
            "Y": 0.47987586
          },
          {
            "Type": "mouthUp",
            "X": 0.64920145,
            "Y": 0.5155665
          },
          {
            "Type": "mouthDown",
            "X": 0.6485316,
            "Y": 0.56190246
          },
          {
            "Type": "leftPupil",
            "X": 0.6221226,
            "Y": 0.35962352
          },
          {
            "Type": "rightPupil",
            "X": 0.7024969,
            "Y": 0.37882823
          },
          {
            "Type": "upperJawlineLeft",
            "X": 0.5927531,
            "Y": 0.3330891
          },
          {
            "Type": "midJawlineLeft",
            "X": 0.598198,
            "Y": 0.5115856
          },
          {
            "Type": "chinBottom",
            "X": 0.6497905,
            "Y": 0.639116
          },
          {
            "Type": "midJawlineRight",
            "X": 0.74155307,
            "Y": 0.5440322
          },
          {
            "Type": "upperJawlineRight",
            "X": 0.7688209,
            "Y": 0.3728635
          }
        ],
        "MouthOpen": {
          "Confidence": 97.98864,
          "Value": false
        },
        "Mustache": {
          "Confidence": 82.85489,
          "Value": true
        },
        "Pose": {
          "Pitch": -0.8480164,
          "Roll": 3.6424594,
          "Yaw": -12.758622
        },
        "Quality": {
          "Brightness": 45.051098,
          "Sharpness": 86.86019
        },
        "Smile": {
          "Confidence": 88.01531,
          "Value": false
        },
        "Sunglasses": {
          "Confidence": 99.46419,
          "Value": false
        }
      }
    }
  ],
  "OrientationCorrection": "",
  "UnindexedFaces": [],
  "ResultMetadata": {}
}
```


### Approche hybride

Afin de réduire les coûts d'AWS rekognition, nous avons adopté une approche hybride utilisant à la fois des outils open source, AWS rekognition et AWS bedrock.

![Hybride](images/hybride.svg#darkmode "Approche hybride")

La stratégie est la suivante, les outils open souce permettent un premier filtrage sur les frames afin de ne conserver qu'un nombre réduit de candidats. AWS Rekognition est ensuite utilisé, image par image, pour établir un second filtrage, plus fin. Sur les dernieres frames, AWS Bedrock permet l'utilisation d'un LLM multi modal pour selectionner le meilleur visuel en fonction de critères liés au programme de la vidéo.

### Détection de scènes

La detection des scènes permet d'identifier les discontinuitées dans le contenu vidéo, correspondantes a des changement de plan.

L'idée est de limiter le nombre d'analyses par plan, en effet, beaucoup de scènes sont relativement statiques lors de dialogues. Cette affirmation est plus ou moins vérifiée selon le programme, elle est tout a fait pertinante pour des programmes tel que "Ici tout commence", "Demain nous appartient", "Plus belle la vie" et la plupart des quotidiennes de TF1 où les plans sont très statiques et sont une succession de dialogues. C'est un peu moins évidant pour certains programmes comme Koh-Lanta ou les scènes sont plus mouvante avec des transitions parfois moins marquées. 

Pour une scene donnée, les personnages présents sont en général les mêmes du début à la fin de la scène. Ceci permet de limiter le nombre d'analyse en ne conservant que N candidats par scene tout en conservant l'exhaustivité des plans.

La commande ffmpeg ci dessous permet de détecter les scènes :

```shell
ffmpeg -i 14188512.mp4 -filter:v "select='gt(scene,0.3)',showinfo" -fps_mode vfr -f null -
```


Ici, "0.3" est le seuil de détection des scènes.

La commande va générer en sortie d'erreur les logs suivants :


```shell
[Parsed_showinfo_1 @ 0x6000019dc160] n:   0 pts: 548000 pts_time:21.92   duration:   1000 duration_time:0.04    fmt:yuv420p cl:left sar:0/1 s:1920x1080 i:P iskey:0 type:P checksum:4D9D3276 plane_checksum:[214FFDD1 8F8EF20E E0224279] mean:[104 121 130] stdev:[57.2 10.8 7.7]
[Parsed_showinfo_1 @ 0x6000019dc160] n:   1 pts: 709000 pts_time:28.36   duration:   1000 duration_time:0.04    fmt:yuv420p cl:left sar:0/1 s:1920x1080 i:P iskey:0 type:B checksum:1C96B95B plane_checksum:[C829B67C 37D11037 9ED7F299] mean:[98 121 129] stdev:[55.8 11.8 8.5]
[Parsed_showinfo_1 @ 0x6000019dc160] n:   2 pts: 844000 pts_time:33.76   duration:   1000 duration_time:0.04    fmt:yuv420p cl:left sar:0/1 s:1920x1080 i:P iskey:0 type:P checksum:417A4441 plane_checksum:[1FF62C11 4ABACA08 183F4E19] mean:[90 122 132] stdev:[52.1 9.1 8.2]
```

pts_time est la position en seconde dans la vidéo du changement de scène.

La vidéo a 25 frames par secondes (fps) :
duration_time = 1 / 25 = 0.04

pts_time = (pts * duration_time) / duration

Pour le premier changement de plan (n=0), on obtient :
pts_time = (548000 * 0.04) / 1000 = 21.92 (en secondes)



### Regroupement des scènes similaires

Lors de dialogues, la caméra peut faire des changements répétitifs entre deux personnages qui au final vont correspondre a deux plans qui vont s'alterner. Regrouper les scènes similaires permet de réduire encore un peu plus le nombre d'analyses nécessaires.

Pour se faire, il est possible de comparer la première frame d'une scène avec la dernière frame des scènes précédantes. SSIM (Structural SIMilarity) ou PSNR (Peak Signal to Noise Ratio) sont deux métriques qui permettent de mesurer la similarité entre deux images. 

![Groups](images/groups.svg#darkmode "Regroupement des scènes")

* les scènes 8 et 10 sont regroupées, il s'agit du même plan sur Camille
* les scènes 9 et 11 sont regroupées, il s'agit du même plan sur Simon

OpenCV permet l'implémentation de ces deux métriques, au delà d'un seuil paramétrable, les deux scène seront regroupées. Un exemple d'implémentation avec OpenCV est disponible [ici](https://docs.opencv.org/4.x/d5/dc4/tutorial_video_input_psnr_ssim.html).

Nos stacks sont développées en Go, il existe un binding Go pour OpenCV : [gocv](https://gocv.io/).

Un exemple d'implémentation de SSIM avec gocv [ici](https://gist.github.com/rpinsonneau/7df55569cf2f784582ce9743286c4ed3).

### Métrique de qualité de l'image

Une fois les scènes détectées et regroupées, il va falloir déterminer une métrique pour conserver les N candidats de chaque groupe de scènes. Le niveau de flou ou de netteté de l'image est un critère important de selection.

Une solution simple pour obtenir cette métrique est de convertir la frame en niveau de gris puis d'appliquer un filtre laplacien. Le filtre laplacien va mettre en évidance les contours des objets, plus l'image est nette plus les contours seront marqués. La variance est une mesure de la dispersion, en calculant celle-ci sur l'image obtenu on obtient un indicateur sur le niveau de détail et de nettetée de l'image : plus la dispersion est importante, plus il y a de contours détaillés plus l'image est nette.

![Laplacian](images/laplacian.jpg "Filtre laplacien")

Ci dessous un exemple de code avec gocv pour calculer la variance du filtre laplacien :

```go
func ComputeLaplacianVariance(input gocv.Mat) float64 {

	// convert to grayscale
	grayscale := gocv.NewMat()
	defer grayscale.Close()
	gocv.CvtColor(input, &grayscale, gocv.ColorBGRToGray)

	// compute laplacian from grayscale
	laplacian := gocv.NewMat()
	defer laplacian.Close()
	gocv.Laplacian(grayscale, &laplacian, gocv.MatTypeCV64F, 1, 1, 0, gocv.BorderDefault)

	// compute mean and stddev of laplacian
	mean := gocv.NewMat()
	defer mean.Close()
	stddev := gocv.NewMat()
	defer stddev.Close()
	gocv.MeanStdDev(laplacian, &mean, &stddev)
	variance := math.Pow(float64(stddev.GetDoubleAt(0, 0)), 2)

	return variance
}
```

### Detection des visages

### Scoring des frames

### Selection de la meilleure frame

## Conclusion