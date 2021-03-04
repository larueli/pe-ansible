# TP Ansible

Ce TP sur Ansible est destiné à des étudiants s'intéressant aux notions suivantes : cloud, automatisation, serveurs, provisionning, déploiement.

Il a été conçu et rédigé par [Ivann LARUELLE](https://ivannlaruelle.fr), étudiant en RT à l'UTT en P21, administrateur du [Système d'Information des Associations de l'UNG](https://ung.utt.fr/tech/sia) et encadré par l'enseignante [Samiha AYED](https://recherche.utt.fr/research-directory/samiha-ayed). 

## Introduction

### Vous avez dit Ansible ?

Quand une tâche devient répétitive, il convient d'automatiser son exécution. C'est le principe même de l'informatique. Cela s'étend à la notion de fonctions en programmation, qui sont des fragments de code réutilisables, très pratique quand on cherche à développer une application qui répond à un besoin métier (gérer les étudiants, les examens, ...).

En administration système, où on a des besoins de gestion système (administrer des machines pour assurer le bon fonctionnement des applications qui s'y exécutent, ou administrer des postes utilisateurs, qu'ils soient fixes ou portable afin que la configuration des postes réponde à la politique IT de l'entreprise en terme de fonctionnalité, de sécurité, ...), la même problématique se pose que lors du développement d'une application : certaines tâches sont répétitives. Imaginez devoir gérer 1 200 postes utilisateurs dans une même société, ou devoir ajouter des serveurs régulièrement et maintenir identique la configuration de sécurité, ou installer et configurer certaines choses en commun sur ces serveurs ! Ansible sert à ça.

Dans ce TP, nous allons voir comment certaines fonctionnalités majeures d'Ansible peuvent nous servir afin de déployer un serveur MediaWiki. MediaWiki est un service web qui sert de base à Wikipédia. Ce service est open-source, donc n'importe qui (à condition d'avoir le matériel pour) peut installer MediaWiki sur un serveur. MediaWiki nécessite une base de données afin de stocker ses données (MariaDB dans notre exemple), un serveur web afin d'assurer son accès depuis les autres machines en utilisant le procole HTTP (Apache dans notre exemple) et un outil permettant d'interpréter son code source afin de savoir comment traiter les requêtes (les développeurs de MediaWiki utilisent PHP, nous allons donc devoir installer PHP).

### Oui, Ansible !

Ansible est un outil open source édité par Red Hat, une multinationale célèbre pour son système d'exploitation orientée serveurs (CentOS), sa distribution Kubernetes (OpenShift), et bien d'autres outils (quay.io, ...).

Dans l'idée, Ansible permet d'effectuer des tâches sur des machines à distance, ces tâches ayant la particularité d'être idempotentes.

Une fonction f est idempotente si f rond f est égal à f. Autrement dit, si f est idempotente, alors f(f(f(f(x)))) = f(x). La fonction valeur absolue est idempotente. Ou encore autrement dit si, que l'on exécute la fonction 1 fois ou 2404 fois d'affilée, le résultat sera identique. La plupart des scripts que l'on développe soi-même n'ont pas cette propriété, et risquent d'avoir des effets indésirables si exécutés plusieurs fois. Les tâches de base fournies par Ansible et la plupart de celles développées par la communauté ont cette particularité. L'intêrét de cette propriété est énorme : vous n'avez plus à "suivre" l'application de la configuration sur tous les postes (est-ce que telle machine est bien configurée ? est-ce que cette machine est configurée avec la dernière configuration sortie ?), vous avez juste à lancer votre script sur tous les postes sans vous poser de question, pour ceux déjà configurés cela ne changera rien, pour ceux qu'il faut mettre à jour, cela sera fait : un gain de temps et de fiabilité.

Ansible est l'outil qui fournit ces fonctions idempotentes, et il permet de les exécuter facilement sur plusieurs postes en même temps. Au delà de ça, on le verra plus tard, il permet quand même d'avoir une configuration très fine (par groupe de machines, par machine directement) et centralisée.

### Le YAML

Avant toute chose, il faut comprendre ce qu'est le YAML (ou YML). Le YAML est une manière de décrire et stocker des données, comme le JSON, les fichiers INI ou encore le XML par exemple. Le YAML est notamment utilisé par Ansible afin de décrire à la fois les variables mais aussi les tâches à effectuer.

C'est en réalité assez simple, car tout est sous la forme "clef valeur", c'est à dire qu'une variable est décrite ainsi : `ma_variable: mavaleur`. Un fichier YAML commence toujours par `---` car cela permet notamment d'aligner plusieurs fichiers de données YAML dans le même flux réseau (donc séparés par `---`), ce qui n'est pas très utilisé il est vrai par Ansible, mais cela reste une convention à respecter. Les noms de variables ne peuvent contenir que des caractères alphanumériques sans accent ainsi que `_` mais ne doivent pas commencer par un chiffre.

Tout ce qui est situé à droite d'un `#` est considéré comme un commentaire.

#### Les principaux types de données en YAML

* Les données simples :
  * Les chaines de caractères : `ma_variable: "ma_valeur"`. Techniquement, sauf cas particulier, on est pas obligé d'utiliser les guillemets, mais en réalité cela permet de gagner du temps car la syntaxe avec les guillemets est toujours valide (notamment si on fait référence à une autre variable). 
  * Les nombres : `ma_variable: 5`. Une valeur en octet doit être précédée de `0`, en hexadécimal précédée par `0x`.
  * Les booléens : `ma_variable: true|false|yes|no`.
* Les listes/tableaux :
  * Ecriture avec crochets : `ma_variable: ["val1", "val2"]`
  * Ecriture sous forme de liste (recommandé car plus lisible) :

```yaml
ma_variable:
  - "val1"
  - "val2    
```

L'indentation est de 2 espaces.

* Les dictionnaires :

```yaml
utilisateur:
  prenom: "Ivann"
  nom: "LARUELLE"
```

#### Utilisation des variables

Ansible utilise la syntaxe Jinja2 pour exploiter les variables. Pour faire référence à une variable, on utile `{{  }}`, exemple : `{{ ma_variable }}`. On peut imbriquer des variables entre elles : par exemple : `chemin_clef_secrete: /etc/ssl/{{ dossier_secret }}/clef.private`. **Attention**, si votre variable imbriquée commence par une autre variable, toute la chaine de caractères doit être encadrée par des guillemets. Exemple :

```yaml
# Exemple avec la définition d'un chemin d'accès au fichier test.txt dont le début est une variable
ma_variable_fausse: {{ ma_variable_precedente }}/test.txt # FAUX et lévera une erreur à l'exécution !
ma_variable_correcte: "{{ ma_variable_precedente }}/test.txt"
```

Pour accéder à un élément d'un tableau, il suffit d'utiliser `{{ tableau[0] }}`. Pour accéder à un élément d'un dictionnaire, on utilise `{{ utilisateur.nom }}` ou `{{ utilisateur["nom"] }}`.

#### Les filtres

Jinja2 permet d'utiliser des filtres pour transformer nos variables. Ces filtres sont appelés via le caractère `|`. Exemple : `{{ ma_variable | lower }}` donnera forcément une chaine de caratères sans lettres capitales (filtre lower appliqué sur la variable ma_variable). On peut aussi s'en servir pour définir une valeur par défaut si la variable souhaitée venait à ne pas avoir été déclarée : `{{ admin_user | default('root') }}` ou encore `{{ une_variable | default(une_autre_variable) }}`.

#### Exercice 1 : Yaml

1. Accéder à une variable : Rédigez le Yaml qui fait en sorte que la variable suivante soit accessible : `{{ tableau[0].clef }}` en utilisant la notation recommandée pour les listes.
2. Filtres : Que fait `{{ ma_variable | default(variable2) | upper }}` ?
3. Variables imbriquées : On souhaite stocker des clefs dans des fichiers. Il y a deux variables : une avec le chemin d'accès du dossier qui va contenir toutes les clefs, une autre avec le chemin d'accès complet d'une des clefs. Montrez un exemple de ces deux variables. Laissez libre cours à votre imagination pour le chemin d'accès du dossier et le nom du fichier ;)

## Environnement du TP

Nous allons utiliser trois machines virtuelles, deux seront des serveurs nous permettant d'héberger nos services web et de base de données, une nous servira à controler et lancer les commandes ansible.

Il faut se connecter sur eve-ng avec les identifiants fournis par l'enseignant. Vous aurez normalement trois machines, les deux premières seront celles controllées, et la troisième sera notre controller avec Ansible dessus.

### Préparation du controller

Pour démarrer, se connecter sur la machine qui nous servira de controlleur en cliquant dessus puis en utilisant les identifiants fournis par l'enseignant, puis mettez vous en root avec `sudo -i`.

Il faut ensuite vous authentifier pour accéder à internet. Tapez les deux lignes suivantes en remplaçant par vos identifiants UTT :

```bash
export http_proxy='http://identifiant:passe@10.23.0.9:3128'
export https_proxy='http://identifiant:passe@10.23.0.9:3128'
```

Il faut ensuite installer ansible et sshpass (un outil qui permet à ansible de se connecter via SSH à des machines). SSH est un protocole très connu de connexion à distance et d'ouverture de session à distance, et est activé sur l'immense majorité des serveurs. Sur votre machine, dans un terminal, tapez :

```bash
apt-get update 
apt-get install -y python3 git nano ansible sshpass
```

Explications :

* apt est un gestionnaire de paquets pour Linux, qui sert notamment à installer facilement des logiciels
* sudo permet d'exécuter des commandes en tant qu'administrateur de la machine (utilisateur `root`)
* `apt-get update` permet de mettre à jour la liste des paquets disponibles
* `apt-get install` permet d'installer des paquets, et l'option `-y` permet d'installer sans demander de confirmation.
  * Les paquets `python3` et `sshpass` permettent de faire fonctionner le paquet `ansible`.
  * Le paquet `git` permettra de récupérer le contenu de ce TP sur votre controller et `nano` permettra d'éditer des fichiers

Sur votre controller, tapez les commandes suivantes afin de récupérer le TP sur votre controller :

```bash
cd
git clone http://git.utt.fr/laruelli/pe-ansible.git
cd pe-ansible
```

Explications :

* La commande `cd` permet de `change directory` (changer de répertoire). Utilisée sans paramètres, elle vous déplace dans votre répertoire utilisateur.
* `git clone` permet de récupérer le contenu d'un repo git. Git est un système de gestion de code source qui permet de versionner et de travailler à plusieurs facilement.

Testez la connexion sur vos autres machines depuis le controller en faisant `ssh ubuntu@IP` avec l'IP de chaque hôte. Confirmez éventuellement les clefs. Une fois la connexion établie, tapez `exit` pour revenir sur votre controller.

### Manipulation de l'inventaire

Ansible base son travail sur l'inventaire. Eh oui, exécuter des tâches c'est une chose, savoir sur quelles machines les exécuter en est une autre ! C'est précisément le rôle de l'inventaire.

L'inventaire permet de :

* Définir les machines et les groupes de machines
* Déclarer les variables associées aux machines et aux groupes

Depuis votre ordinateur, allez dans le dossier inventory et regardez le fichier hosts.yml. Il s'agit d'un fichier sous cette forme

```yaml
groupe: # nom du groupe
  hosts:
    hote1: # nom de chaque machine dans le groupe
    hote2:
```

#### Exercice 2

Combien y a-t-il de groupes dans le fichier hosts actuel ? Combien de machines ?

Toutes les machines font également partie du groupe spécial `all` qui n'a pas besoin d'être déclaré.

### Les variables

On définit des variables dans l'inventaire. Certaines sont propres à vos besoins, certaines sont propre à Ansible. Un exemple, la variable `ansible_host` permet de définir l'IP réelle d'une machine. 

Pour connaitre vos adresses IP, ouvrez un terminal sur les deux machines 1 et 2, puis tapez `ip a sh | grep inet`. Le caractère `|` est composé en appuyant sur Alt Gr + 6. Deux adresse s'affichent normalement, une en 127.0.0.1 qui ne nous intéresse pas, car elle est locale, et une autre en `10.100.X.Y`. Notez bien cette dernière IP.

Si vous allez dans inventory/host_vars/machine1/main.yml, vous pourrez voir que la variable `ansible_host` attend une valeur. Sur le controller, faites `nano inventory/host_vars/machine1/main.yml` et ajoutez l'IP de la machine1 dans la variable `ansible_host` et dans la variable `mediawiki_url` et faites de même pour machine2 dans le dossier correspondant. Pour quitter l'éditeur de texte nano, les commandes sont affichées en bas : `Ctrl + O puis entrée` pour sauvegarder le fichier, `Ctrl + X` pour quitter.

Modifiez également le fichier `inventory/group_vars/servers/proxy.yml` pour mettre vos paramètres de connexion afin que les moudles Ansible puissent se connecter à internet lors de leur exécution sur les machines distantes.

#### L'ordre des variables

Mais là, une question devrait vous interroger... On définit deux fois la variable `ansible_host`. Comment est-ce possible ?

En réalité, les mêmes variables peuvent être définies à plusieurs endroit, et sont **évaluées hote par hote** (une même variable lors d'une même exécution peut être différente d'un hote à un autre). Ansible suit l'ordre suivant pour savoir quelle doit être la valeur d'une variable (à lire dans l'ordre suivant : une variable plus basse écrasera la variable plus haute).

* Les variables par défaut des rôles (on verra après ce que c'est)
* Les variables du groupe all
* Les variables du groupe de la tâche en cours
* Les variables de l'hôte
* Les variables découvertes par Ansible (au lancement d'un ensemble de tâche, Ansible va chercher à "découvrir" la machine et stocke ces informations (nombre de processeurs, OS, ...) dans des variables accessibles dans les tâches suivantes)
* Les variables de playbooks
* Les variables de role
* Les variables de block
* Les variables de tâche
* Les variables définies à la volée (par exemple on peut stocker le résultat d'une tâche dans une variable)
* Les varables passées en ligne de commande

[Plus d'informations ici](https://docs.ansible.com/ansible/latest/user_guide/playbooks_variables.html#understanding-variable-precedence)

Les variables de groupes sont définies dans les fichiers yaml suivants : `inventory/group_vars/nom_du_group/*.yml`. Les variables d'un hote sont définies dans les yaml suivants : `inventory/host_vars/nom_de_l_hote/*.yml`. *Les fichiers de chaque dossier sont lus dans l'ordre lexicographique.*

##### Exercice 3

Donc ici, `ansible_host` est bien défini au sein de chaque hôte donc c'est normal. Cela permet des combinaisons intéressantes. Par exemple, dans le fichier group_vars/servers/main.yml, je définis le nom d'utilisateur avec lequel on va se connecter. Comment pourrait-on faire pour changer cette valeur juste pour un hôte (sans supprimer cette ligne dans le fichier de groupe) ?

## Place à l'action

Maintenant que les bases sont posées, on peut désormais commencer à exploiter les capacités d'Ansible. Il faut tout de même définir de nouvelles notions.

* Les modules. Ils permettent d'effectuer des actions, comme par exemple le module ansible.builtin.apt qui permet d'installer des paquets via apt (comme ce qu'on a fait tout à l'heure à la main). Ils prennent plusieurs paramètres souvent fournis sous la forme de dictionnaire. La commande `apt-get install -y python3 git nano ansible sshpass` serait écrite en Ansible :

```yaml
ansible.builtin.apt:
  name:
    - git
    - nano
    - python3
    - ansible
    - sshpass
  state: "latest"
```

ou encore via une variable :

```yaml
# Fichier de variable de groupe
paquets_basiques:
- git
- nano
- python3
- ansible
- sshpass

# Dans le fichier de la tâche
ansible.builtin.apt:
  name: "{{ paquets_basiques }}"
  state: "latest"
```

Sachant que les variables peuvent être assez complexes, on peut faire des scénarios très intéressants avec des paquets différents par machine mais pourtant avec la même tâche, juste en changeant le niveau de déclaration de la variable `paquets_basiques`.

* Les playbooks, qui sont des ensembles de tâches (*book*) qui doivent être jouées (*play*) sur un ou plusieurs hotes.
* Les rôles qui sont un ensemble de tâches pouvant être lancées depuis un playbook. Plutôt que d'écrire toutes nos tâches directement dans un playbook, on les met dans des rôles. Ces rôles sont ensuite appelés par les playbooks via le module `include_role`. Cela permet de distribuer entre plusieurs infrastructure (ou même sur internet) un ensemble de tâche. On peut retrouver sur le net des rôles permettant de déployer facilement des serveurs de base de données, des serveurs web, ... Vous pouvez en retrouver dans le dossier `roles`.

Chaque dossier contient un rôle. Combien y a-t-il de rôles dans notre projet ? Chacun des rôles dispose de l'arboresence suivante : tasks/main.yml pour les tâches, defaults/main.yml pour les variables par défaut (si notre roles utilise des variables, celles-ci doivent avoir une valeur par défaut au cas où l'utilisateur du rôle n'aie pas redéfini les variables pour son usage). On verra après ce que signifient les autres fichiers. 

* Les collections. C'est un ensemble de rôles, de playbooks et de modules facilement distribuables et installables. C'est une nouveauté récente d'Ansible afin de clairifier et standardiser les échanges de rôles/playbooks/... La collection la plus importante et installée par défaut est `ansible.builtin`. Pour les besoins de ce TP, il faut installer la collection `community.mysql` afin d'utiliser les modules de base de données. L'outil `ansible-galaxy` est celui qui permet de télécharger, d'installer et créer des rôles ou des collections. Tapez `ansible-galaxy collection install community.mysql` sur votre controller afin d'installer la collection nécessaire pour les bases de données.

### Votre premier playbook

Ouvrez le fichier `init.yml` et décrivez son contenu. Quels sont les machines sur lequelles ce playbook va s'exécuter ? Quel est l'objectif de ce playbook ?

Lancez le playbook en tapant `ansible-playbook init.yml --ask-pass --ask-become-pass`. Explications :

* `ansible-playbook` est l'outil permettant de lancer des playbooks
* `init.yml` est le playbook
* `--ask-pass` permet de demander le mot de passe de connexion SSH
* `--ask-become-pass` permet de demander le mot de passe pour utiliser la commande sudo  et exécuter des commandes en administrateur (toutes les tâches indiquées par `become: true`). Nous nous connectons avec l'utilisateur `ubuntu`, il nous faut donc demander le mot de passe pour passer en `root`. Il suffit de taper entrée pour reprendre la même valeur que le mot de passe de connexion classique.

#### Exercice 4

1. Décrire ce qui s'affiche à l'écran.
2. Relancer une deuxième fois le playbook. Quelle différence ?

On constate également que la première tâche est `gather facts`. Cette tâche permet à Ansible de scanner la machine afin de créer des variables qui nous indiquent la version du système d'exploitation, la liste des cartes réseau, ... On s'en sert notamment afin de faire des tâches conditionnelles comme ici.

```yml
- name: Installation de Powerline sur Debian
  ansible.builtin.package:
    name:
      - fonts-powerline # Ce paquet n'est dispo que sur le système d'exploitation Debian. Cette tâche ne doit s'exécuter que sur des machines sous Debian
    state: latest
  when:
    - ansible_distribution == "Debian" # Variable définie directement par Ansible (facts)
```

La documentation officielle des facts est [ici](https://docs.ansible.com/ansible/latest/user_guide/playbooks_vars_facts.html).

On va agrémenter ce playbook afin d'y ajouter des fonctionnalités. Nous allons maintenant faire en sorte de ne plus avoir à saisir de mot de passe pour la connexion SSH. Cela est réalisable grâce au chiffrement asymétrique ! Le principe est qu'il y a deux clefs, ce qui est chiffré avec une ne peut être que déchiffré par l'autre. On garde une des clefs (clef privée), et on donne l'autre au public. Avec notre clef publique, les autres peuvent chiffrer un message et vu que nous seuls avons l'autre clef pour déchiffrer, c'est confidentiel. Dans l'autre sens, si on chiffre quelque chose avec notre clef privée, techniquement tout le monde pourra le déchiffrer grâce à notre clef. Or, vu que l'on est le seul à posséder la clef qui a permis de chiffrer ce message, notre message est authentifié. (chaque couple de clef est unique, donc si quelqu'un nous donne sa clef publique et qu'on reçoit un message que seule sa clef publique peut déchiffer, alors forcémenet c'est associé à sa clef privée). Le chiffrement assymétrique est très pratique dans des environnements non sécurisés, car la seule chose à transmettre de manière non sécurisée est la clef publique (or elle est publique donc ce n'est pas grave si quelqu'un écoute nos conversations et tombe dessus). Le tout est que la clef privée reste absolument privée, si quelqu'un nous la vole, tout le principe s'effondre.

SSH exploite cet outil, et nous allons l'utiliser pour ne plus avoir à saisir de mot de passe. Cela se fait en deux étapes : générer un couple de clef sur notre pc, puis mettre la clef publique sur chacun des serveurs (dans un dossier particulier connu par le service de connexion SSH). Pour la partie génération des clefs, c'est assez simple. Sur votre controller, vous pouvez générer votre clef en tapant `ssh-keygen`, on vous demande où stocker la clef. Notez bien le chemin où la clef sera enregistrée, puis appuyez sur suivant pour choisir un mot de passe associée à la clef privée et qui permettra de déverouiller son utilisation (cela consiste une protection au cas où la clef tomberait entre de mauvaises mains). La deuxième partie consiste à copier la clef sur chacun des serveurs à un à un en utilisant certains utilitaires (scp ou autres). C'est long et pénible, et il faut garder une trace de "a-t-on déployé la clef sur ce serveur ?". Autant qu'Ansible le fasse pour nous !

#### Exercice 5

En lisant [la documentation du module authorized_key d'Ansible](https://docs.ansible.com/ansible/latest/collections/ansible/posix/authorized_key_module.html), complétez le playbook `init.yml` en vous inspirant des tâches déjà existantes dans le playbook afin d'y ajouter un tâche permettant de s'assurer que notre clef publique autorise la connexion sur l'utilisateur root. Indications :

* Le fichier contenant la clef publique correspond au chemin que vous avez noté plus haut lors de création suffixé par `.pub`
* Pour récupérer le conteu d'un fichier, on peut utiliser la fonction lookup avec le paramètre `file` : `ma_varible: {{ lookup('file', 'chemin_vers_la_fichier' }}`.

Appliquez ensuite le playbook avec la commande vue plus haut. Pour vérifier, testez maintenant sur votre controller :

* Tapez `eval $(ssh-agent)` pour lancer le système qui fournit les clefs privées lors des connexions SSH
* Il faut ajouter notre clef : `ssh-add ~/.ssh/id_rsa`, vérifier avec `ssh-add -L` qu'une ligne apparait bien pour signaler qu'une clef a été ajoutée.
* On peut maintenant retenter de se connecter : `ssh root@machine1` et constater qu'on ne nous demande plus de mot de passe de connexion ! Désormais, nous pourrons lancer tous nos playbook sans indiquer `--ask-pass --ask-become-pass`.

### Manipulation des rôles

On a pu voir ce qu'était un playbook : on y spécifie des hotes, et des tâches. Le problème, c'est que c'est assez peu portable/ditribuable. Les playbooks sont finalement qu'un ensemble en vrac de fichier yaml. Difficile dans tout ça de distinguer les tâches, les fichiers à copier, les templates, les variables par défaut, ... Et surtout, que distribuer ? Un ensemble de YAML ? Tout bien rangé des dossiers ? Avec quel standard pour la nomenclature des dossiers ?

Les rôles viennent répondre à cette problèmatique. Voilà comment un role est structuré :

```
nom-du-role/
  tasks/
    main.yml
  handlers/
    main.yml
  library/
    main.yml
  files/
    main.yml
  templates/
    main.yml
  vars/
    main.yml
  defaults/
    main.yml
  meta/
    main.yml
```

#### Exercice 6

En regardant le chemin d'accès du dossier des rôles dans `ansible.cfg`, puis en regardant dans le dossier en question, indiquer combien il y a de rôles et leurs noms.

Dans ces dossiers, le premier fichier qui est toujours lu est `main.yml`.
#### Tâches

Le dossier tâches contient une suite de tâches. On peut séparer les tâches en plusieurs fichiers pour plus de lisibilité. Pour importer des tâches d'un fichier (exemple fictif `config_zsh.yml`), on rajoute cette tâche dans le fichier `main.yml` :

```yml
- name: Config ZSH
  include_tasks: config_zsh.yml
```

#### Handlers

Le dossier handlers contient des tâches particulières. En effet, certaines de nos tâches traditionnelles du dossier `tasks` peuvent nécessiter de redémarrer un service si la tâche change quelque chose, mais seulement si configuration change, et uniquement à la fin de toutes les tâches.

Exemple dans le rôle webserver. Dans les tâches, trouver la tâche qui modifie la config d'Apache (le serveur web). Elle comporte une entrée :

```yml
notify:
    - Restart Apache # On peut notifier plusieurs handlers sous forme de liste
```

Dans le dossier handlers, on retrouve une tâche du même nom (`Restart Apache`). Cette tâche ne sera exécutée que si elle est notifiée au moins une fois par une tâche ayant l'état `changed`. [La documentation officielle sur les handlers](https://docs.ansible.com/ansible/latest/user_guide/playbooks_handlers.html).

#### Library

Le dossier library contient des modules pythons personnalisés qui peuvent être utilisés dans des tâches.

#### Files

Le dossier files contient des fichiers qui ont vocation à être copiés sur l'hote. Quand un module fait référence à un fichier, comme [copy](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/copy_module.html) ou [unarchive](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/unarchive_module.html), le nom de fichier dans le champ `src` fait référence à un nom de fichier dans le dossier `files`.

#### Templates

Les templates sont des fichiers de configuration destinés à être copiés sur l'hote via le module [template](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/template_module.html). La spécificité est que les templates sont des fichiers comprenant des variables avec la syntaxe jinja2. Le fichier est lu, les variables sont remplacées puis le fichier final est envoyé sur la machine distante.

Le fichier est lu avec la syntaxe Jinja2. On peut faire itérer jinja2 sur un tableau. Avec un tableau de dictionnaire, comme par exemple :

```yml
monTableau:
  - nom: "Wow"
    autreClef: "ee"
  - nom: "Wowo"
    autreClef: "ii"
```

Si dans notre template on a :

```j2
{% for valeur in monTableau %}
Données : {{ valeur.nom }}-{{ valeur.autreClef | upper }}
{% endfor %}
```

cela générera :

```
Données : Wow-EE
Données : Wowo-II
```

On peut également insérer du texte si une valeur est définie :

```
{% if maVariable is defined %}
TEXTE A INSERERR
{% endif %}
```

ou encore des tests logiques (ici `maVariable` est un booléen) :

```
{% if maVariable and ansible_distribution == 'Centos' %}
INSERER DU TEXTE
{% endif %}
```

On peut parfaitement imbriquer des variables, des for dans des if. Les variables sont toujours référencés via les accolades : `{{ maVariable }}` tandis que les instructions logiques sont référencées par `{% if/for %}` et `{% endif/enfor %}`.

Les [possibilités de templating](https://docs.ansible.com/ansible/latest/user_guide/playbooks_templating.html) sont assez phénoménales avec Ansible, ce qui rend Ansible très populaire.

#### Vars

Le dossier vars contient des variables qui ne peuvent pas être écrasés sauf en ligne de commande. C'est assez pratique quand notre rôle contient des variables que l'on veut pouvoir changer, mais que l'on veut s'assurer que ces variables soient strictement identiques d'un hote/groupe à un autre (cf priorité des variables vues plus haut).

#### Defaults

Le dossier defaults contient les variables par défaut. En effet, selon vos besoins, votre rôle va avoir besoin de variables qui devront être fixées dans l'inventaire groupe/hôte. Si jamais une variable venait à ne pas être définie dans l'inventaire ni dans toute la chaine des priorités de variable vue plus haut, sa valeur serait cherchée dans le dossier defaults. Si cette valeur n'est pas trouvée dans ce dossier, une erreur est déclenchée par Ansible.

#### Meta

Le dossier meta comprend des données sur le rôle en lui même, très pratique quand on distribue le rôle : dépend-il d'autres rôles, quel est son auteur, vers quel site puis-je trouver de la documentation, quelle est la version du rôle, ...

#### Tests et .travis.yml

Ces fichiers servent à faire du CI/CD sur le rôle afin de vérifier sa qualité et son fonctionnement. Cela est notamment fortement recommandé pour des rôles très utilisés et comprenant plusieurs développeurs en simultané.

La commande `ansible-galaxy init nom-du-role` va créer les dossiers nécessaires pour `nom-du-role`.

En réalité, dans l'arboresence vue plus haut, le dossier `tasks` suffit pour faire un rôle valide.

La [documentation officielle des rôles](https://docs.ansible.com/ansible/latest/user_guide/playbooks_reuse_roles.html)  est très détaillée.

#### Exercice 7

1. Combien y a-t-il de tâches dans le rôle mediawiki ?
2. Dans le rôle mediawiki, quelle est la valeur par défaut de `mediawiki_path` ?
3. En lisant le template dans webserver, donnez un exemple de virtualhost suffisant pour le template.
4. En lisant le template dans webserver, si je veux ajouter un certificat SSL, dois-je simplement définir les variables `ssl.cert_path` et `ssl.key_path` ? Que dois-je faire ?

### utiliser un rôle dans un playbook

Dans un playbook, on peut dire à un certain groupe de machines d'exécuter un rôle de la manière suivante :

```yml
- hosts: servers
  name: Installation de la base de données
  include_role:
    name: "dbserver"
  become: true
```

C'est exactement ce qui est fait dans le plyabook `deploy_mediawiki.yml`. Allez le voir, et constatez que les trois rôles sont lancés dans ce playbook.

Les variables nécessaires au bon déroulement du playbook sont déjà définies.

Rendez-vous maintenant sur votre navigateur et pour chaque machine tester "ip_machine:8080" comme URL. Bravo !

## Déployer SSL

Nous souhaitons un peu personnaliser la configuration en incluant un certificat SSL pour faire du HTTPS, mais sur un seul serveur !

Il y a plusieurs phases à cela :

1. Générer le certificat SSL
2. Modifier la configuration afin de référencer le certificat et relancer le playbook précédent

### Générer le certificat

La génération se fait via le playbook `ssl.yml`.

#### Exercice 8

1. Que fait chaque tâche de ce playbook ? Aidez-vous d'internet en tapant sur Google pour chaque module `ansible nom_module docs`.
2. A l'aide de la documentation [du module template](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/template_module.html), indiquez à quoi sert `validate` dans le module `template`.
3. En lisant bien ce playbook et le template associé, indiquez toutes les variables obligatoires afin de générer sans erreur le certificat.
4. Afin de déployer votre certificat sur machine1, où mettriez-vous les variables associées à la génération du certificat ?
5. Complétez l'inventaire afin de définir les bonnes variables

Indications :

1. SSL fonctionne avec les noms DNS des machines. Nous n'avons pas de DNS, donc comme seule et unique entrée vous devez donner l'IP de machine1.
2. Le chemin pour le dossier des certificats ne doit pas avoir de `/` à la fin.

Lancez le playbook ainsi `ansible-playbook ssl.yml --limit machine1`. Quand on ne sait pas à l'avance sur quel groupe ou quel machine on va faire tourner un playbook, on dit qu'il tourne sur tous les hotes (`hosts: all`), et on utilise l'argument `--limit nom_machine` afin de ne faire tourner le playbook que sur une seule machine. On peut éventuellement doubler avec vérification dans le playbook qu'une limitation est bien faite et qu'on est pas en train d'exécuter le playbook sur toutes les machines.

### Adapter la configuration

#### Exercice 9

Maintenant que notre certificat est généré sur machine1, il faut adapter la configuration de notre web server, dans les virtualhosts.

1. Où est actuellement défini le virtualhost (dans quel groupe / quel hote) ?
2. Si je veux laisser la configuration générale du groupe et faire une configuration spécifique pour une machine, que dois-je faire (quelle variable mettre à quel endroit) ?
3. Appliquez la question 2 et modifiez la configuration pour y rajouter votre certificat SSL. N'oubliez pas de modifier l'URL de votre mediawiki dans vos variables.

Relancez le playbook web, reconnectez-vous sur l'URL de machine1 en mettant https et vous devriez accéder à MediaWiki !

## Pour aller plus loin

1. Qu'est-ce qu'Ansible Vault ? Pourquoi et comment l'utiliser ? Essayez de l'utiliser dans le projet actuel.
2. Quelles utilisations possibles entre Ansible et un fournisseur cloud (Proxmox, AWS, GCP, ...) ?
3. Qu'est-ce qu'Ansible-Lint ? Quel intêrét ?
3. Quel est l'intérêt d'AWX / Ansible Tower ?
