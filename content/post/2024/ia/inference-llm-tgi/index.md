---
title: Déploiement d'un LLM à l'échelle avec TGI
date: 2024-06-26T09:00:00
hero: /post/2024/ia/inference-llm-tgi/images/hero.jpg
excerpt: "Découvrez comment nous réalisons l'inférence de nos LLMs en production."
authors:
  - rpinsonneau
description: "Découvrez comment nous réalisons l'inférence de nos LLMs en production."
---

## L'inférence d'un Large Language Model

Les LLMs (_Large Language Model_) sont de plus en plus adoptés en entreprise, leur coût, leur mise à l'échelle en production ou la confidentialité des données peuvent être un véritable défi.

La solution la plus simple pour réaliser l'inférence d'un modèle consiste à payer une solution clé en main, telle que :
* [openAI (chatGPT)](https://platform.openai.com/docs)
* [Google (Gemini)](https://ai.google.dev)
* [Anthropic (Claude)](https://docs.anthropic.com/en/api)

Certains services, comme [AWS Bedrock](https://docs.aws.amazon.com/bedrock/latest/userguide/what-is-bedrock.html) permettent de déployer différents modèles.

![AWS Bedrock](images/aws-bedrock.png#darkmode "Modèles d'AWS Bedrock")

Souvent, le prix se fera en fonction du nombre de tokens (input + output), ce qui n'aide pas à se projeter sur le coût réel de la solution. La confidentialité des données est également un point de vigilence d'un point de vu RGPD. En utilisant une solution SaaS, nous entrons dans une logique de "vendor lock-in", et nous perdons également la maîtrise sur les potentielles mises à jour des modèles.

L'autre possibilité est de déployer soi-même un modèle Open Source :
* [Mistral](https://mistral.ai/technology/#models)
* [LLama](https://llama.meta.com/llama3/)
* [Phi3](https://azure.microsoft.com/en-us/products/phi-3)

Chaque modèle vient avec sa façon de réaliser l'inférence, il existe cependant des outils qui permettent d'abstraire le déploiement de ces modèles sans distinction:
* [Ollama](https://ollama.com/)
* [llama.cpp](https://github.com/ggerganov/llama.cpp)
* [vLLM](https://docs.vllm.ai/en/stable/)
* [TGI (Text Generation Inference)](https://github.com/huggingface/text-generation-inference)
* [TEI (Text Embeddings Inference)](https://github.com/huggingface/text-embeddings-inference)

### Hugging Face & TGI

À eTF1, nous utilisons les solutions d'[hugging face](https://huggingface.co/) sur nos environnements AWS. [Ollama](https://ollama.com/) est également utilisé sur les postes de développement pour du prototypage.

[Hugging face](https://huggingface.co/) est un hub, qui permet de partager des datasets et des modèles à la communauté. C'est aussi un ensemble de bibliothèques qui permettent l'entraînement et l'inférence de ces modèles.

![Hugging Face](images/huggingface.png#darkmode "Hugging Face Hub")

[TGI](https://github.com/huggingface/text-generation-inference) (_Text Generation Inference_) permet de déployer facilement un modèle, fourni sous forme d'image Docker, [TGI](https://github.com/huggingface/text-generation-inference) peut être facilement déployé sur une infrastructure existante.

L'inférence d'un LLM nécessite cependant l'utilisation d'une configuration matérielle (_hardware_) spécifique, [TGI](https://github.com/huggingface/text-generation-inference) permet l'inférence sur différents types de matériels :
* Une carte graphique NVIDIA, AMD ou Intel : sur AWS, à Paris (zone eu-west-3), les instances EC2 de type [G4dn](https://aws.amazon.com/fr/ec2/instance-types/g4/) sont disponibles avec des cartes NVIDIA T4 avec 16GB de VRAM pour ~0,6$ de l'heure pour un g4dn.xlarge.
* Une carte d'accélération spécifique Inferentia, Gaudi, TPU : sur AWS, les instances EC2 de type [Inf2](https://aws.amazon.com/fr/ec2/instance-types/inf2/) sont disponibles avec des cartes Inferentia2 avec 2x16GB de mémoire pour ~1$ de l'heure pour inf2.xlarge.

Pour démarrer l'inférence du modèle, il suffit de démarrer le conteneur TGI. Sur AWS, nous déployons ces conteneurs dans un cluster [EKS](https://aws.amazon.com/fr/eks/). Nous avons une configuration spécifique de [Karpenter](https://karpenter.sh/) pour provisionner des instances EC2 de type [G4dn](https://aws.amazon.com/fr/ec2/instance-types/g4/) ou [Inf2](https://aws.amazon.com/fr/ec2/instance-types/inf2/) sur ces déploiements.

Exemple de commande docker :
```shell
docker run --rm -p 8080:80                                 \
       -v $(pwd)/data:/data                                \
       --gpus all                                          \
       -e HUGGING_FACE_HUB_TOKEN=${HF_TOKEN}               \
       ghcr.io/huggingface/text-generation-inference:2.0.4 \
       --model-id=mistralai/Mistral-7B-Instruct-v0.3
```

Pour tester Meta Llama 3 au lieu de Mistral il suffit de changer l'argument `--model-id`. Les identifiants des modèles sont indiqués sur le hub Hugging Face.

Au démarrage, TGI va télécharger dans un volume (`/data`) le modèle spécifié si celui-ci n'est pas déjà présent.

### Quantization

En plus d'une configuration matérielle spécifique, il faudra être vigilent à la quantité de mémoire nécessaire pour exécuter le modèle. Nous parlons bien ici de mémoire GPU (ou mémoire de la carte d'accelération) et non de RAM.

Ci-dessous les prérequis pour exécuter les différentes variantes de Mistral :

| Name               | Number of parameters | Number of active parameters | Min. GPU RAM for inference (GB) |
| ------------------ | -------------------- | --------------------------- | ------------------------------- |
| Mistral-7B-v0.3    | 7.3B                 | 7.3B                        | 16                              |
| Mixtral-8x7B-v0.1  | 46.7B                | 12.9B                       | 100                             |
| Mixtral-8x22B-v0.3 | 140.6B               | 39.1B                       | 300                             |
| Codestral-22B-v0.1 | 22.2B                | 22.2B                       | 60                              |

Heureusement, il existe plusieurs techniques pour rendre l'inférence possible sur du matériel relativement modeste ou grand public (_commodity hardware_) : le sharding et la [quantization](https://huggingface.co/docs/optimum/concept_guides/quantization).

Lorsque TGI a terminé le téléchargement d'un modèle dans son volume, il commence ensuite le chargement de celui-ci dans la mémoire de la carte d'accélération.

![nvidia-smi](images/nvidia-smi.png#darkmode "nvidia-smi")

Le modèle est souvent sauvegardé sous forme de fichier binaire, par exemple [pytorch](https://pytorch.org/tutorials/beginner/saving_loading_models.html) ou [safetensors](https://github.com/huggingface/safetensors).

![Mistral model files](images/mistral-files.png#darkmode "Mistral model files")

Ces fichiers volumineux contiennent les fameux paramètres du modèle. En réalité il s'agit de matrices contenant les différents paramètres sous forme de nombres flottants (`float`). Nous pouvons facilement trouver la précision des nombres flottants d'un modèle en consultant le [fichier de configuration](https://huggingface.co/mistralai/Mistral-7B-Instruct-v0.3/blob/main/config.json) de Hugging Face du modèle. Pour Mistral et Llama 3, l'attribut `torch_dtype` a comme valeur ["bfloat16"](https://pytorch.org/docs/stable/tensors.html).

En réduisant la précision des nombres flottants d'un modèle, nous réduisons aussi sa taille. Mistral-7B-v0.3 nécessite 16GB de VRAM pour une inférence avec 16 bits de précision, la quantité de mémoire nécessaire pour une précision de 8 bits ou 4 bits sera deux ou quatre fois moins élevée.

Sous macOS, avec [Ollama](https://ollama.com/), les modèles sont en général en 4 bits, et s'exécutent dans la mémoire unifiée. Les NVIDIA supportent la quantization, tandis que sur Inferentia ce n'est pas le cas. Il faut donc vérifier les contraintes en fonction de la configuration matérielle. De plus, il existe plusieurs implémentations pour _quantizer_ un modèle, certaines nécessitent une variante du modèle d'origine : 
* [bitsandbytes](https://github.com/TimDettmers/bitsandbytes) : fonctionne avec tous les modèles mais peut être plus lent
* [EETQ](https://github.com/NetEase-FuXi/EETQ ): 8 bits, fonctionne avec tous les modèles
* [AWQ](https://github.com/casper-hansen/AutoAWQ) : 4 bits, nécessite un modèle [spécifique](https://hf.co/models?search=awq)

Par exemple pour une quantization 8 bits sur T4 avec eetq :
```shell
docker run --rm -p 8080:80                                 \
       -v $(pwd)/data:/data                                \
       --gpus all                                          \
       -e HUGGING_FACE_HUB_TOKEN=${HF_TOKEN}               \
       ghcr.io/huggingface/text-generation-inference:2.0.4 \
       --model-id=mistralai/Mistral-7B-Instruct-v0.3       \
       --quantize eetq
```

### Sharding

Sur les instances inferentia2 d'AWS, 2 coeurs (_cores_) sont disponibles sur les inf2.xlarge. On a donc 2x16GB, pour exploiter pleinement les 32GB disponibles il faut scinder le modèle en deux.

![TGI architecture](images/tgi-sharding.png#darkmode "TGI architecture")

TGI supporte nativement le _sharding_ et va donc exploiter les deux coeurs disponibles.

Exemple de commande docker pour mistral sur inferentia2 inf2.xlarge :
```shell
docker run --rm -p 8080:80                            \
       -v $(pwd)/data:/data                           \
       --device=/dev/neuron0                          \
       -e HF_TOKEN=${HF_TOKEN}                        \
       -e HF_BATCH_SIZE=1                             \
       -e HF_SEQUENCE_LENGTH=4096                     \
       -e HF_AUTO_CAST_TYPE="fp16"                    \
       -e HF_NUM_CORES=2                              \
       ghcr.io/huggingface/neuronx-tgi:latest         \
       --model-id=mistralai/Mistral-7B-Instruct-v0.3  \
       --max-batch-size 1                             \
       --max-total-tokens 4096
```

Pour exécuter TGI sur inferentia2, il est nécessaire d'utiliser une image Docker [spécifique](https://github.com/huggingface/optimum-neuron/tree/main/text-generation-inference). D'autre part, le modèle doit être recompilé pour s'exécuter sur inferentia à l'aide d'un outil : [Neuron Compiler](https://awsdocs-neuron.readthedocs-hosted.com/en/latest/compiler/index.html). Hugging Face dispose d'un cache avec les modèles précompilés, par exemple pour Mistral nous trouvons la liste [ici](https://huggingface.co/aws-neuron/optimum-neuron-cache/blob/main/inference-cache-config/mistral.json). Les paramètres `HF_BATCH_SIZE`, `HF_SEQUENCE_LENGTH`, `HF_AUTO_CAST_TYPE` et `HF_NUM_CORES` doivent correspondre au paramètres utilisés lors de la compilation.

![Neuron top](images/neuron-top.png#darkmode "Neuron top")

### Batching

Pour augmenter les performances d'inférence en production, il est primordial d'utiliser la technique de batching.
Lorsqu'un prompt est soumis à un LLM, il est transformé en tokens, la première itération permet de déterminer le premier token de la réponse. La seconde itération prend en entrée le prompt et le premier token généré de la réponse afin de générer le deuxième token.
Le temps d'inférence est principalement lié au temps necessaire pour charger les données dans la mémoire du GPU, il est donc indispensable de faire travailler le GPU sur plusieurs prompts en même temps : les itérations des différents prompts sont _batchées_ pour minimiser le nombre de chargements en mémoire.

TGI supporte nativement le _continuous batching_, pour affiner le comportement des batch il est possible de jouer sur les paramètres suivants :

```shell
docker run --rm -p 8080:80                                 \
       -v $(pwd)/data:/data                                \
       --gpus all                                          \
       --shm-size 1g                                       \
       -e HUGGING_FACE_HUB_TOKEN=${HF_TOKEN}               \
       ghcr.io/huggingface/text-generation-inference:2.0.4 \
       --model-id=mistralai/Mistral-7B-Instruct-v0.3       \
       --quantize eetq                                     \
       --max-input-length 1984                             \
       --max-total-tokens 2048
```

L'option [`--max-total-tokens`](https://huggingface.co/docs/text-generation-inference/basic_tutorials/launcher#maxtotaltokens) est structurante. Celle-ci doit être déterminée en fonction du nombre de tokens en input et du nombre de tokens attendus en output. Plus sa valeur sera basse, plus un batch pourra contenir d'itérations.

A noter, sur les instances inferentia, le batching est statique et déterminé à la compilation contrairement à d'autres configurations matérielles où le nombre d'itérations dans un batch est dynamique.

### Guidance / JSON

Une des difficultés récurrente de l'utilisation d'un LLM est son intégration avec des briques logicielles. Un LLM est conçu pour répondre en langage naturel. Pour obtenir un retour en JSON il peut être fastidueux de décrire un retour précis dans le prompt, qui de toute façon ne serait pas toujours respecté. Pour contrer cela il est possible de préciser un JSON Schema lors de l'appel à TGI.
Le schema est alors inclut dans les batchs et va pondérer les poids sur les tokens de sortie afin de respecter le schema précisé.

Cette fonctionnalité est documentée [ici](https://huggingface.co/docs/text-generation-inference/conceptual/guidance).

```shell
curl 'http://localhost:8080/generate' \
  -H 'Accept: application.json'       \
  -H 'Content-Type: application/json' \
  --data-raw $'{
    "inputs":"< précisez ici votre prompt ... >",
    "parameters": {
        "temperature": 0.5,
        "max_new_tokens": 50,
        "repetition_penalty": 1.03,
        "grammar": {
            "type": "json",
            "value": {
                "properties": {
                    "field1": {
                        "type": "array",
                        "description": "< field 1 description >",
                        "items": {
                            "type": "string",
                            "enum": [
                                "value1",
                                "value2"
                            ]
                        },
                        "minItems": 1,
                        "maxItems": 2,
                        "uniqueItems": true
                    }
                },
                "required": [
                    "field1"
                ]
            }
        }
    }
}' |jq '.generated_text | fromjson'
```

### Embeddings

Pour mettre en place des techniques de RAG (_Retrieval Augmented Generation_) il est possible d'utiliser un autre outil d'Hugging Face : [TEI](https://github.com/huggingface/text-embeddings-inference) (_Text Embeddings Inference_).
AWS supporte un certain nombre de [base de données vectorielles](https://aws.amazon.com/what-is/vector-databases/) qui permettent de stocker les embeddings de vos documents.

Exemple de commande docker pour démarrer TEI sur une instance T4 (architecture turing)
```shell
docker run --rm -p 8081:80                                      \
       -v $(pwd)/data:/data                                     \
       --gpus all                                               \
       --shm-size 1g                                            \
       -e HF_TOKEN=${HF_TOKEN}                                  \
       ghcr.io/huggingface/text-embeddings-inference:turing-1.2 \
       --model-id BAAI/bge-m3                                   \
       --port 80
```

## Conclusion

TGI permet de déployer simplement un LLM Open Source et fait abstraction des spécificités des modèles.
Le support de différentes configurations matérielles apporte plus de souplesse selon les besoins.
Les fonctionnalités de _sharding_, _batching_ et _quantization_ permettent d'optimiser les performances tout en rendant possible l'inférence sur du materiel grand public. Les instances EC2 équipées d'une NVIDIA T4 ne consomment que 70W ce qui est plutôt raisonnable pour déployer un LLM.

La possibilité de guider le modèle avec un JSON Schema est un vrai plus pour l'automatisation de tâches.

TGI répond à nos besoin actuels, vLLM serait intérressant à explorer pour des besoins d'inférence avec des enjeux plus importants en scalabilité.
