# TP Ansible : haproxy et nginx en conteneurs Podman (Quadlet) sur une VM

Durée : environ 2 h 20 (défi en plus) · niveau débutant

## Objectif

Déployer avec Ansible, depuis ton poste, deux conteneurs sur une VM Debian 13 :

```
poste de contrôle ──SSH──► VM
                            ├─ haproxy  :80   seul port exposé (réseau tp + réseau back)
                            └─ nginx          réseau back interne : joignable uniquement par haproxy
```

Les conteneurs sont gérés par systemd grâce à Quadlet.

## Méthode

- Tu écris un seul fichier, `site.yml`, qui grandit d'étape en étape. À chaque étape tu le relances en entier : ce qui est déjà en place ne change pas.
- Le contenu technique (fichiers Podman, haproxy) est fourni. Les tâches Ansible, c'est à toi de les écrire, avec `ansible-doc ansible.builtin.<module>` (la section `EXAMPLES` en bas de page).
- Le résultat de chaque étape est vérifié par Ansible lui-même, dans un second play « Contrôles » que tu complètes au fil du TP.

## Avant de commencer

- **Poste de contrôle** : Linux, macOS ou WSL, avec Ansible (`ansible --version`).
- **VM Debian 13** en accès par pont (pas NAT), sans autre service sur le port 80. Quadlet exige Podman ≥ 4.4 : Debian 12 et Ubuntu 22.04 ne conviennent pas.
- **Accès SSH par clé** et **sudo sans mot de passe** pour ton compte (VM jetable).

```bash
ssh-keygen -t ed25519                  # si tu n'as pas de clé
ssh-copy-id ton_compte@IP_VM
```

Sur la VM, en console :

```bash
echo "$USER ALL=(ALL) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/tp
```

Dans la suite, remplace `IP_VM` et `ton_compte` par tes valeurs (`ip -br a` sur la VM donne l'IP).

---

## Étape 1 : Inventaire et commandes ad hoc (15 min)

**Notions** : inventaire, module, commande ad hoc.

Ansible n'installe rien sur la VM : il s'y connecte en SSH. Il lui faut deux informations : **quelles machines** piloter (l'*inventaire*) et **quelle action** exécuter (un *module* : une brique comme `ping` ou `command`).

Crée le dossier `~/tp-ansible` et le fichier `inventory.yaml`. C'est l'unique inventaire du TP : un seul groupe, `ma_vm_ansible`, qui contient ta VM.

```yaml
---
# Inventaire du TP : une VM, dans le groupe ma_vm_ansible
all:
  vars:
    ansible_user: ton_compte        # compte SSH, valable pour toutes les machines
  children:
    ma_vm_ansible:                  # groupe visé par les playbooks (hosts: ma_vm_ansible)
      hosts:
        mavm:                       # nom de la VM dans Ansible (surnom libre)
          ansible_host: IP_VM       # adresse IP réelle de la VM
```

`---` marque le début du document YAML et `#` commence un commentaire : les deux sont ignorés par Ansible.

**Comprendre l'indentation.** Chaque niveau d'indentation veut dire « appartient à ». En YAML, seule l'indentation dit à quoi se rattache chaque information. Lis le fichier comme une arborescence de dossiers :

```
all                                ← groupe racine : toutes les machines
├── vars                           ← variables communes à toutes les machines
│   └── ansible_user               ← le compte SSH
└── children                       ← les sous-groupes de all
    └── ma_vm_ansible              ← notre groupe
        └── hosts                  ← les machines de ce groupe
            └── mavm               ← la VM
                └── ansible_host   ← son adresse IP réelle
```

`children:` et `hosts:` lèvent l'ambiguïté : sans eux, Ansible ne saurait pas si `ma_vm_ansible` est un groupe ou une machine. Règles YAML : 2 espaces par niveau, jamais de tabulation, un espace après `:` quand une valeur suit. Un nom de groupe ne contient que des lettres, chiffres et underscores (pas de tiret).

Crée `ansible.cfg` dans le même dossier, pour ne pas taper `-i inventory.yaml` à chaque commande :

```ini
[defaults]
inventory = inventory.yaml
# supprime un avertissement sans importance sur l'interpréteur Python
interpreter_python = auto_silent
```

Une **commande ad hoc** lance un seul module, une seule fois, sans écrire de playbook : `ansible <cible> -m <module> -a "<arguments>"`.

| Option | Rôle |
|---|---|
| `-m` | le module à lancer. Sans `-m`, c'est le module `command`, qui exécute une commande sur la VM |
| `-a` | les arguments du module. Pour `command` : la commande à exécuter |
| `-b` | *become* : exécute avec sudo, donc en root |

```bash
ansible-inventory --graph
ansible ma_vm_ansible -m ping
ansible ma_vm_ansible -a "whoami"
ansible ma_vm_ansible -b -a "whoami"
```

`whoami` affiche le compte qui exécute la commande. `command` répond toujours `CHANGED`, même pour une simple lecture : il ne peut pas savoir si la commande modifie la VM (on y revient en 2.6).

**Contrôle**
- [ ] `ansible-inventory --graph` affiche `mavm` sous `@ma_vm_ansible`.
- [ ] `ping` répond `pong` : c'est un test de connexion Ansible (SSH + Python), pas un ping réseau.
- [ ] Le premier `whoami` affiche ton compte, le second affiche `root`.

---

## Étape 2 : Les notions du playbook (45 min)

**Notions** : playbook, play, tâche, facts, `assert`, `register`, tags, `--check`, `PLAY RECAP`.

Chaque notion ci-dessous est un playbook **indépendant** : tu le crées dans `~/tp-ansible/notions/`, tu le lances, puis tu le réutilises dans un exercice. Aucun ne dépend d'un autre. À la fin (2.10), tu assembles `site.yml`.

```bash
mkdir -p ~/tp-ansible/notions
cd ~/tp-ansible                    # tous les playbooks se lancent depuis ce dossier
```

### 2.1 Le playbook

Un **playbook** est un fichier YAML exécuté par `ansible-playbook`. Il contient une liste de *plays*.

`notions/2-1-playbook.yml`
```yaml
---
# Un playbook est un fichier YAML qu'on exécute avec : ansible-playbook <fichier>
# Il commence par --- et contient une liste de plays (chaque play commence par « - »).
- name: Mon premier playbook
  hosts: ma_vm_ansible
  tasks:
    - name: Dire bonjour depuis la VM
      ansible.builtin.debug:                  # debug affiche un message
        msg: "Bonjour depuis {{ inventory_hostname }}"   # nom de la machine dans l'inventaire
```

```bash
ansible-playbook notions/2-1-playbook.yml
```

Observe les blocs `PLAY`, `TASK`, puis le message.

**À toi** : écris `notions/2-1-bis.yml` sur ce modèle, pour afficher l'adresse IP de la VM, contenue dans la variable `ansible_host`.

### 2.2 Le play

Un **play** associe des machines (`hosts`) à une liste de tâches. Il porte aussi des options, comme `become`.

`notions/2-2-play.yml`
```yaml
---
# Un play associe des machines (hosts) à une liste de tâches (tasks).
# Un playbook peut contenir plusieurs plays : ils s'exécutent l'un après l'autre.
- name: Play 1, sur la VM
  hosts: ma_vm_ansible        # un groupe de l'inventaire
  become: true                # option du play : ses tâches s'exécutent avec sudo
  tasks:
    - name: Où suis-je ?
      ansible.builtin.debug:
        msg: "Exécuté sur {{ inventory_hostname }}"

- name: Play 2, sur le poste de contrôle
  hosts: localhost            # le poste qui lance ansible-playbook
  tasks:
    - name: Où suis-je ?
      ansible.builtin.debug:
        msg: "Exécuté sur {{ inventory_hostname }}"
```

```bash
ansible-playbook notions/2-2-play.yml
```

Observe : deux blocs `PLAY`, chacun sur sa machine.

**À toi** : ajoute un troisième play ciblant `all`. Sur quelles machines s'exécute-t-il ? `localhost` en fait-il partie ?

### 2.3 La tâche

Une **tâche** est un appel de module : un nom, le module, ses arguments. Ce sont les mêmes arguments qu'en ligne de commande (`-a "clé=valeur"`), écrits un par ligne.

`notions/2-3-tache.yml`
```yaml
---
# Une tâche = un nom (name) + un module + les arguments du module.
# Ce sont les arguments de la ligne de commande ad hoc, écrits un par ligne :
#   ansible ma_vm_ansible -b -m apt -a "name=tree state=present update_cache=true"
- name: Gérer un paquet
  hosts: ma_vm_ansible
  become: true
  tasks:
    - name: S'assurer que tree est installé
      ansible.builtin.apt:        # le module
        name: tree                # le paquet à gérer
        state: present            # l'état voulu : installé. Autres valeurs : absent, latest
        update_cache: true        # équivaut à apt update
```

Arguments du module `apt` (la liste complète : `ansible-doc ansible.builtin.apt`) :

| Argument | Rôle | Valeurs |
|---|---|---|
| `name` | le ou les paquets à gérer | un nom (`tree`) ou une liste de noms |
| `state` | l'état voulu | `present` : installé (valeur par défaut) · `absent` : désinstallé · `latest` : installé et à jour |
| `update_cache` | met à jour l'index des paquets avant d'agir, comme `apt update` | `true` ou `false` (par défaut) |

```bash
ansible-playbook notions/2-3-tache.yml
ansible-playbook notions/2-3-tache.yml
```

Observe : la 1ʳᵉ exécution affiche `changed` (`tree` n'est pas installé), la 2ᵉ affiche `ok`.

**Question** : le même playbook, lancé deux fois, donne deux résultats différents. Pourquoi ?

**À toi** : passe `state` à `absent`, lance le playbook deux fois, puis remets `present` et relance. Compare `changed` à chaque exécution.

### 2.4 Les facts

Au début de chaque play, Ansible collecte des **facts** : des informations sur la machine (système, mémoire, réseau). On les lit dans `ansible_facts`.

`notions/2-4-facts.yml`
```yaml
---
# Au début de chaque play, Ansible collecte des « facts » : des informations sur la machine.
# On les lit avec ansible_facts['nom'].
# Pour les lister : ansible ma_vm_ansible -m setup
- name: Lire des facts
  hosts: ma_vm_ansible
  tasks:
    - name: Afficher la distribution et sa version
      ansible.builtin.debug:
        msg: "{{ ansible_facts['distribution'] }} {{ ansible_facts['distribution_version'] }}"

    - name: Afficher la mémoire totale
      ansible.builtin.debug:
        msg: "{{ ansible_facts['memtotal_mb'] }} Mo"
```

```bash
ansible-playbook notions/2-4-facts.yml
ansible ma_vm_ansible -m setup -a "filter=ansible_memtotal*"      # la liste complète : sans -a
```

Dans `ansible_facts[...]`, le nom perd son préfixe : `ansible_memtotal_mb` s'écrit `ansible_facts['memtotal_mb']`.

**À toi** : affiche le nombre de processeurs de la VM et son adresse IP (fact `default_ipv4`, sous-clé `address`). Cherche les noms avec `ansible ma_vm_ansible -m setup -a "filter=ansible_processor*"` et `filter=ansible_default_ipv4`.

### 2.5 `assert`

`assert` vérifie des conditions. Si l'une est fausse, le playbook s'arrête en erreur. Associé aux facts, il refuse une machine inadaptée avant de la modifier.

`notions/2-5-assert.yml`
```yaml
---
# assert vérifie des conditions. Si l'une est fausse, le playbook s'arrête en erreur.
# Ici : refuser une machine qui n'est pas une Debian 13 ou plus.
- name: Précondition
  hosts: ma_vm_ansible
  tasks:
    - name: Vérifier la version de Debian
      ansible.builtin.assert:
        that:                                  # toutes les conditions doivent être vraies
          - ansible_facts['distribution'] == 'Debian'
          - ansible_facts['distribution_major_version'] | int >= 13   # | int : texte -> nombre
        fail_msg: Quadlet demande Debian 13 minimum
        success_msg: Debian 13 ou plus
```

```bash
ansible-playbook notions/2-5-assert.yml
```

**À toi** : remplace `13` par `99` pour provoquer l'échec et lire le message, puis remets `13`. Ajoute une condition : au moins 1000 Mo de mémoire.

### 2.6 `register`

`register` range le résultat d'une tâche dans une variable, réutilisable ensuite.

`notions/2-6-register.yml`
```yaml
---
# register range le résultat d'une tâche dans une variable, réutilisable ensuite.
- name: Garder un résultat
  hosts: ma_vm_ansible
  tasks:
    - name: Lire la version du noyau
      ansible.builtin.command: uname -r
      register: noyau             # le résultat est rangé dans la variable « noyau »
      changed_when: false         # lire ne modifie rien (sans cette ligne, command compte « changed »)

    - name: Afficher ce qui a été gardé
      ansible.builtin.debug:
        msg: "Noyau : {{ noyau.stdout }}"   # stdout = ce que la commande a affiché
```

```bash
ansible-playbook notions/2-6-register.yml
```

**À toi** : lis la version de Debian avec `cat /etc/debian_version`, range-la avec `register`, puis vérifie avec `assert` qu'elle commence par `13` (`variable.stdout.startswith('13')`).

### 2.7 Les tags

Un **tag** étiquette une tâche ou un play. On lance ensuite seulement les tâches étiquetées (`--tags`), ou on les exclut (`--skip-tags`).

`notions/2-7-tags.yml`
```yaml
---
# Un tag étiquette une tâche (ou un play entier).
# On peut ensuite ne lancer que les tâches étiquetées (--tags), ou les exclure (--skip-tags).
- name: Tâches étiquetées
  hosts: ma_vm_ansible
  tasks:
    - name: Préparation
      ansible.builtin.debug:
        msg: "Préparation"
      tags: preparation

    - name: Contrôle
      ansible.builtin.debug:
        msg: "Contrôle"
      tags: controles
```

```bash
ansible-playbook notions/2-7-tags.yml
ansible-playbook notions/2-7-tags.yml --tags controles
ansible-playbook notions/2-7-tags.yml --skip-tags controles
```

**À toi** : ajoute une troisième tâche portant le tag `controles`. Combien de tâches s'exécutent avec `--tags controles` ?

### 2.8 `--check` et `--diff`

`--check` simule l'exécution : Ansible annonce ce qui changerait, sans rien modifier. `--diff` y ajoute le contenu des fichiers modifiés.

`notions/2-8-check.yml`
```yaml
---
# Avec --check, Ansible simule : il annonce ce qui changerait, sans rien modifier.
# Avec --diff, il montre en plus le contenu des fichiers modifiés.
- name: Simulation
  hosts: ma_vm_ansible
  tasks:
    - name: Déposer un fichier
      ansible.builtin.copy:
        dest: /tmp/tp-check.txt
        content: "version 1\n"
```

```bash
ansible-playbook notions/2-8-check.yml --check --diff
ansible ma_vm_ansible -a "ls /tmp/tp-check.txt"            # le fichier n'existe pas
ansible-playbook notions/2-8-check.yml
```

**À toi** : passe le contenu à `version 2` et lance `--check --diff` : lis le diff. Vérifie sur la VM que le fichier n'a pas changé (`ansible ma_vm_ansible -a "cat /tmp/tp-check.txt"`), puis applique pour de vrai.

### 2.9 Le `PLAY RECAP`

À la fin de chaque exécution, Ansible affiche une ligne par machine : le `PLAY RECAP`. Ce playbook produit exprès chacun des compteurs.

`notions/2-9-recap.yml`
```yaml
---
# Ce playbook produit exprès chacun des compteurs du PLAY RECAP.
- name: Produire des compteurs
  hosts: ma_vm_ansible
  tasks:
    - name: Tâche sans effet (ok)
      ansible.builtin.debug:
        msg: "ok"

    - name: Déposer un fichier (changed la 1re fois, ok ensuite)
      ansible.builtin.copy:
        dest: /tmp/tp-recap.txt
        content: "recap\n"

    - name: Tâche dont la condition est fausse (skipped)
      ansible.builtin.debug:
        msg: "jamais affiché"
      when: false

    - name: Échec volontaire, seulement avec -e echec=true (failed)
      ansible.builtin.fail:
        msg: "Échec provoqué"
      when: echec | default(false) | bool
```

```bash
ansible-playbook notions/2-9-recap.yml
ansible-playbook notions/2-9-recap.yml
ansible-playbook notions/2-9-recap.yml -e echec=true
```

Résultats attendus, dans l'ordre : `ok=3 changed=1 skipped=2`, puis `ok=3 changed=0 skipped=2`, puis `failed=1`.

| Compteur | Sens |
|---|---|
| `ok` | tâches exécutées sans erreur, **y compris** celles qui ont modifié la VM. Ici 3 = 1 `Gathering Facts` (collecte automatique, un par play) + 2 tâches |
| `changed` | parmi les `ok`, celles qui ont **modifié** la VM |
| `unreachable` | machine injoignable (SSH, IP, clé) |
| `failed` | tâche en échec ; Ansible arrête le play pour cette machine |
| `skipped` | tâche non exécutée (condition `when` fausse, module sans effet en `--check`) |
| `rescued`, `ignored` | erreurs gérées par `rescue` ou `ignore_errors` : non utilisés dans ce TP |

Lecture : `failed` et `unreachable` doivent être à 0. `changed` indique si la VM a bougé. Un playbook rejoué sur une VM conforme donne `changed=0`.

**À toi** : écris un playbook de quatre tâches qui donne, à la première exécution, exactement `changed=1` et `skipped=1`. Annonce les trois compteurs `ok`, `changed` et `skipped` **avant** de le lancer, puis compare.

### 2.10 Assemblage : `site.yml`

Tu as maintenant toutes les notions. Assemble-les : le premier play vérifie la VM (2.4, 2.5) et installe Podman (2.3) ; le second contrôle le résultat (2.6) et porte un tag (2.7).

Crée `site.yml` et remplace les `???` (mêmes paramètres qu'en 2.3, pour le paquet `podman`) :

```yaml
---
- name: Déployer la pile web
  hosts: ma_vm_ansible
  become: true

  tasks:
    - name: Vérifier que la VM est une Debian 13 ou plus
      ansible.builtin.assert:
        that:
          - ansible_facts['distribution'] == 'Debian'
          - ansible_facts['distribution_major_version'] | int >= 13
        fail_msg: Quadlet demande Debian 13 minimum

    - name: Installer Podman
      ansible.builtin.apt:
        name: ???
        state: ???
        update_cache: true

- name: Contrôles
  hosts: ma_vm_ansible
  tags: controles
  tasks:
    - name: Lire la version de Podman
      ansible.builtin.command: podman --version
      register: podman_version
      changed_when: false

    - name: Vérifier que Podman est en version 5
      ansible.builtin.assert:
        that: "'version 5' in podman_version.stdout"
```

```bash
ansible-playbook site.yml --check --skip-tags controles    # simulation : sans les contrôles
ansible-playbook site.yml
ansible-playbook site.yml
ansible-playbook site.yml --tags controles                 # les contrôles seuls
```

La simulation exclut les contrôles : en `--check`, Podman n'est pas installé, donc `podman --version` ne peut pas s'exécuter.

**Contrôle**
- [ ] 1ʳᵉ exécution : `ok=6 changed=1 failed=0` (2 `Gathering Facts` + 4 tâches). 2ᵉ exécution : `changed=0`.
- [ ] La simulation `--check` annonce `changed=1` sans rien installer.
- [ ] `--tags controles` s'exécute sans erreur.

---

## Étape 3 : nginx en conteneur avec Quadlet (25 min)

**Notions** : modules `copy` et `systemd_service`, contrôles HTTP, dérive.

**Quadlet.** Tu déposes un fichier `xxx.container` dans `/etc/containers/systemd/`. Au `daemon-reload`, Quadlet en génère un service systemd `xxx.service`. systemd lance donc le conteneur, au démarrage de la VM comme après un plantage. Ansible n'a que trois choses à faire : déposer les fichiers, recharger systemd, démarrer le service.

Fichiers à déposer sur la VM :

`/etc/containers/systemd/tp.network` : un réseau Podman dédié, sur lequel les conteneurs se joignent par leur nom.
```ini
[Network]
```

`/etc/containers/systemd/web1.container`
```ini
[Container]
Image=docker.io/library/nginx:alpine
ContainerName=web1
Network=tp.network
PublishPort=8081:80

[Service]
Restart=always

[Install]
WantedBy=multi-user.target
```
`PublishPort=8081:80` : le port 8081 de la VM mène au port 80 du conteneur. `WantedBy` : démarrage au boot.

**À faire** : ajoute ces trois tâches au play « Déployer la pile web », à la suite.

| Tâche | Module | À chercher dans `ansible-doc` |
|---|---|---|
| Déposer `tp.network` | `copy` | `dest`, `content` |
| Déposer `web1.container` | `copy` | `dest`, `content` |
| Démarrer `web1.service` | `systemd_service` | `name`, `state`, `daemon_reload` |

`content: |` permet d'écrire un texte sur plusieurs lignes dans le YAML. Quadlet ne crée `web1.service` qu'après un `daemon-reload`, qui est un paramètre du module `systemd_service`.

**Contrôles** : ajoute ces cinq tâches au play « Contrôles ».

| Tâche | Module | À chercher |
|---|---|---|
| Lire l'état des services | `service_facts` | résultat dans `ansible_facts.services` |
| Vérifier que `web1.service` est `running` | `assert` | `ansible_facts.services['web1.service'].state` |
| Attendre que le port 8081 réponde | `wait_for` | `host`, `port`, `timeout` |
| Lire la page de nginx | `uri` | `url`, `return_content`, `register` |
| Vérifier que la page contient `nginx` | `assert` | `reponse.content` |

Les deux tâches `wait_for` et `uri` portent `delegate_to: localhost` : elles s'exécutent sur ton poste, comme le ferait un client, et non sur la VM. L'adresse de la VM est `"{{ ansible_host }}"` (entre guillemets : une valeur ne peut pas commencer par `{{` sans eux).

```bash
ansible-playbook site.yml        # 1re exécution : télécharge l'image, patiente
ansible-playbook site.yml
```

**Dérive.** Simule un incident : quelqu'un arrête le service à la main, puis laisse Ansible ramener la VM à l'état décrit.

```bash
ansible ma_vm_ansible -b -m systemd_service -a "name=web1.service state=stopped"
ansible-playbook site.yml --tags controles      # échoue : web1 est arrêté
ansible-playbook site.yml                       # répare
```

**Contrôle**
- [ ] 2ᵉ exécution : `changed=0`, contrôles OK.
- [ ] `ansible ma_vm_ansible -b -a "podman ps"` liste le conteneur `web1`.
- [ ] Après la dérive : les contrôles échouent, puis le playbook complet rétablit `web1` (`changed=1`).

**Question** : tu n'as jamais tapé `podman run`. Qui a lancé le conteneur ? (`ansible ma_vm_ansible -a "systemctl status web1.service"`)

---

## Étape 4 : Variables, templates, handler (25 min)

**Notions** : `vars`, `template`, `notify` et handler, `--diff`, `-e`.

**4.1 Une page générée.** Un *template* est un fichier contenant des `{{ variables }}` qu'Ansible remplace. Crée `templates/index.html.j2` :

```jinja
<h1>{{ message }} depuis {{ inventory_hostname }}</h1>
```

`inventory_hostname` est fournie par Ansible (ici `mavm`). `message` est à toi : déclare-la dans le play « Déployer la pile web », au même niveau que `tasks:`, avec `vars:`, valeur `Bonjour`.

À faire :
- Avant la tâche `web1.container`, ajoute deux tâches : créer le dossier `/opt/tp/web1` (`file` : `path`, `state`), puis générer `/opt/tp/web1/index.html` depuis le template (`template` : `src`, `dest`).
- Dans `web1.container`, ajoute sous `[Container]` : `Volume=/opt/tp/web1:/usr/share/nginx/html:ro`.
- Dans les contrôles, remplace le test « contient `nginx` » par « contient `inventory_hostname` » (`inventory_hostname in reponse.content`).

```bash
ansible-playbook site.yml --diff       # --diff affiche ce qui change dans les fichiers
```

Ansible annonce `changed` pour `web1.container`, et pourtant le contrôle **échoue** : la page est toujours « Welcome to nginx! ». Le fichier est à jour sur la VM, mais le conteneur tourne avec l'ancienne configuration, car Podman ne lit le fichier qu'au démarrage.

**4.2 Un handler.** Il faut redémarrer `web1.service` quand, et seulement quand, `web1.container` change. Un **handler** est une tâche rangée sous `handlers:` (même niveau que `tasks:`), exécutée en fin de play, uniquement si une tâche qui la *notifie* a fait `changed`.

- Écris le handler `Redémarrer web1` : module `systemd_service`, `state: restarted`, `daemon_reload: true`.
- Dans la tâche qui dépose `web1.container`, ajoute `notify: Redémarrer web1` (texte identique au `name` du handler).

Relance : rien ne se passe et le contrôle échoue toujours. Le fichier étant déjà à jour, aucune tâche n'a changé : le handler n'est pas notifié. Simule une dérive :

```bash
ansible ma_vm_ansible -b -m file -a "path=/etc/containers/systemd/web1.container state=absent"
ansible-playbook site.yml
```

**4.3 Une variable pour l'image.** Convertis `web1.container` en template :
- crée `templates/web1.container.j2` avec le contenu actuel du fichier, en remplaçant la valeur de `Image=` par `{{ nginx_image }}` ;
- déclare `nginx_image: docker.io/library/nginx:alpine` dans `vars:` ;
- remplace le module `copy` par `template` (`src: web1.container.j2`) dans la tâche, en gardant le `notify`.

```bash
ansible-playbook site.yml                                    # changed=0 : même résultat qu'avant
ansible-playbook site.yml --diff -e nginx_image=docker.io/library/nginx:stable-alpine
ansible ma_vm_ansible -b -a "podman ps"
ansible-playbook site.yml
```

`-e` surcharge une variable en ligne de commande, sans modifier les fichiers. La dernière commande, sans `-e`, ramène l'image déclarée dans le playbook.

**Contrôle**
- [ ] Après la dérive de 4.2 : `web1.container` recréé, le handler `Redémarrer web1` s'exécute, les contrôles passent, `curl http://IP_VM:8081` affiche « Bonjour depuis mavm ».
- [ ] 4.3, 1ʳᵉ commande : `changed=0`. Avec `-e` : une seule ligne change (`Image=`), le handler s'exécute, `podman ps` affiche `stable-alpine`.
- [ ] Une relance sans modification : `changed=0`, aucun handler.

**Question** : change `message` (`ansible-playbook site.yml -e message=Salut`). La page change-t-elle ? Le handler s'exécute-t-il ? Pourquoi ?

---

## Étape 5 : haproxy et isolation du réseau (30 min)

**Notions** : `loop`, réseau interne, autonomie.

**5.1 haproxy.** Les deux fichiers sont fournis, les tâches sont à écrire sans tableau.

`/opt/tp/haproxy/haproxy.cfg`
```
defaults
    mode http
    timeout connect 5s
    timeout client  30s
    timeout server  30s

frontend entree
    bind *:8080
    default_backend serveurs_web

backend serveurs_web
    server web1 web1:80 check
```
`web1:80` : le conteneur nommé `web1`, joignable par son nom sur le réseau Podman. haproxy écoute sur 8080 *dans* son conteneur : l'image s'exécute sans root et ne peut pas prendre le port 80.

`/etc/containers/systemd/haproxy.container`
```ini
[Unit]
Requires=web1.service
After=web1.service

[Container]
Image=docker.io/library/haproxy:lts-alpine
ContainerName=haproxy
Network=tp.network
PublishPort=80:8080
Volume=/opt/tp/haproxy/haproxy.cfg:/usr/local/etc/haproxy/haproxy.cfg:ro

[Service]
Restart=always

[Install]
WantedBy=multi-user.target
```
`Requires` et `After` : haproxy démarre après `web1` (il résout son nom au démarrage) et redémarre avec lui.

À faire :
1. déposer les deux fichiers (le dossier `/opt/tp/haproxy` est à créer) ;
2. démarrer `haproxy.service` ;
3. redémarrer haproxy, et lui seul, quand l'un de ses deux fichiers change ;
4. ajouter aux contrôles : `haproxy.service` est `running`, le port 80 répond, la page lue sur le port 80 contient `inventory_hostname`.

**5.2 Factoriser.** Tu as deux tâches `file` identiques à une valeur près. Remplace-les par une seule tâche avec `loop` :

```yaml
      loop:
        - /opt/tp/web1
        - /opt/tp/haproxy
```
et `path: "{{ item }}"`. Relance : `changed=0` prouve que la VM est dans le même état qu'avant.

**5.3 Isoler nginx.** Aujourd'hui, nginx est encore joignable directement sur le port 8081 et partage le réseau `tp` avec haproxy. L'objectif : tout le trafic vers nginx passe par haproxy.

Un réseau Quadlet avec `Internal=true` n'a pas de sortie vers l'extérieur et ne publie aucun port. Si nginx n'est que sur ce réseau, rien d'extérieur à la VM ne peut l'atteindre, et lui ne peut rien joindre au dehors. haproxy se rattache aux deux réseaux : `tp` (port publié) et `back` (nginx).

- Dépose `/etc/containers/systemd/back.network` :
  ```ini
  [Network]
  Internal=true
  ```
- `web1.container` : remplace `Network=tp.network` par `Network=back.network` et supprime `PublishPort=8081:80`.
- `haproxy.container` : ajoute une seconde ligne `Network=back.network`.
- Contrôles : supprime ceux du port 8081 (ils ne sont plus valables) et ajoute une tâche `wait_for` avec `state: stopped` qui vérifie que les ports 8080 et 8081 sont fermés (une seule tâche, `loop` sur les deux ports).

**5.4 Redémarrage de la VM.**

```bash
ansible-playbook site.yml
ansible ma_vm_ansible -b -m reboot
ansible-playbook site.yml --tags controles
```

**Contrôle final**
- [ ] `ansible-playbook site.yml` : `changed=0`, `failed=0`.
- [ ] `curl http://IP_VM` affiche « Bonjour depuis mavm » (via haproxy).
- [ ] `curl http://IP_VM:8081` est refusé. Seuls les ports 22 et 80 répondent depuis ton poste.
- [ ] Après le redémarrage de la VM, les contrôles passent : systemd a relancé les deux conteneurs.

---

## Défi : deux nginx derrière haproxy

Ajoute `web2` (page différente) et fais répartir haproxy entre `web1` et `web2`.

```bash
for i in 1 2 3 4 5 6; do curl -s http://IP_VM; done     # les deux noms apparaissent
```

Pistes : une variable `serveurs: [web1, web2]`, `loop: "{{ serveurs }}"` et `{{ item }}` dans `dest` et dans les templates ; un handler peut aussi avoir un `loop` ; dans `haproxy.cfg` et `haproxy.container`, une boucle Jinja `{% for s in serveurs %} … {% endfor %}`.

## Bilan : ce que chaque mécanisme garantit

| Mécanisme | Ce qu'il garantit | Vu à l'étape |
|---|---|---|
| Idempotence | relancer un playbook ne casse rien : `changed=0` | 2, 3 |
| Facts + `assert` | refuser une VM inadaptée avant de la modifier | 2 |
| `--check`, `--diff` | voir ce qui changerait avant d'appliquer | 2, 4 |
| `register` + `assert`, `uri`, `wait_for` | le playbook vérifie lui-même son résultat | 2 à 5 |
| Tags | rejouer les contrôles seuls | 2 à 5 |
| Dérive | l'état décrit est rétabli, quelle que soit la cause de l'écart | 3, 4 |
| Template + variable | une valeur à un seul endroit, surchargeable avec `-e` | 4 |
| Handler | redémarrer un service seulement si sa configuration a changé | 4 |
| `loop` | une tâche au lieu de plusieurs copies | 5 |

## Si ça coince

- `UNREACHABLE` : teste d'abord `ssh ton_compte@IP_VM` à la main. VM en NAT ?
- Erreur YAML : indentation (2 espaces, pas de tabulation), ou valeur commençant par `{{` sans guillemets.
- `Could not find the requested service` : `daemon_reload: true` manquant sur la tâche `systemd_service`.
- Port 80 déjà utilisé sur la VM : un autre service (par exemple le paquet `nginx`) l'occupe.
- Un paramètre de module ? `ansible-doc ansible.builtin.<module>`.

## Pour aller plus loin

Ranger le playbook en rôles (`ansible-galaxy role init`), mettre les variables dans `group_vars/`, chiffrer un secret avec `ansible-vault`, passer les conteneurs en rootless, vérifier le style avec `ansible-lint`.
