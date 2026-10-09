## Formalismes pour le TAL

## Représentation des connaissances

## Projet conjoint

Conception d'un moteur à base de règles pour le chunking automatique de textes

Darya ZDRELYUK, M1 IDL

## Introduction

Ce rapport présente la conception et l'évaluation d'un moteur à base de règles (MBR) pour la segmentation automatique de textes en chunks, selon la définition d'Abney. Le travail s'appuie sur un article du Gorafi comme texte de référence, à partir duquel un système de règles et un lexique grammatical ont été élaborés. Le moteur a ensuite été testé sur deux autres textes : un second article du Gorafi en français et un article en anglais issu de The Onion.

Le texte de référence utilisé pour construire le système est l'article suivant :

« Selon une étude, la file d'attente de l'autre caisse avançait plus vite », Le Gorafi, 22 août 2025.

Source : https://www.legorafi.fr/2025/08/22/selon-une-etude-la-file-dattente-de-lautre-caisse-avancait-plus-vite/

## I. Chunking manuel, règles et lexique

## I.1. Définition du chunk

Un chunk, c’est un mot lexical (un nom, un verbe, un adjectif), et tous les petits mots grammaticaux qui gravitent autour. Le mot lexical est appelé « tête » du chunk. Cette définition suit l'approche d'Abney : le chunk est une unité syntaxique non récursive, plus petite qu'un syntagme complet mais plus grande qu'un simple mot.

On distingue plusieurs catégories de chunks :

- Les chunks nominaux (N) : dont la tête est un nom commun, déclenchés par un déterminant. Les chunks nominaux prépositionnels (PN) sont introduits par une préposition.

- Les chunks verbaux (SV) : dont la tête est un verbe, déclenchés par un pronom personnel sujet. La catégorie V regroupe les verbes modaux ou isolés (comme « sont », « seront »).

- Les chunks adverbiaux (Adv) et adjectivaux (Adj) : détectés par des marques morphologiques (par exemple suffixe -ment pour les adverbes).

- Les chunks de ponctuation (Pct, PctFin) et de guillemets (GO, GF) : chacun contient un seul signe.


- Les chunks de conjonction : conjonctions de subordination (ConjSub) et de coordination (ConjCoor).

## I.2. Formalisme des règles

Les règles de chunking sont de la forme :

MarqueurCatégorie → [ChunkCatégorie.

Chaque règle associe une catégorie de marqueur (trouvée dans le lexique) à un type de chunk à ouvrir. Les règles ont un empan maximum de 2 tokens.

On distingue deux types de règles :

- Les règles simples (1 token) : par exemple Det  [N signifie « si le token courant est un déterminant, ouvrir un chunk nominal ».

- Les règles à empan 2 (avec « + _ ») : par exemple Prep + _  [PN signifie « si le token courant est une préposition, ouvrir un chunk PN et forcer le token suivant dans le chunk ». Le « _ » indique que le token suivant est absorbé quel que soit son type.


Une règle spéciale utilise l'état du chunk précédent : chunk_précédent(N) + Det + _ → [Adj. Cette règle permet de traiter le cas du superlatif (« la plus lente ») en ouvrant un chunk adjectival au lieu d'un chunk nominal quand un déterminant apparaît juste après un chunk N.

## I.3. Formalisme du lexique

Le lexique est une ressource minimaliste qui ne contient que des mots grammaticaux. Aucun nom, verbe conjugué, adjectif ou adverbe n'y figure, ces catégories sont marquées « Ensemble infini ».

Les catégories du lexique sont :

- Prep (prépositions),

- Det (déterminants),

- pPersSuj (pronoms personnels sujets),

- pDemSuj (pronoms démonstratifs sujets),

- pRel (pronoms relatifs),

- pIndSuj (pronoms indéfinis sujets),

- ConjSub (conjonctions de subordination),

- ConjCoor (conjonctions de coordination),

- Pct (ponctuation),

- PctFin (ponctuation finale),

- Mod (modaux/auxiliaires),

- GO (guillemet ouvrant),

- GF (guillemet fermant).


Le lexique inclut des locutions à deux mots comme « parce que », « alors que », traitées comme des entrées uniques.

## I.4. Résultat attendu

Après chunking manuel du texte de référence, on obtient 194 chunks répartis en 12 catégories. Ce texte chunké manuellement sert de référence pour évaluer les résultats du moteur automatique.

## II. Présentation du moteur à base de règles

Le moteur est implémenté en Python sous forme d'un notebook Jupyter. Il suit plusieurs étapes : chargement du lexique et des règles depuis des fichiers Excel, tokenisation du texte, chunking par application des règles, et génération de la sortie en XML et HTML.

## II.1. Chargement des ressources

La fonction read_lexique lit le fichier Excel du lexique. Chaque ligne contient une entrée de la forme « Prep = {sur, à, en, de, ...} ». La fonction extrait la catégorie et les mots, et construit un dictionnaire Python {mot: catégorie}. La virgule, utilisée comme séparateur dans le fichier, est traitée spécialement pour être aussi reconnue comme token de ponctuation.

La fonction read_rules lit les règles et les transforme en un dictionnaire {catégorie_déclencheur: (type_chunk, force)}. Le booléen « force » indique si la règle contient « + _ », c'est-à-dire si le token suivant doit être absorbé dans le chunk.

## II.2. Tokenisation

La tokenisation utilise une expression régulière adaptée au français :

```
re.findall(r"\bqu'|\b[cdlnmtsj]'|\w+(?:-\w+)*|[^\w\s]", texte, re.IGNORECASE)
```

Cette regex traite en priorité les élisions (qu', c', l', d', n', s', j', m', t') comme des tokens uniques, puis les mots composés avec tiret (peut-on, attaché-case), puis les mots simples, et enfin chaque signe de ponctuation isolément. Les apostrophes typographiques sont normalisées avant la tokenisation.


## II.3. Recherche de catégorie

La fonction chercher_categorie tente d'associer une catégorie à chaque token, dans l'ordre suivant :

1) Guillemets typographiques (test Unicode direct pour distinguer GO et GF).

2) Locutions à 2 tokens : on teste si le token courant + le token suivant forment une locution du lexique (« parce que », « alors que »).

3) Token seul dans le lexique.

4) Morphologie par regex : les mots finissant par -ment sont détectés comme adverbes ; les formes verbe-pronom inversé (peut-on, explique-t-il) sont détectées comme pronoms personnels sujets.

## II.4. Algorithme de chunking

Le chunker parcourt les tokens un par un. Pour chaque token, il cherche sa catégorie et la règle correspondante. Si une règle s'applique, le chunk en cours est fermé et sauvegardé, et un nouveau chunk est ouvert. Si aucune règle ne s'applique, le token est ajouté au chunk en cours.

Trois mécanismes supplémentaires sont implémentés :

- Absorption forcée (« + _ ») : quand une règle à empan 2 s'applique, le token suivant est automatiquement ajouté au chunk, même s'il a une catégorie dans le lexique.

- Fermeture des chunks de ponctuation : quand un token sans catégorie arrive après un chunk Pct, PctFin, GO ou GF, le chunk de ponctuation est fermé et le token commence un nouveau segment.

- Fusion Det+Det : si un déterminant arrive alors qu'un chunk N est déjà ouvert, il est absorbé au lieu d'ouvrir un nouveau chunk. Cela évite de séparer des exemples comme « tous les produits » en deux chunks.

Un post-traitement fusionne les segments sans catégorie ([None]) avec le chunk suivant, pour garantir que chaque chunk a une étiquette.

## II.5. Sortie XML

La sortie est générée au format XML avec la bibliothèque ElementTree de Python. Chaque chunk est représenté par un élément <chunk> avec un attribut « cat » indiquant sa catégorie. Une version HTML est aussi générée pour la visualisation.

Le choix du XML comme format de sortie n'est pas anodin. Le XML (eXtensible Markup Language) est un format de sérialisation qui permet de structurer des données de manière hiérarchique et lisible. Dans notre cas, il sert à représenter les connaissances linguistiques produites par le chunker.

La structure XML utilisée est la suivante :


```
v<texte_chunke> Conjsub Alors que]
<chunk cat="ConjSub">Alors que</chunk>
tous sv
parvenus]
&tes
vous
<chunk V">vous parvenus</chunk>
prendre]
a
produits
<chunk "PN">3 prendre</chunk>
inscrits)
Tes
<chunk "N">tous les produits
List
votre
PN
sur
<chunk votre liste</chunk: PN de courses]
<chunk PN en seulement 23 minutes]
<chunk
<chunk </chunk> sv il ne]
<chunk Sv vous reste plus]
<chunk "SV">vous reste qu]
<chunk at="ConjSub">qu’</chunk> a payer]
<chunk cat="PN">3 payer</chunk>
<chunk cat="Pct">,</chunk> et]
<chunk
<chunk est 1a</chunk>
<chunk cat="ConjSub" >que</chunk>
<chunk risquez</chunk>
<chunk cat: perdre</chunk>
<chunk avance] N votre
<chunk Pet,
<chunk cat="ConjSub">comme< /chunk> Conjsub_conme]
```

Chaque chunk est un élément XML avec un attribut « cat » qui encode la catégorie. Le contenu textuel de l'élément représente les tokens du chunk. Cette structure permet de séparer clairement le contenu (les mots) des métadonnées (la catégorie), ce qui facilite le traitement automatique en aval : un autre programme peut lire ce fichier XML et exploiter les catégories sans avoir à refaire l'analyse.

Le XML est aussi un format interopérable et standardisé, ce qui permet d'échanger les résultats du chunking entre différents outils de TAL.

## II.6. Modélisation du processus

L'algorithme du chunker peut être représenté comme un processus séquentiel en quatre étapes :

- 1. Chargement : lecture du lexique et des règles depuis les fichiers Excel, construction des structures de données internes (dictionnaires Python).

- 2. Tokenisation : découpage du texte brut en une liste ordonnée de tokens grâce à l'expression régulière.

- 3. Chunking : parcours séquentiel des tokens, recherche de catégorie dans le lexique, application des règles, ouverture et fermeture des chunks. Ce processus est linéaire (complexité en O(n), n étant le nombre de tokens) : chaque token est traité une seule fois, et pour chaque token, on parcourt l'ensemble des règles.

- 4. Sérialisation : transformation de la liste de chunks en fichier XML.

Ce modèle de traitement correspond à un système robuste au sens du cours : il produit toujours exactement une solution, même si cette solution est partiellement incorrecte. À l'opposé, un système combinatoire explorerait toutes les segmentations possibles, ce qui augmenterait la complexité mais pourrait donner de meilleurs résultats.

## III. Expérimentation sur le texte initial

Le texte initial est l'article du Gorafi sur les files d'attente de supermarché.

Source : https://www.legorafi.fr/2025/08/22/selon-une-etude-la-file-dattente-de-lautre-caisse-avancait-plus-vite/


## III.1. Résultats quantitatifs

|   | Frontières | Catégories |
| --- | --- | --- |
| VP | 143 | 139 |
| FP | 39 | 43 |
| FN | 51 | 55 |
| Précision | 0.7857 | 0.7637 |
| Rappel | 0.7371 | 0.7165 |
| F-mesure | 0.7606 | 0.7394 |

Le chunker produit 182 chunks contre 194 attendus. La F-mesure de 0.76 pour les frontières montre que le système identifie correctement la majorité des chunks.

## III.2. Analyse qualitative

Le chunker remplit son rôle principal : il segmente le texte en chunks étiquetés, ouvre et ferme les chunks aux bons endroits pour la majorité des cas. Les catégories Pct (26/26), GO (4/4), GF (4/4), ConjCoor (10/10) sont parfaitement détectées.

Cependant, on observe plusieurs types d'erreurs :

- Chunks trop longs : les mots pleins (verbes, noms, adjectifs) qui n'ont pas de catégorie dans le lexique sont absorbés dans le chunk en cours. Par exemple, « l'univers a décidé » est étiqueté [N] alors que « a décidé » est un verbe.

- Ambiguïté des pronoms : « vous » et « nous » sont toujours traités comme pronoms sujets (pPersSuj), même quand ils sont objets (« pour vous contrarier »).

- Absence de chunks Adj : la catégorie Adj (7 attendus) n'apparaît jamais dans la sortie automatique, car les adjectifs ne sont pas dans le lexique et la regex morphologique a été limitée aux adverbes en -ment pour éviter les faux positifs.

## III.3. Analyse des usages des règles

Les règles les plus utilisées sont Prep + _ → [PN (44 fois) et pPersSuj → [SV (27 fois). Les règles Det → [N et ConjSub → [ConjSub suivent. Les règles Adj → [Adj et Adv → [Adv ne se déclenchent que via la regex morphologique (-ment), pas via le lexique.

## III.4. Pistes d'amélioration

- Ajouter des regex morphologiques pour les verbes à l'infinitif (-er, -ir, -re) et les participes (-é, -ant). Cependant, ces regex produisent des faux positifs (« métier » n'est pas un infinitif).

- Enrichir le lexique avec des formes manquantes.


## IV. Expérimentation sur un second texte

Le second texte est un autre article du Gorafi, en français :

« 83% des Français avouent dormir au bureau pour récupérer de leur week-end en famille », Le Gorafi, 4 mars 2026.

Source : https://www.legorafi.fr/2026/03/04/83-des-francais-avouent-dormir-au-bureau-pour-recuperer-de-leur-week-end-en- famille/

## IV.1. Avec le lexique initial

|   | Frontières | Catégories |
| --- | --- | --- |
| VP | 41 | 41 |
| FP | 31 | 31 |
| FN | 76 | 76 |
| Précision | 0.5694 | 0.5694 |
| Rappel | 0.3504 | 0.3504 |
| F-mesure | 0.4339 | 0.4339 |

La dégradation est nette : la F-mesure passe de 0.76 à 0.43. Le chunker ne produit que 72 chunks contre 117 attendus. La cause principale est le manque de mots dans le lexique : « je », « j' », « mes », « sa », « son », « des », « quand », « puisque », « chez », « aux » sont absents car ils n'apparaissaient pas dans le texte 1. Les chunks résultants sont beaucoup trop longs (par exemple, « Je sais que je vais enfin avoir droit » est un seul chunk ConjSub).

## IV.2. Avec le lexique enrichi

Après enrichissement du lexique (ajout de je, j', mes, sa, son, des, quand, puisque, chez, aux, par, pendant, avec, puis, où, auquel, lesquels, ainsi qu'), les résultats s'améliorent :

|   | Frontières | Catégories |
| --- | --- | --- |
| VP | 79 | 79 |
| FP | 19 | 19 |
| FN | 38 | 38 |
| Précision | 0.8061 | 0.8061 |
| Rappel | 0.6752 | 0.6752 |
| F-mesure | 0.7349 | 0.7349 |


La F-mesure remonte à 0.73, proche du texte 1 (0.76). Cela confirme que les erreurs du texte 2.1 étaient principalement dues à un lexique insuffisant, et non à un problème de règles. Le moteur est robuste : les mêmes règles fonctionnent sur les deux textes, seul le lexique doit être enrichi.

## IV.3. Erreurs persistantes

Certaines erreurs restent même après enrichissement : les verbes conjugués (« témoigne », « a décidé », « pouvait ») ne sont pas détectés car ils ne sont pas dans le lexique. Les chunks NP (noms propres comme « Lilian ») ne sont pas reconnus car le système n'a pas de règle pour détecter les majuscules.

## IV.4. Analyse des usages des règles

Sur le texte 2 avec le lexique initial, on observe que seules 8 catégories de chunks sont produites (contre 12 attendues). Les règles pPersSuj → [SV et Det → [N ne se déclenchent presque pas car « je », « j' », « mes », « sa », « son », « des » sont absents du lexique. La règle Prep + _ → [PN reste la plus utilisée (27 occurrences), ce qui crée des chunks PN trop longs qui absorbent tout ce qui suit la préposition.

Avec le lexique enrichi, la distribution des catégories se rapproche de la référence : PN passe de 27 à 32 (référence : 32), SV de 8 à 18 (référence : 19), N de 10 à 17 (référence : 17). Les catégories Adj (3 attendues), V (9 attendues) et NP (2 attendues) restent absentes de la sortie automatique.

## IV.5. Pistes d'amélioration

Pour améliorer les résultats sur le texte 2, il faudrait enrichir le lexique de manière plus systématique en y ajoutant l'ensemble des pronoms personnels sujets français (je, tu, il, elle, nous, vous, ils, elles) et l'ensemble des déterminants possessifs (mon, ton, son, ma, ta, sa, mes, tes, ses, notre, votre, leur, nos, vos, leurs). L'ajout de « où » et « auquel » dans pRel permettrait aussi de mieux détecter les propositions relatives. Enfin, une règle basée sur les majuscules en début de mot (hors début de phrase) permettrait de détecter les noms propres comme « Lilian ».

## V. Expérimentation dans une autre langue

Le troisième texte est un article en anglais issu de The Onion :

« Overambitious Man Wants To Get 2 Things Done Today », The Onion.

Source : https://theonion.com/overambitious-man-wants-to-get-2-things-done-today/

Pour cette expérimentation, les mêmes règles ont été conservées et le lexique a été traduit en anglais. La fonction deviner_categorie a été adaptée : le suffixe adverbial -ment (français) a été remplacé par - ly (anglais), et la détection des formes inversées (peut-on, explique-t-il) a été désactivée car l'anglais n'utilise pas ce mécanisme.

## V.1. Avec le lexique initial

|   | Frontières | Catégories |
| --- | --- | --- |
| VP | 46 | 43 |


| FP | 37 | 40 |
| --- | --- | --- |
| FN | 86 | 89 |
| Précision | 0.5542 | 0.5181 |
| Rappel | 0.3485 | 0.3258 |
| F-mesure | 0.4279 | 0.4000 |

## V.2. Avec le lexique enrichi

|   | Frontières | Catégories |
| --- | --- | --- |
| VP | 60 | 58 |
| FP | 32 | 34 |
| FN | 72 | 74 |
| Précision | 0.6522 | 0.6304 |
| Rappel | 0.4545 | 0.4394 |
| F-mesure | 0.5357 | 0.5179 |

L'enrichissement du lexique améliore la F-mesure de 0.43 à 0.54. Les résultats sont inférieurs au français, ce qui s'explique par plusieurs différences structurelles :

- Les phrasal verbs : en anglais, des verbes comme « pick up », « slow down », « set up » sont séparés par la préposition, ce qui crée des chunks incorrects.

- Les contractions : « you're », « you've », « there's » sont mal tokenisées car le tokeniseur est conçu pour les élisions françaises.

- L'ambiguïté de « that » : il peut être conjonction de subordination ou déterminant selon le contexte.

- Les noms propres : « James Chao », « Aaron Steiner » ne sont pas détectés.

Malgré ces limitations, le fait que les mêmes règles produisent des résultats exploitables en anglais montre que la structure du chunker est transférable entre les langues. L'enjeu est d'adapter le lexique et les règles morphologiques, pas le moteur lui-même.

## V.3. Analyse des usages des règles

En anglais, la règle Prep + _ → [PN est la plus utilisée (20 occurrences), suivie par pPersSuj → [SV (12 occurrences). La règle Mod → [V se déclenche pour « was » et « were » (6 occurrences), ce qui est spécifique à l'anglais où les auxiliaires sont dans le lexique.


On note que la règle ConjSub → [ConjSub produit des résultats différents du français. En français, « que » est presque toujours une conjonction de subordination, mais en anglais, « that » peut être déterminant (« That guy ») ou conjonction (« adding that »). Le système ne peut pas distinguer les deux usages sans analyse syntaxique plus profonde.

## V.4. Pistes d'amélioration

Plusieurs adaptations spécifiques à l'anglais amélioreraient les résultats. Il faudrait ajouter une regex pour les verbes au gérondif (-ing) et au participe passé régulier (-ed). Les phrasal verbs (pick up, slow down) pourraient être traités comme des locutions dans le lexique. Le tokeniseur devrait être adapté pour gérer les contractions anglaises (you're → you + 're, don't → do + n't). Enfin, le mot « that » devrait être traité par une règle contextuelle, car il est un source d'ambiguïté.

## VI. Synthèse et conclusion

Ce travail a permis de concevoir un moteur de chunking robuste et fonctionnel. Les principaux résultats sont résumés dans le tableau suivant :

| Texte | F-mesure frontières | F-mesure catégories |
| --- | --- | --- |
| Texte 1 (FR, lexique initial) | 0.7606 | 0.7394 |
| Texte 2 (FR, même lexique) | 0.4339 | 0.4339 |
| Texte 2 (FR, lexique enrichi) | 0.7349 | 0.7349 |
| Texte 3 (EN, lexique initial) | 0.4279 | 0.4000 |
| Texte 3 (EN, lexique enrichi) | 0.5357 | 0.5179 |

Le moteur remplit son rôle : il segmente le texte en chunks étiquetés, est robuste (aucun mot n'est laissé hors d'un chunk), et produit toujours une sortie. Les erreurs proviennent principalement de deux sources

:

- Le lexique minimaliste : les mots pleins ne sont pas référencés, ce qui empêche la détection des frontières de chunks à l'intérieur de séquences longues.

- Les ambiguïtés catégorielles : certains mots grammaticaux appartiennent à plusieurs catégories (« de » est à la fois préposition et déterminant partitif, « que » est à la fois conjonction et pronom relatif).

Les pistes d'amélioration incluent l'enrichissement du lexique, l'ajout de regex morphologiques plus fines (avec gestion des exceptions), et l'utilisation de règles à état pour traiter des cas comme le superlatif. Le moteur pourrait aussi bénéficier d'une règle de détection des noms propres basée sur les majuscules.


En conclusion, ce chunker illustre bien le compromis entre robustesse et précision dans un système de TAL minimaliste. Le moteur lui-même est adaptable à tout type de texte et à toute langue — l'enjeu est dans la qualité des règles et du lexique.
