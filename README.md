# Problème : Génération de texte

## Objectif principal

Ce problème consiste à réaliser un programme d'aide à la rédaction de texte, à
la manière du système d'aide à la saisie présent sur les téléphones portables. 

On envisage trois principales fonctionnalités :
1.  Proposer, étant donnés les mots déjà tapés par l'utilisateur, une liste de
    mots suivants possibles.
1.  Générer un texte aléatoire de manière autonome, à la manière d'un agent
    conversationnel comme ChatGPT (mais sans le *deep learning*...), à partir
    d'un corpus de textes de référence. Par exemple :
    ```bash
    $ python textgen.py --auto
    ```
    pourrait engendrer le texte :
    ```
    un voyageur sur quatre opte pour des conflits externes afin d'en détourner l'attention
    ```
1.  Proposer une ou plusieurs complétions ou corrections du mot courant à l'aide
    d'un lexique (liste de mots de la langue utilisée).

On attend de ce programme qu'il permette d'ajuster plusieurs paramètres
concernant la génération (mode autonome ou non, auto-correction, nombre de
propositions de mots, fichiers d'entraînement...).

## Récupération et préparation d'un corpus de texte

Pour proposer des mots vraisemblables ou générer un texte automatiquement, il
est nécessaire de s'appuyer sur une grande collection de textes afin de
constituer un corpus. Plusieurs corpus sont possibles selon les objectifs ou le
domaine visé. Pour commencer, on pourra par exemple récupérer dans un premier
temps un corpus de textes en français à partir de la [collection de corpus de
l'Université de Leipzig](https://corpora.uni-leipzig.de/fr).

Comme dans toute procédure de traitement de texte, il est conseillé de préparer
à l'avance les textes recueillis afin de les normaliser. Il peut être
intéressant de réfléchir au traitement de plusieurs éléments, tels que les
majuscules, la ponctuation, les sauts de ligne,en fonction du résultat escompté.

Il pourra être utile de consulter soigneusement la documentation sur le type
[`str`](https://docs.python.org/3/library/stdtypes.html#textseq) et sur le
module [`string`](https://docs.python.org/3/library/string.html).

## Génération de texte via chaines de Markov

Pour générer du texte (ou compléter un texte déjà saisi), on s'inspire de
méthodes utilisant des chaines de Markov. Le principe de cette approche est de
fabriquer aléatoirement du texte en imitant le style d'un corpus prédéfini.

Le principe est de choisir les mots successifs un par un, « de proche en proche
», c'est à dire que l'on utilise les quelques derniers mots saisis pour
déterminer le prochain mot à écrire, sans prendre en compte le début du texte.

### Version avec contexte à un mot

Dans une version simplifiée, on peut se contenter de ne prendre en considération
que le dernier mot du texte pour générer le mot suivant. Pour cela, étant donné
le corpus choisi, on commence par **compter toutes les successions de mots**
apparaissant dans le corpus. On organise ensuite ces données sous la forme d'une
table, dans laquelle chaque ligne correspond à un mot du corpus et chaque
colonne correspond à un possible mot suivant. Le coefficient de la ligne $u$ et
de la colonne $v$ indique combien de fois le mot $v$ est apparu immédiatement
après le mot $u$ dans le corpus. 

On interprète alors chaque ligne de ce tableau comme une distribution de
probabilité : plus un mot apparaît fréquemment à la suite du mot $u$ dans le
corpus, plus il est probable que le programme suggérera ou choisira ce mot comme
successeur du mot $u$ dans le texte saisi.

Le premier mot du texte est soit choisi au hasard parmi les premiers mots
possibles du corpus, soit saisi par l'utilisateur.

**Exemple** : Supposons que notre corpus soit composé des deux textes suivants :

```text
Je mange une pomme jaune.
```

et

```text
Le ver mange la pomme jaune.
```

Les successions de mots peuvent alors être représentées sous la forme du tableau
suivant :

|           | je | mange | une | pomme | jaune | le | ver | la |
|-----------|----|-------|-----|-------|-------|----|-----|----|
| **je**    | 0  | 1     | 0   | 0     | 0     | 0  | 0   | 0  |
| **mange** | 0  | 0     | 1   | 0     | 0     | 0  | 0   | 1  |
| **une**   | 0  | 0     | 0   | 1     | 0     | 0  | 0   | 0  |
| **pomme** | 0  | 0     | 0   | 0     | 2     | 0  | 0   | 0  |
| **jaune** | 0  | 0     | 0   | 0     | 0     | 0  | 0   | 0  |
| **le**    | 0  | 0     | 0   | 0     | 0     | 0  | 1   | 0  |
| **ver**   | 0  | 1     | 0   | 0     | 0     | 0  | 0   | 0  |
| **la**    | 0  | 0     | 0   | 1     | 0     | 0  | 0   | 0  |

Supposons que le premier mot saisi (ou généré aléatoirement) est `je`.

On voit dans la table que le mot `je` est suivi une fois du mot `mange` dans le
corpus, et jamais d'aucun autre mot. Par conséquent le programme propose avec
probabilité 1 le mot `mange` comme mot suivant. On obtient donc avec probabilité
1 le texte:

```text
je mange
```

On continue ensuite ce procédé de mot en mot, en prenant en compte le dernier
mot généré. Dans le corpus le mot `mange` est suivi une fois du mot `une` et une
fois du mot `la`, et d'aucun autre. Ces deux mots seront donc choisis avec
probabilité $1/2$ (et aucun autre ne le sera). On génère donc avec une chance
sur deux la chaine

```text
je mange une
```

et avec une chance sur deux la chaine:

```text
je mange la
```

### Version avec contexte ajustable

Pour augmenter la qualité du modèle, on peut prendre en considération non plus
le seul dernier mot généré, mais les `N` derniers mots du texte, où `N` est un
paramètre ajustable par l'utilisateur.

Par exemple, si l'on fixe `N = 3`, on doit prendre en compte les trois derniers
mots écrits pour déterminer les probabilités d'apparition du prochain mot. On
adaptera donc la méthode de comptage précédente afin qu'elle compte non plus les
mots pouvant succéder à chaque mot isolé, mais les mots pouvant succéder à
chaque trigramme (suite de trois mots).

### Remarque sur l'aide à la saisie

Pour permettre à l'utilisateur de guider la génération de texte, la solution la
plus simple est d'afficher une liste numérotée de tous les candidats et de
demander à l'utilisateur de saisir le numéro souhaité. On peut trier cette liste
par probabilité décroissante (le mot le plus probable sera donc affiché en
premier).

Il est également possible de laisser l'utilisateur taper le prochain mot
lui-même, ou choisir d'arrêter la génération de texte. Le texte final est alors
affiché sur le terminal.

## Travail demandé

Nous présentons ici la partie obligatoire du travail à réaliser, qu'il est
indispensable de terminer entièrement pour obtenir une bonne appréciation. Une
liste d'améliorations optionnelles est fournie plus loin dans le sujet.

La partie obligatoire du travail est d'écrire un programme générant du texte à
l'aide de la méthode décrite ci-dessus. On s'assurera également d'implémenter
les paramètres suivants (spécifiables par l'utilisateur en ligne de commande) à
l'aide du module `argparse` :

* `--taille <entier>` : Réglage de la taille des contextes utilisés (par défaut
  : 3).
* `--debut <chaine>` : Portion initiale de texte utilisée pour la génération
  (par défaut : chaîne vide).
* `--nbchoix <entier>` : Nombre de choix de mots proposés à l'utilisateur
  (valeur par défaut : 3).
* `--saisie <booléen>` : Possibilité pour l'utilisateur de rentrer lui-même un
  nouveau mot plutôt que de choisir dans la liste (valeur par défaut : `true`).
* `--auto` : Synonyme de `--nbchoix 1` et `--saisie false`, correspond à un mode
  automatique où le texte est entièrement généré par le programme.

## Conseils

* L'utilisation de la structure de données `dict` est recommandée.
* Ne négligez pas la phase de pré-traitement des textes, elle aura un impact
  direct sur la qualité de vos résultats.
* Commencez par gérer du texte mis en minuscule, sans aucune ponctuation, puis
  ajoutez de la complexité au fur et à mesure si souhaité.
* Si vous la maitrisez ou souhaitez la découvrir, la bibliothèque
  [re](https://docs.python.org/3/library/re.html#module-re) permettant
  d'utiliser des *expressions régulières* peut être utile pour la phase de
  traitement du corpus.
* Si vous traitez de très gros corpus qui mettent du temps à charger, le module
  `pickle` pourra vous aider à stocker les résultats de vos calculs.
* N'hésitez pas à varier les tailles de corpus et de contexte, pour voir les
  paramétrisations donnant les résultats les plus convaincants.

## Améliorations possibles

Si vous avez entièrement terminé la partie obligatoire, vous êtes libre de
réaliser toute amélioration qui vous intéresse, ou même de simplement vous
documenter et rassembler des informations pertinentes sur le sujet (que vous
pourrez présenter sous la forme d'un bref rapport, par exemple dans un fichier
`rapport.md`). Voici quelques idées d'approfondissements possibles.

### Mise en forme du texte

Plutôt que de générer un texte non ponctué et sans majuscule, il est
envisageable de prendre en charge la mise en majuscule, la ponctuation, la
découpe en phrases, ou même la mise en forme du texte (gras, italique, découpe
en paragraphes, etc). 

Si la ponctuation a été gérée, il est nécessaire de s'assurer que l'ordre des
guillemets et parenthèses soit correctement respecté.

### Gestion améliorée du corpus

On peut envisager de gérer plusieurs corpus de texte différents et de permettre
le choix du corpus à partir duquel la génération doit être faite.

Ceci peut être complété par une récupération de corpus depuis d'autres sources.
On pourra, par exemple, utiliser le module python ```wikipedia``` pour
télécharger et récupérer le contenu de plusieurs articles Wikipédia, et de les
stocker localement. On pourra également penser à des œuvres littéraires en accès
libre, des posts de réseaux sociaux, ou n'importe quel corpus de texte
accessible en ligne. Par ailleurs, on pourra accéder à des corpus en anglais.

Enfin, il est possible d'implémenter une fonctionnalité de récupération des
corpus "à la volée", au moment de l'exécution du programme.

### Interface graphique

Il est envisageable de créer une interface graphique `fltk` permettant de
charger les différents corpus, de gérer les tailles des contextes et la longueur
du texte produit, ainsi que de sélectionner le prochain mot dans une liste de
candidats (en mode interactif seulement).

### Aide à la saisie pour chaque mot

Cette amélioration consiste à implémenter une aide à la saisie *en cours de
frappe* d'un mot. Après chaque lettre, le programme peut afficher une ou
plusieurs propositions de complétions ou de corrections du mot, en s'appuyant
sur la liste de tous les mots du corpus ou sur un lexique préalablement choisi.

Un appui sur une touche spéciale (`tab` par exemple) provoque le remplacement du
mot en cours de saisie par l'une des suggestions affichées.

Plusieurs algorithmes pour la complétion ou la correction de mots, prenant en
compte par exemple la suppression, l'ajout ou le remplacement de lettres,
existent. Les choix effectués devront absolument être expliqués.

De manière semblable, un simulateur de saisie sur pavé numérique (de type T9)
peut être proposé.

Pour cette amélioration, il est conseillé de réaliser une interface `fltk`
permettant de récupérer les événements clavier plus facilement.

### Amélioration de l'algorithme de suggestion de mots

On pourra proposer toute idée, heuristique ou algorithme alternatif pour
améliorer la plausibilité des mots suggérés ou des textes engendrés en mode
automatique. Il est par exemple possible de prendre en compte les habitudes
personnelles de l'utilisateur (en rendant plus probables les mots déjà choisis
ou saisis par le passé, y compris lors de précédentes exécutions du programme),
de prendre en compte d'une manière ou d'une autre une plus grande portion du
texte, etc.

*Remarque :* il existe des cours en ligne pour apprendre à coder son propre LLM
(par exemplei [ici](https://karpathy.ai/zero-to-hero.html) ou
[là](https://www.freecodecamp.org/news/how-to-build-a-large-language-model-from-scratch-using-python/))
mais c'est nettement au-delà de ce que nous envisageons pour ce travail.

### Idées diverses

Voici quelques autres pistes d'améliorations mineures ou plus difficiles :

* Permettre de sauvegarder et de charger les textes générés.
* Imposer une restriction sur la longueur des phrases générées.
* Ajouter des règles de syntaxe et de grammaire dans la génération du texte, par
  exemple pour l'accord des verbes.

Comme toujours, cette liste n'est évidemment pas exhaustive, et d'autres
propositions d'améliorations sont bienvenues.

