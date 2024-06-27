---
title: Déploiement d'un LLM à l'echelle avec TGI
date: 2024-06-26T09:00:00
hero: /post/2024/ia/inference-llm-tgi/images/hero.jpg
excerpt: "Découvrez comment nous réalisons l'inference de nos LLMs en production."
authors:
  - rpinsonneau
description: "Découvrez comment nous réalisons l'inference de nos LLMs en production."
---

## L'inference d'un Large Language Model

Les LLMs (Large Language Model) sont de plus en plus adoptés en entreprise, leur coût, leur mise à l'echelle en production ou la confidentialité des données peuvent être un véritable challenge.

La solution la plus simple pour réaliser l'inférence d'un modèle est de payer une solution clé en main :
* [openAI (chatGPT)](https://platform.openai.com/docs)
* [Google (Gemini)](https://ai.google.dev)
* [Anthropic (Claude)](https://docs.anthropic.com/en/api)

Certains services, comme [AWS Bedrock](https://docs.aws.amazon.com/bedrock/latest/userguide/what-is-bedrock.html) permettent de déployer différents modèles.

![AWS Bedrock](images/aws-bedrock.png#darkmode "Modèles d'AWS Bedrock")

Souvent, le pricing se fera en fonction du nombre de tokens (input + output), ce qui n'aide pas a se projeter sur le coût réel de la solution. La confidentialité des données est également un point de vigilence d'un point de vu RGPD. En utilisant une solution SAS, on entre dans une logique de "vendor lock-in", on perd également la maitrise sur les potentielles mises à jour des modèles.

L'autre possibilité est de déployer soit même un modèle open source:
* [Mistral](https://mistral.ai/technology/#models)
* [LLama](https://llama.meta.com/llama3/)
* [Phi3](https://azure.microsoft.com/en-us/products/phi-3)

Chaque modèle vient avec sa façon de réaliser l'inference, il existe cependant des outils qui permettent d'abstraire le déploiement de ces modèles sans distinction:
* [Ollama](https://ollama.com/)
* [llama.cpp](https://github.com/ggerganov/llama.cpp)
* [vLLM](https://docs.vllm.ai/en/stable/)
* [TGI (Text Generation Inference)](https://github.com/huggingface/text-generation-inference)
* [TEI (Text Embeddings Inference)](https://github.com/huggingface/text-embeddings-inference)

### Hugging Face & TGI

A eTF1, nous utilisons les solutions d'[hugging face](https://huggingface.co/) sur nos environnement AWS. [Ollama](https://ollama.com/) est également utilisé sur les postes des developpeurs pour du prototypage.

[Hugging face](https://huggingface.co/) est un hub, qui permet de partager des datasets et des modèles à la comunauté. C'est aussi un ensemble de bibliothèques qui permettent l'entrainement et l'inference de ces modèles.

![Hugging Face](images/huggingface.png#darkmode "Hugging Face Hub")

[TGI](https://github.com/huggingface/text-generation-inference) (Text Generation Inference) permet de déployer facilement un modèle, fourni sous forme d'image docker, [TGI](https://github.com/huggingface/text-generation-inference) peut être facilement déployé sur une infrastructure existante.

L'inference d'un LLM nécessite cependant l'utilisation de hardware spécifique, [TGI](https://github.com/huggingface/text-generation-inference) permet l'inference sur différents matériels :
* Une carte graphique Nvidia, AMD ou Intel: sur AWS, à Paris (zone eu-west-3), les instances EC2 de type [G4dn](https://aws.amazon.com/fr/ec2/instance-types/g4/) sont disponibles avec des cartes Nvidia T4 avec 16GB de VRAM (~0,6$ de l'heure pour un g4dn.xlarge).
* Une carte d'acceleration spécifique Inferentia, Gaudi, TPU: sur AWS, les instances EC2 de type [Inf2](https://aws.amazon.com/fr/ec2/instance-types/inf2/) sont disponibles avec des cartes Inferentia2 avec 2x16GB de mémoire (~1$ de l'heure pour inf2.xlarge)

Pour démarrer l'inference du model il suffit de démarrer le contener TGI. Sur AWS, nous déployons ces conteneurs dans un cluster [EKS](https://aws.amazon.com/fr/eks/). Nous avons une configuration spécifique de [Karpenter](https://karpenter.sh/) pour provisionner des instances EC2 de type [G4dn](https://aws.amazon.com/fr/ec2/instance-types/g4/) ou [Inf2](https://aws.amazon.com/fr/ec2/instance-types/inf2/) sur ces déploiements.

Exemple de commande docker :
```shell
docker run --rm -p 8080:80 \
       -v $(pwd)/data:/data \
       --gpus all \
       -e HUGGING_FACE_HUB_TOKEN=${HF_TOKEN} \
       ghcr.io/huggingface/text-generation-inference:2.0.4 \
       --model-id=mistralai/Mistral-7B-Instruct-v0.3
```

Pour tester llama3 au lieu de mistral il suffit de changer l'argument --model-id. Les id des modèles sont indiqués sur le hub hugging face.

Au démarrage, TGI va télécharger dans un volume le modèle spécifié si celui-ci n'est pas déjà présent.

### Quantization

En plus d'un hardware sépcifique il faudra être vigilent à la quantité de mémoire necessaire pour faire tourner le modèle. On parle bien ici de mémoire GPU (ou mémoire de la carte d'accelération) et non de RAM.

Ci-dessous les prérequis pour faire tourner les différentes variantes de mistral :

| Name               | Number of parameters | Number of active parameters | Min. GPU RAM for inference (GB) |
| ------------------ | -------------------- | --------------------------- | ------------------------------- |
| Mistral-7B-v0.3    | 7.3B                 | 7.3B                        | 16                              |
| Mixtral-8x7B-v0.1  | 46.7B                | 12.9B                       | 100                             |
| Mixtral-8x22B-v0.3 | 140.6B               | 39.1B                       | 300                             |
| Codestral-22B-v0.1 | 22.2B                | 22.2B                       | 60                              |

Heureusement il existe plusieurs techniques pour rendre l'inference possible sur du matériel relativement modeste ou grand public : le sharding et la [quantization](https://huggingface.co/docs/optimum/concept_guides/quantization).

Lorsque TGI a terminé le téléchargement d'un modèle dans son volume, il commence ensuite le chargement de celui-ci dans la mémoire de l'accelerateur. Le modèle est souvent sauvegardé sous forme de fichier binaire, par exemple [pytorch](https://pytorch.org/tutorials/beginner/saving_loading_models.html) ou [safetensors](https://github.com/huggingface/safetensors).

![Mistral model files](images/mistral-files.png#darkmode "Mistral model files")

Ces fichiers volumineux contiennent les fameux paramètres du modèle. En réalité il s'agit de matrices contenant les différents paramètres sous forme de float. On peut trouver facielement la précision des floats d'un modèle en consultant le [fichier de configuration](https://huggingface.co/mistralai/Mistral-7B-Instruct-v0.3/blob/main/config.json) d'hugging face du modèle. Pour mistral et llama, l'attribut "torch_dtype" a comme valeur ["bfloat16"](https://pytorch.org/docs/stable/tensors.html).

En réduisant la précision des floats d'un modèle on réduit la taille du modèle. Mistral-7B-v0.3 nécessite 16GB de VRAM pour l'inference en 16 bits, la quantité de mémoire necessaire en 8 bit ou 4 bit sera deux ou quatre fois moins élevé.

Sur Mac, avec ollama, les modèles sont en général en 4 bit (ça tourne dans la mémoire unifié), les Nvidia supportent la quantization, sur inferentia ce n'est pas le cas. Il faut donc vérifier les contraintes en fonction du hardware. De plus il existe plusieurs implémentation pour quantizer un modèle, certaines nécessitent une variante du modèle d'origine : 
* [bitsandbytes](https://github.com/TimDettmers/bitsandbytes): fonctionne avec tous les modèles mais peut être plus lent
* [EETQ](https://github.com/NetEase-FuXi/EETQ): 8 bits, fonctionne avec tous les modèles
* [AWQ](https://github.com/casper-hansen/AutoAWQ): 4 bits, nécessite un modèle [spécifique](https://hf.co/models?search=awq)

Par exemple pour une quantization 8 bits sur T4 avec eetq :
```shell
docker run --rm -p 8080:80 \
       -v $(pwd)/data:/data \
       --gpus all \
       -e HUGGING_FACE_HUB_TOKEN=${HF_TOKEN} \
       ghcr.io/huggingface/text-generation-inference:2.0.4 \
       --model-id=mistralai/Mistral-7B-Instruct-v0.3 \
       --quantize eetq \
```

## Sharding & inferentia

Sur les instances inferentia2 d'AWS 2 cores sont disponibles sur les inf2.xlarge. On a donc 2x16GB, pour exploiter pleinement les 32GB disponibles il faut couper le modèle en deux.

![TGI architecture](images/tgi-sharding.png#darkmode "TGI architecture")

TGI supporte nativement le sharding et va donc exploiter les deux cores disponibles.

Exemple de commande docker pour mistral sur inferentia2 inf2.xlarge
```shell
docker run --rm -p 8080:80 \
       -v $(pwd)/data:/data \
       --device=/dev/neuron0 \
       -e HF_TOKEN=${HF_TOKEN} \
       -e HF_BATCH_SIZE=1 \
       -e HF_SEQUENCE_LENGTH=4096 \
       -e HF_AUTO_CAST_TYPE="fp16" \
       -e HF_NUM_CORES=2 \
       ghcr.io/huggingface/neuronx-tgi:latest \
       --model-id=mistralai/Mistral-7B-Instruct-v0.3 \
       --max-batch-size 1 \
       --max-input-length 3686 \
       --max-batch-prefill-tokens 3686 \
       --max-total-tokens 4096
```

Pour lancer TGI sur inferentia2 il est nécessaire d'utiliser une image docker [spécifique](https://github.com/huggingface/optimum-neuron/tree/main/text-generation-inference). D'autre part, le modèle doit être recompilé pour tourner sur inferentia a l'aide d'un outil : [Neuron Compiler](https://awsdocs-neuron.readthedocs-hosted.com/en/latest/compiler/index.html). Hugging face dispose d'un cache avec les modèles précompilé, par exemple pour mistral on trouve la liste [ici](https://huggingface.co/aws-neuron/optimum-neuron-cache/blob/main/inference-cache-config/mistral.json). Les paramètres HF_BATCH_SIZE, HF_SEQUENCE_LENGTH, HF_AUTO_CAST_TYPE et HF_NUM_CORES doivent correspondre au paramètres utilisés lors de la compilation.






