# Corrigé et notes formateur

Ne pas distribuer ce dossier avant la fin du TP.

| Fichier | Contenu |
|---|---|
| `site.yml` | État final après l'étape 5 (commentaires « étape N » = quand chaque bloc apparaît) |
| `notions/` | Les 9 playbooks indépendants de l'étape 2 (2.1 à 2.9), identiques à ceux du TP. À lancer depuis `corrige/` : `ansible-playbook notions/2-1-playbook.yml` |
| `templates/` | `index.html.j2` (étape 4.1), `web1.container.j2` (étape 4.3) |
| `defi/` | Corrigé du défi (2 nginx, `loop`) : `ansible-playbook defi/site.yml` |
| `reset.yml` | Remet la VM dans l'état d'avant le TP : conteneurs, unités Quadlet, réseaux, images, `/opt/tp`, paquet `podman` |
| `inventory.yaml`, `ansible.cfg` | Pointent vers la VM de test |

Lancer depuis ce dossier : `cd corrige && ansible-playbook site.yml`.

## État de validation

Le TP a été joué étape par étape sur une VM Debian 13.7 (VirtualBox, Podman 5.4.2), dans un dossier vierge, en n'utilisant que le texte du TP.

**Vérifié sur la VM**

- Étapes 1 à 5 : toutes les sorties annoncées dans « Contrôle » ont été observées (`changed=1` puis `0`, simulation `--check`, dérive et réparation, handler non déclenché sur fichier déjà à jour puis déclenché après dérive, `-e` sur l'image, boucle `changed=0`, isolation réseau).
- `site.yml` depuis une VM remise à zéro (`reset.yml`) : 35 s, 2ᵉ exécution `changed=0`.
- Vu du poste : seuls les ports 22 et 80 répondent. Dans le réseau `back`, nginx ne sort pas sur Internet et haproxy l'atteint par son nom.
- Défi : `changed=0` à la 2ᵉ exécution, contrôles OK. haproxy ne répartit pas strictement à parts égales (`web1, web2, web1, web1` puis 4 × web1 et 2 × web2 sur 6 requêtes) : le contrôle vérifie que les deux serveurs répondent, pas qu'ils alternent.
- Redémarrage de la VM : la pile revient seule (29 s), contrôles OK. Après un arrêt brutal de la VM (voir ci-dessous), les trois conteneurs ont aussi redémarré seuls au boot.
- `reset.yml` : la VM est propre après exécution (plus de Podman, rien sur les ports 80, 8080, 8081, `/opt/tp` supprimé).

**Non vérifié**

- Un second redémarrage (`ansible -m reboot`), effectué sur la pile du défi, a fait passer la VM VirtualBox en `gurumeditation` (plantage de l'hyperviseur). Cause non identifiée, non reproduite ; le premier redémarrage s'était déroulé normalement. Si cela se produit en séance, c'est un incident de la VM, pas du playbook.
- Ubuntu 24.04 (Podman 4.9) : non testé.

Les durées sont une estimation : seul le temps machine a été mesuré (12 à 35 s par exécution). Estimation par étape : 1 : 15 min · 2 : 45 · 3 : 25 · 4 : 25 · 5 : 30, soit environ 2 h 20 avec le défi en plus. Pour raccourcir : l'étape 4.3 (variable d'image), l'étape 5.2 (factorisation `loop`), ou la fusion des notions 2.1 et 2.2.

Les neuf playbooks de l'étape 2 et la solution de chacun des exercices ont été exécutés sur la VM. Le lint ne signale que l'absence de `mode:` sur les deux `copy` de 2.8 et 2.9 : voulu, pour garder ces exemples minimaux.

## Solutions des exercices de l'étape 2

- **2.1** : `msg: "{{ ansible_host }}"` affiche l'adresse IP de la VM.
- **2.2** : un play sur `all` ne s'exécute que sur `mavm`. Le `localhost` implicite n'est pas dans `all`.
- **2.3** : `state: absent` donne `changed=1`, puis `changed=0` à la relance ; en remettant `present`, `changed=1` (réinstallation).
- **2.4** : `{{ ansible_facts['processor_vcpus'] }}` et `{{ ansible_facts['default_ipv4']['address'] }}`. Piège : le nom du fact perd le préfixe `ansible_`.
- **2.5** : avec `99`, la tâche échoue avec `fail_msg`. Condition mémoire : `- ansible_facts['memtotal_mb'] >= 1000`.
- **2.6** : `command: cat /etc/debian_version` avec `register: version` et `changed_when: false`, puis `assert` avec `that: version.stdout.startswith('13')`.
- **2.7** : deux tâches s'exécutent avec `--tags controles`.
- **2.8** : le diff affiche `-version 1` et `+version 2`. Le fichier de la VM reste à `version 1` tant qu'on n'applique pas.
- **2.9** : exemple conforme : deux `debug` simples, un `copy` vers un nouveau chemin, un `debug` avec `when: false`. Compteurs attendus : `ok=4 changed=1 skipped=1`.

## Réponses aux questions du TP

- **2.3** : `state: present` décrit un état (« tree est installé »), pas une action. La 1ʳᵉ fois, `tree` manque : Ansible l'installe (`changed`). La 2ᵉ fois, l'état voulu est atteint : Ansible ne fait rien (`ok`). C'est l'idempotence. La question est posée en 2.3 et non à l'étape 1 : l'élève y dispose du sens de `state` et de l'observation.
- **Étape 3** : c'est systemd qui a lancé le conteneur, via le service généré par Quadlet (`systemctl status web1.service` montre le `podman run` de l'`ExecStart`).
- **Étape 4** : avec `-e message=Salut`, la page change tout de suite : seule la tâche `template` de la page est `changed`, et nginx relit le fichier monté à chaque requête. Le handler ne s'exécute pas : `web1.container` n'a pas changé, il n'est donc pas notifié.

## Prérequis VM

Debian 13 (Podman 5.4). Quadlet demande Podman ≥ 4.4 : Debian 12 (4.3) et Ubuntu 22.04 (3.4) ne conviennent pas. La VM doit être propre : un service déjà sur le port 80 (par exemple le paquet `nginx`) empêche haproxy de démarrer. `/etc/containers/systemd` est fourni par le paquet `podman` : aucune tâche `file` n'est nécessaire pour ce dossier.

Si la VM est installée avec `--no-install-recommends`, ajouter `aardvark-dns` (résolution des noms entre conteneurs) et `nftables` à la liste des paquets de l'étape 2.

## Choix de conception

- **Inventaire en YAML**, avec la structure commentée et l'arborescence : c'est le format des playbooks, une seule syntaxe à apprendre.
- **Contrôles écrits en Ansible** (second play, tag `controles`) : `register`, `assert`, `service_facts`, `uri`, `wait_for`. Ils remplacent les `curl` manuels et détectent la dérive. Le tag permet de les rejouer seuls et de les exclure en `--check` (en simulation, `command` n'est pas exécuté et les unités n'existent pas encore).
- **Facts + `assert` en précondition** : la VM est refusée avant toute modification si elle n'est pas en Debian ≥ 13.
- **Dégradation progressive de l'aide** : étape 2, trous à remplir ; étape 3, tableau module / paramètres ; étape 5, objectif seul.
- **Modules natifs uniquement** : pas de collection `containers.podman`. Le fichier Quadlet déposé avec `copy` montre ce qui se passe réellement, et il n'y a rien d'autre à installer.
- **Étape 4, le piège du handler** : le contrôle échoue parce que le conteneur n'a pas redémarré. Le handler n'est testable qu'en provoquant un écart (suppression du fichier), puisqu'une relance sur fichier à jour ne notifie rien. Cette dérive est la même démarche qu'à l'étape 3.
- **Isolation (étape 5.3)** : haproxy ne filtre que ce qui lui est adressé ; il ne peut pas empêcher un autre chemin vers nginx. Les garanties viennent de la configuration réseau : nginx ne publie aucun port et n'est que sur un réseau `Internal=true`, seul haproxy est rattaché à la fois à ce réseau et au réseau publié. Le pare-feu de l'hôte (nftables) n'est pas traité dans ce TP.
- **`web1` dès le départ** : le défi ajoute `web2` sans rien renommer ni supprimer.
- **haproxy écoute en 8080 dans son conteneur** : l'image tourne sous l'utilisateur `haproxy` et ne peut pas lier le port 80 ; le 80 est publié côté VM.
- **`Requires=web1.service`** : redémarrer `web1` redémarre aussi haproxy, qui doit re-résoudre le nom `web1` après la recréation du conteneur.
- **Pas de `enabled:` sur les services Quadlet** : ce sont des unités générées, le `[Install]` du fichier suffit.
- **`message` dans `vars` du play** : visible dans le play de déploiement uniquement ; les contrôles testent `inventory_hostname`, qui est global.

## Pièges attendus

| Symptôme | Cause | Réponse |
|---|---|---|
| Contrôles en échec en `--check` | `command` non exécuté, unités absentes | utiliser `--skip-tags controles` |
| `ansible.cfg` ignoré (avertissement) | dossier accessible en écriture à tous (dossier partagé de VM) | travailler dans le `$HOME` |
| `UNREACHABLE … No route to host` | VM en NAT, ou IP changée après un redémarrage (bail DHCP) | accès par pont ; vérifier l'IP avec `ip -br a` |
| `sudo: un mot de passe est requis` | prérequis sudo oublié | commande `sudoers.d` du TP, en console de la VM |
| 1ʳᵉ exécution de l'étape 3 longue | téléchargement de l'image nginx | patienter |
| `Could not find the requested service` | `daemon_reload: true` oublié | l'ajouter à la tâche systemd |
| Erreur de gabarit dans une commande ad hoc | Ansible interprète `{{ … }}` (par exemple `podman ps --format '{{.Name}}'`) | éviter ces motifs en ad hoc, ou les protéger avec `{% raw %}` |
| haproxy en échec, `could not resolve address 'web1'` | `aardvark-dns` absent, ou `web1` et haproxy sur des réseaux différents | vérifier les lignes `Network=` des deux fichiers |
| Handler jamais exécuté | texte de `notify` ≠ `name` du handler, ou fichier déjà à jour | comparer les textes ; provoquer l'écart |
| Étape 5.3 : `Timeout when waiting for …:8081` | anciens contrôles du port 8081 laissés en place | les supprimer, ajouter le contrôle `state: stopped` |

## Remise à zéro entre deux séances

```bash
cd corrige && ansible-playbook reset.yml
```

Le sudo sans mot de passe (`/etc/sudoers.d/tp`) n'est pas retiré.
