# TP Ansible

Ce TP sur Ansible est destiné à des étudiants s'intéressant aux notions suivantes : cloud, automatisation, serveurs, provisionning, déploiement.

Il a été conçu et rédigé par [Ivann LARUELLE](https://ivannlaruelle.fr), étudiant en RT à l'UTT en P21, administrateur du [Système d'Information des Associations de l'UNG](https://ung.utt.fr/tech/sia) et encadré par l'enseignante [Samiha AYED](https://recherche.utt.fr/research-directory/samiha-ayed).

## Introduction

### Vous avez dit Ansible ?

Quand une tâche devient répétitive, il convient d'automatiser son exécution. C'est le principe même de l'informatique. Cela s'étend à la notion de fonctions en programmation, qui sont des fragments de code réutilisables, très pratique quand on cherche à développer une application qui répond à un besoin métier (gérer les étudiants, les examens, ...).

En administration système, où on a des besoins de gestion système (administrer des machines pour assurer le bon fonctionnement des applications qui s'y exécutent, ou administrer des postes utilisateurs, qu'ils soient fixes ou portable afin que la configuration des postes réponde à la politique IT de l'entreprise en terme de fonctionnalité, de sécurité, ...), la même problématique se pose que lors du développement d'une application : certaines tâches sont répétitives. Imaginez devoir gérer 1 200 postes utilisateurs dans une même société, ou devoir ajouter des serveurs régulièrement et maintenir identique la configuration de sécurité, ou installer et configurer certaines choses en commun sur ces serveurs ! Ansible sert à ça.

Dans ce TP, nous allons voir comment certaines fonctionnalités majeures d'Ansible peuvent nous servir afin de déployer un serveur MediaWiki. MediaWiki est un service web qui sert de base à Wikipédia. Ce service est open-source, donc n'importe qui (à condition d'avoir le matériel pour) peut installer MediaWiki sur un serveur. MediaWiki nécessite une base de données afin de stocker ses données (MariaDB dans notre exemple), un serveur web afin d'assurer son accès depuis les autres machines en utilisant le procole HTTP (Apache dans notre exemple) et un outil permettant d'interpréter son code source afin de savoir comment traiter les requêtes (les développeurs de MediaWiki utilisent PHP, nous allons donc devoir installer PHP).

### Oui, Ansible !

Ansible est un outil open source édité par Red Hat, une multinationale célèbre pour son système d'exploitation orienté serveurs (CentOS), sa distribution Kubernetes (OpenShift), et bien d'autres outils (quay.io, ...).

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

Ansible utilise la syntaxe Jinja2 pour exploiter les variables. Pour faire référence à une variable, on utilise `{{  }}`, exemple : `{{ ma_variable }}`. On peut imbriquer des variables entre elles : par exemple : `chemin_clef_secrete: /etc/ssl/{{ dossier_secret }}/clef.private`. **Attention**, si votre variable imbriquée commence par une autre variable, toute la chaine de caractères doit être encadrée par des guillemets. Exemple :

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

Le pavé numérique est parfois instable sur l'interface via le navigateur, privilégiez `Maj + chiffre en haut du clavier` pour saisir des chiffres.

Pour lire un fichier, utilisez la commande `less` : `less chemin/vers/le/fichier`. Vous pouvez utiliser les flêches du clavier pour vous déplacer. Apuuyez sur q pour quitter.

Pour modifier un fichier, utilisez `nano` : `nano chemin/vers/le/fichier`. Vous pouvez déplacer le curseur avec les flèches du clavier. Les commandes sont affichées en bas, les principales sont `Ctrl + O` puis Entrée pour sauvegarder, `Ctrl + X` pour quitter. Pour quitter sans sauvegarder, `Ctrl + X` puis appuyez sur `n` pour refuser d'enregistrer.

Quand vous tapez un chemin d'accès à un fichier dans la ligne de commande (pour less ou nano par exemple), utilisez les tabulations (touche `tab` à gauche) afin de compléter automatiquement les chemins d'accès selon les premières lettres que vous avez déjà tapé.

Pour récupérer une commande déjà tapée, utilisez les fléches du clavier (Haut et Bas) pour naviguer dans l'historique des commandes.

### Préparation du controller

Pour démarrer, se connecter sur la machine qui nous servira de controlleur (la troisième) en cliquant dessus puis en utilisant les identifiants fournis par l'enseignant, puis mettez vous en root avec `sudo -i` : sudo permet d'exécuter des commandes en tant qu'administrateur de la machine (utilisateur `root`), `sudo -i` permet d'ouvrir une session root et de la maintenir.

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
* `apt-get update` permet de mettre à jour la liste des paquets disponibles
* `apt-get install` permet d'installer des paquets, et l'option `-y` permet d'installer sans demander de confirmation.
  * Les paquets `python3` et `sshpass` permettent de faire fonctionner le paquet `ansible`.
  * Le paquet `git` permettra de récupérer le contenu de ce TP sur votre controller et `nano` permettra d'éditer des fichiers

Sur votre controller, tapez les commandes suivantes afin de récupérer le TP sur votre controller :

```bash
cd
git clone https://github.com/larueli/pe-ansible.git
cd pe-ansible
```

Explications :

* La commande `cd` permet de `change directory` (changer de répertoire). Utilisée sans paramètres, elle vous déplace dans votre répertoire utilisateur.
* `git clone` permet de récupérer le contenu d'un repo git. Git est un système de gestion de code source qui permet de versionner et de travailler à plusieurs facilement.

Afin de clarifier le terminal, tapez `hostnamectl set-hostname machine3`, puis deux fois `exit` afin de vous déconnecter totalement et reconnectez vous puis remettez vous en root (`sudo -i`). Cela vous aidera à savoir rapidement sur quelle machine vous êtes. Replacez-vous bien dans le dossier du projet `pe-ansible` : `cd /root/pe-ansible`.

#### Préparation de chaque hote

Sur les deux premiers hotes, connectez-vous, saisissez `sudo -i` afin de devenir root, puis indiquez vos identifiants pour connecter à internet :

```bash
export http_proxy='http://identifiant:passe@10.23.0.9:3128'
export https_proxy='http://identifiant:passe@10.23.0.9:3128'
```

Il faut ensuite installer le serveur SSH qui va nous permettre de prendre le controle à distance de la machine : `apt-get update && apt-get install -y openssh-server`.

Afin de clarifier le terminal, tapez `hostnamectl set-hostname machine1/2` (changez selon le numéro de votre hote), puis `exit` et reconnectez vous. Cela vous aidera à savoir rapidement sur quelle machine vous êtes.

Pour connaitre vos adresses IP, ouvrez un terminal sur les deux machines 1 et 2, puis tapez `ip a sh | grep inet`. Le caractère `|` est composé en appuyant sur Alt Gr + 6. Deux adresse s'affichent normalement, une en 127.0.0.1 qui ne nous intéresse pas, car elle est locale, et une autre en `10.100.X.Y`. Notez bien cette dernière IP pour chaque machine.

#### Vérification

Revenez sur votre controlleur (3éme machine) et testez la connexion sur vos autres machines en faisant `ssh administrateur@IP` avec l'IP de chaque hôte. Confirmez éventuellement les clefs (`yes`). Une fois la connexion établie, vous pouvez constater que votre invite a changé de `root@machine3` à `administrateur@machine1/2`, preuve que vous êtes bien sur une machine différente. Tapez `exit` pour revenir sur votre controller.

### Manipulation de l'inventaire

Ansible base son travail sur l'inventaire. Eh oui, exécuter des tâches c'est une chose, savoir sur quelles machines les exécuter et avec quels paramètres en est une autre ! C'est précisément le rôle de l'inventaire.

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

On définit des variables dans l'inventaire. Certaines sont propres à vos besoins, certaines sont propres à Ansible. Un exemple, la variable `ansible_host` permet de définir l'IP réelle d'une machine.

Si vous allez dans `inventory/host_vars/machine1/main.yml`, vous pourrez voir que la variable `ansible_host` attend une valeur. Sur le controller, faites `nano inventory/host_vars/machine1/main.yml` et ajoutez l'IP de la machine1 dans la variable `ansible_host` et faites de même pour machine2 dans le dossier correspondant. Pour quitter l'éditeur de texte nano, les commandes sont affichées en bas : `Ctrl + O puis entrée` pour sauvegarder le fichier, `Ctrl + X` pour quitter.

Modifiez également le fichier `inventory/group_vars/servers/proxy.yml` pour mettre vos paramètres de connexion afin que les modules Ansible puissent se connecter à internet lors de leur exécution sur les machines distantes. Tapez `clear` pour effacer l'écran.

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

Donc ici, `ansible_host` est bien défini au sein de chaque hôte donc c'est normal. Cela permet des combinaisons intéressantes. Par exemple, dans le fichier `group_vars/servers/main.yml`, je définis le nom d'utilisateur avec lequel Ansible va se connecter en SSH dans la variable `ansible_user`. Comment pourrait-on faire pour changer cette valeur juste pour un hôte (sans supprimer cette ligne dans le fichier de groupe) ?

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
* Les rôles qui sont un ensemble de tâches pouvant être lancées depuis un playbook. Plutôt que d'écrire toutes nos tâches directement dans un playbook, on les met dans des rôles. Ces rôles sont ensuite appelés par les playbooks via le module `include_role`. Cela permet de distribuer entre plusieurs infrastructures (ou même sur internet) un ensemble de tâches. On peut retrouver sur le net des rôles permettant de déployer facilement des serveurs de base de données, des serveurs web, des dns, ... Vous pouvez en retrouver dans le dossier `roles`.
* Les collections. C'est un ensemble de rôles, de playbooks et de modules facilement distribuables et installables. C'est une nouveauté récente d'Ansible afin de clairifier et standardiser les échanges de rôles/playbooks/... La collection la plus importante et installée par défaut est `ansible.builtin`. Pour les besoins de ce TP, il faut installer la collection `community.mysql` afin d'utiliser les modules de base de données. L'outil `ansible-galaxy` est celui qui permet de télécharger, d'installer et créer des rôles ou des collections. Tapez `ansible-galaxy collection install community.mysql` sur votre controller afin d'installer la collection nécessaire pour les bases de données.

### Votre premier playbook

Ouvrez le fichier `init.yml`.

#### Exercice 4

1. Décrivez les tâches. Quel est l'objectif de ce playbook ?
2. Quels sont les machines sur lequelles ce playbook va s'exécuter ?

Lancez le playbook en tapant `ansible-playbook init.yml --ask-pass --ask-become-pass`. Explications :

* `ansible-playbook` est l'outil permettant de lancer des playbooks
* `init.yml` est le playbook
* `--ask-pass` permet de demander le mot de passe de connexion SSH
* `--ask-become-pass` permet de demander le mot de passe pour utiliser la commande sudo  et exécuter des commandes en administrateur (toutes les tâches indiquées par `become: true`). Nous nous connectons avec l'utilisateur `administrateur`, il nous faut donc demander le mot de passe pour passer en `root`. Il suffit de taper entrée pour reprendre la même valeur que le mot de passe de connexion classique.

3. Décrire ce qui s'affiche à l'écran et mettez une capture d'écran du résultat.
4. Relancer une deuxième fois le playbook et mettez une capture d'écran du résultat. Quelle différence ?

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

On va agrémenter ce playbook afin d'y ajouter des fonctionnalités. Nous allons maintenant faire en sorte de ne plus avoir à saisir de mot de passe pour la connexion SSH. Cela est réalisable grâce au chiffrement asymétrique ! Le principe est qu'il y a deux clefs, ce qui est chiffré avec une ne peut être que déchiffré que par l'autre. On garde une des clefs (clef privée), et on donne l'autre au public. Avec notre clef publique, les autres peuvent chiffrer un message et vu que nous seuls avons l'autre clef pour déchiffrer, c'est confidentiel. Dans l'autre sens, si on chiffre quelque chose avec notre clef privée, techniquement tout le monde pourra le déchiffrer grâce à notre clef. Or, vu que l'on est le seul à posséder la clef qui a permis de chiffrer ce message, notre message est authentifié, seul nous avons pu l'écrire. Chaque couple de clef est unique, donc si quelqu'un nous donne sa clef publique et qu'on reçoit un message que seule sa clef publique peut déchiffer, alors forcément c'est que sa clef privée, qu'il est le seul à avoir, a chiffré le message. Le chiffrement asymétrique est très pratique dans des environnements non sécurisés, car la seule chose à transmettre est la clef publique. Le tout est que la clef privée reste absolument privée, car si quelqu'un nous la vole, tout le principe s'effondre. Ce chiffrement est rendu possible grâce à des propriétés arithmétiques complexes trouvées par Whitfield Diffie, Martin Hellman et Ralph Merkle mais dont l'implémentation la plus populaire est celle de Ronald Rivest, Adi Shamir et Leonard Adleman (algorithme RSA). [Un exemple simple et non-optimisé de chiffrement RSA a été réalisé dans le cours de NF05](https://github.com/larueli/RSANF05UTT).

SSH exploite ce chiffrement (tout comme HTTPS, avec [la notion de confiance en plus](https://fr.wikipedia.org/wiki/Certificat_%C3%A9lectronique)), et nous allons l'utiliser pour ne plus avoir à saisir de mot de passe. Cela se fait en deux étapes : générer un couple de clef sur notre pc, puis mettre la clef publique sur chacun des serveurs (dans un dossier particulier connu par le service de connexion SSH). Pour la partie génération des clefs, c'est assez simple. Sur votre controller, vous pouvez générer votre clef en tapant `ssh-keygen`, on vous demande où stocker la clef. Laissez la valeur par défaut mais notez bien le chemin où la clef sera enregistrée, puis appuyez sur entrée pour choisir un mot de passe associée à la clef privée et qui permettra de déverouiller son utilisation (cela consiste une protection au cas où la clef tomberait entre de mauvaises mains). Vous êtes libres de ne pas en choisir (mot de passe vide) car cette clef ne sera utilisée que dans le cadre de ce TP. La deuxième partie consiste à copier la clef sur chacun des serveurs un à un en utilisant certains utilitaires (scp ou autres). C'est long et pénible, et il faut garder une trace de "a-t-on déployé la clef sur ce serveur ?".

Autant qu'Ansible le fasse pour nous !

#### Exercice 5

En lisant [la documentation du module authorized_key d'Ansible](https://docs.ansible.com/ansible/latest/collections/ansible/posix/authorized_key_module.html), complétez le playbook `init.yml` en vous inspirant des tâches déjà existantes dans le playbook afin d'y ajouter une tâche permettant de s'assurer que notre clef publique autorise la connexion sur l'utilisateur root. Indications :

* Le fichier contenant la clef publique correspond au chemin que vous avez noté plus haut lors de création suffixé par `.pub`
* Pour récupérer le conteu d'un fichier, on peut utiliser la fonction lookup avec le paramètre `file` : `ma_varible: {{ lookup('file', 'chemin_vers_la_fichier' }}`.

Comme indiqué dans la documentation du module, ce dernier appartient à la collection `ansible.posix` qu'il convient d'installer avec la commande suivante : `ansible-galaxy collection install ansible.posix`. Appliquez ensuite le playbook avec la commande vue plus haut. Pour vérifier, testez maintenant sur votre controller :

* Tapez `eval $(ssh-agent)` pour lancer le système qui fournit les clefs privées lors des connexions SSH
* Il faut ajouter notre clef : `ssh-add /root/.ssh/id_rsa`, vérifier avec `ssh-add -L` qu'une ligne apparait bien pour signaler qu'une clef a été ajoutée.
* On peut maintenant retenter de se connecter : `ssh root@IP_machine_1/2` et constater qu'on ne nous demande plus de mot de passe de connexion, et qu'on est directement connecté en root sur notre machine ! Désormais, nous pourrons lancer tous nos playbook sans indiquer `--ask-pass --ask-become-pass`.

Dans le fichier `inventory/group_vars/servers/main.yml`, changez `ansible_user: administrateur` par `ansible_user: root`, et relancez le playbook `init.yml` mais sans options : `ansible-playbook init.yml`.

### Manipulation des rôles

On a pu voir ce qu'était un playbook : on y spécifie des hotes, et des tâches. Le problème, c'est que c'est assez peu portable/ditribuable. Les playbooks sont finalement qu'un ensemble en vrac de fichier yaml. Difficile dans tout ça de distinguer les tâches, les fichiers à copier, les templates, les variables par défaut, ... Et surtout, que distribuer ? Un ensemble de YAML ? Tout bien rangé dans des dossiers ? Avec quel standard pour la nomenclature des dossiers ?

Les rôles viennent répondre à cette problèmatique. Voilà comment un role est structuré (le premier fichier qui est toujours lu est `main.yml`) :

```raw
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

#### Tasks

Le dossier tasks contient les tâches. On peut séparer les tâches en plusieurs fichiers pour plus de lisibilité. Pour importer des tâches d'un fichier (exemple fictif `config_zsh.yml`), on rajoute cette tâche dans le fichier `main.yml` :

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

```raw
Données : Wow-EE
Données : Wowo-II
```

On peut également insérer du texte si une valeur est définie :

```j2
{% if maVariable is defined %}
TEXTE A INSERERR
{% endif %}
```

ou encore des tests logiques (ici `maVariable` est un booléen) :

```j2
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

La commande `ansible-galaxy init nom-du-role` permet de créer les dossiers nécessaires pour `nom-du-role`.

En réalité, dans l'arboresence vue plus haut, le dossier `tasks` suffit pour faire un rôle valide.

La [documentation officielle des rôles](https://docs.ansible.com/ansible/latest/user_guide/playbooks_reuse_roles.html)  est très détaillée.

#### Exercice 7

1. Combien y a-t-il de tâches dans le rôle mediawiki ?
2. Dans le rôle mediawiki, quelle est la valeur par défaut de `mediawiki_path` ?
3. En lisant le template dans webserver, donnez un exemple de virtualhost suffisant pour le template.
4. Le fichier de configuration de mediawiki ne doit pas être installé sur le serveur avant son premier lancement depuis un naviagteur. Comment cela se traduit-il dans la tâche de configuration de mediawiki ?
5. Certaines tâches comprennent `loop` et font référence à une variable `item`. Pourquoi ? A quoi ça sert ? N'hésitez pas à vous aider d'internet.

### Utiliser un rôle dans un playbook

Dans un playbook, on peut dire à un certain groupe de machines d'exécuter un rôle de la manière suivante :

```yml
- hosts: servers
  name: Installation de la base de données
  include_role:
    name: "dbserver"
  become: true
```

C'est exactement ce qui est fait dans le plyabook `deploy_mediawiki.yml`. Allez le voir, et constatez que les trois rôles sont lancés dans ce playbook.

Les variables nécessaires au bon déroulement du playbook sont déjà définies. Installez bien les collections:

* `community.mysql` qui permet de manipuler les bases de données avec la commande `ansible-galaxy collection install community.mysql`
* `community.general` qui contient notamment des modules pour Apache, notre serveur web avec la commande `ansible-galaxy collection install community.general`

Lancez ensuite le playbook avec `ansible-playbook deploy_mediawiki.yml`. Si demain vous venez rajouter 4 autres machines, il suffit des les ajouter dans le groupe servers dans `inventory/hosts.yml`, de leur créer un dossier `inventory/host_vars/nom_machine` et d'y mettre un fichier yml contenant les mêmes paramètres que les autres machines (IP, ...) et de relancer le playbook ! Ansible nous simplifie vraiment la tâche pour provisionner des machines à la volée.

Rendez-vous maintenant sur votre navigateur et pour chaque machine tester "http://ip_machine" comme URL. Si jamais vous n'avez pas de navigateur, utilisez la commande curl depuis la machine controller : `curl http://IP_machine1/2 | less` si vous voyez défiler du HTML qui parle de Mediawiki, c'est tout bon ! Bravo !

Si vous utilisez un navigateur, MediaWiki vous proposera ensuite de récupérer un fichier de configuration appelé `LocalSetting.php`. Téléchargez le, comparez le avec le template du role mediawiki, apportez les corrections nécessaires au template, changez la variable `mediawiki_is_already_installed` à `true` pour la machine configurée afin de débloquer la tâche de déploiement de la configuration et faites de même pour l'autre machine.

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

1. SSL fonctionne avec les noms DNS des machines mais aussi avec leur IP. Pour simplifier, vous devez donner l'IP de machine1.
2. Le chemin pour le dossier des certificats ne doit pas avoir de `/` à la fin.
3. Le port standard pour du https est le 443.

Lancez le playbook ainsi `ansible-playbook ssl.yml --limit machine1`. Quand on ne sait pas à l'avance sur quel groupe ou quel machine on va faire tourner un playbook, on dit qu'il tourne sur tous les hotes (`hosts: all`), et on utilise l'argument `--limit nom_machine` afin de ne faire tourner le playbook que sur une seule machine. On peut éventuellement doubler avec vérification dans le playbook qu'une limitation est bien faite et qu'on est pas en train d'exécuter le playbook sur toutes les machines.

### Adapter la configuration

#### Exercice 9

Maintenant que notre certificat est généré sur machine1, il faut adapter la configuration de notre web server, dans les virtualhosts.

1. Où est actuellement défini le virtualhost (dans quel groupe / quel hote) ?
2. Si je veux laisser la configuration générale du groupe et faire une configuration spécifique pour une machine, que dois-je faire (quelle variable mettre à quel endroit) ?
3. Appliquez la question 2 et modifiez la configuration pour y rajouter votre certificat SSL. Regardez la section SSL dans le template du rôle webserver pour savoir quelles variables ajouter pour un virtualhost SSL. N'oubliez pas de modifier l'URL de votre mediawiki dans vos variables.

Relancez le playbook `deploy_mediawiki.yml`.

Vous pouvez désormais ressayer la commande `curl` depuis le controller, mais avec du https : `curl https://IP_machine_1 | less`. curl devrait se plaindre qu'on ne peut pas faire confiance au certificat. Dans ce cas, recommencer mais avec l'option `--insecure` : `curl --insecure https://IP_machine_1 | less`

## Pour aller plus loin

1. Qu'est-ce qu'Ansible Vault ? Pourquoi et comment l'utiliser ? Essayez de l'utiliser dans le projet actuel.
2. Quelles utilisations possibles entre Ansible et un fournisseur cloud (Proxmox, AWS, GCP, ...) ?
3. Qu'est-ce qu'Ansible-Lint ? Quel intêrét ?
4. Quel est l'intérêt d'AWX / Ansible Tower ?

## Troubleshoot

### Ansible ne trouve pas mes fichiers YML

* Etes-vous bien sur machine3 ?
* Etes-vous connecté en root ? L'invite de commande devrait afficher `root@machine3`, si c'est `administrateur@machine3`, tapez `sudo -i`
* Etes-vous bien dans le dossier `/root/pe-ansible` ? Vérifiez avec `pwd`, si ce n'est pas le cas : `cd /root/pe-ansible`

### Erreur de proxy

#### Dans un playbook

Vérifiez deux fois que le fichier `inventory/group_vars/servers/proxy.yml` contienne bien vos informations de connexion UTT. Veillez à bien encadrer votre mot de passe par des guillemets, surtout s'il posséde des caractères spéciaux. Le pavé numérique est parfois instable sur l'interface via le navigateur, privilégiez `Maj + chiffre en haut du clavier` pour saisir des chiffres.

#### Sur un hote (en faisant apt-get, ...)

Faites `echo $http_proxy`, puis `echo $https_proxy`. Si rien ne s'affiche vous devez saisir vos identifiants ainsi :

```bash
export http_proxy='http://identifiant:passe@10.23.0.9:3128'
export https_proxy='http://identifiant:passe@10.23.0.9:3128'
```

Veillez à bien encadrer les variables avec des guillemets **simples** comme dans l'exemple, car les guillemets double entrainent l'interprétation de la chaine de caractères par bash, ce que nous ne souhaitons pas. Le pavé numérique est parfois instable sur l'interface via le navigateur, privilégiez `Maj + chiffre en haut du clavier` pour saisir des chiffres.

Faites `clear` pour effacer la console et masquer vos identifiants.

### Ansible ne trouve pas les modules

Avez vous bien installés les trois collections supplémentaires ?

* `ansible-galaxy collection install ansible.posix`
* `ansible-galaxy collection install community.mysql`
* `ansible-galaxy collection install community.general`

### Ansible ne se lance pas

* Etes-vous bien sur la machine 3 ?
* Votre proxy fonctionne-t-il ?
* Installez bien le paquet ansible : `apt-get update && apt-get install -y ansible`

### Ansible ne se connecte pas en SSH

Le pavé numérique est parfois instable sur l'interface via le navigateur, privilégiez `Maj + chiffre en haut du clavier` pour saisir des chiffres.

* Etes-vous bien sur la machine 3 ?
* Avez vous installé sshpass ? `apt-get update && apt-get install -y sshpass`
* Avec quel utilisateur vous-vous vous connecter ? La méthode avec mot de passe est sur l'utilisateur administrateur, la méthode sans mot de passe via clef SSH est sur l'utilisateur `root`. Vérifiez donc que la variable `ansible_user` dans `inventory/group_vars/servers/main.yml` corresponde.
* Tentez manuellement de vous connecter en SSH : sur machine3, tapez `ssh administrateur@IP_MACHINE`, et regardez. Avez-vous bien accepté les clefs ? Vous devez à minima vous connecter au moins une fois à la main sur chaque hote en SSH depuis machine3 afin d'accepter les clefs.
* Sur chacun des hotes machine1/2, avez vous bien installé le paquet `openssh-server` : `apt-get update && apt-get install -y openssh-server`. Configurez votre proxy sur l'hote pour que la commande apt-get fonctionne.
* Pour la connexion en root sans mot de passe depuis machine3, avez-vous bien votre clef publique dans votre agent SSH ? `ssh-add -L`, si rien ne s'affiche, ce n'est pas bon. Recommencez la procédure avec `eval $(ssh-agent)` puis `ssh-add /root/.ssh/id_rsa`.
