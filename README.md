# IaC-hybride

Automatisation et durcissement d'un parc hétérogène (Ubuntu Server et Windows 11) avec Ansible.
Projet réalisé pour le mémoire de Master 2 de l'ENI Fianarantsoa : « Automatisation et orchestration de l'administration système : approche Infrastructure as Code pour la gestion et le durcissement de parcs informatiques hétérogènes ». Cas d'étude : Service Régional du Budget Haute Matsiatra.

- Dépôt (privé) : https://github.com/Luca-andoniaina/IaC-hybride
- Laboratoire (Vagrantfile, dépôt privé) : https://github.com/Luca-andoniaina/vagrant-lab
- Version du mémoire : tag `v1.0-memoire`

## 1. Ce que fait le projet

| Fonction | Playbook | Ubuntu | Windows 11 |
|---|---|---|---|
| Inventaire (un rapport Markdown par machine dans `reports/`) | `playbooks/inventory.yml` | oui | oui |
| Configuration de base | `playbooks/baseline.yml` | nom d'hôte, index apt, fuseau horaire, ntpsec, compte dédié (optionnel) | nom d'hôte, fuseau horaire, w32time, recherche de mises à jour et compte dédié (optionnels) |
| Durcissement CIS niveau 1 | `playbooks/hardening.yml` | `os_hardening` (devsec.hardening) | `ansible-lockdown` Windows-11-CIS (rôle `windows_11_cis`) |
| Pare-feu local (étiquette `iac_firewall`) | `playbooks/hardening.yml` | `ufw` | `win_firewall_rule` |

Les rôles du projet sont `linux_baseline`, `windows_baseline`, `linux_hardening` et `windows_hardening`. Les deux rôles de durcissement appellent le rôle tiers en dépendance (fichier `meta/main.yml`).
La mesure de conformité n'est pas un playbook : elle se fait avec Lynis (Ubuntu) et CIS-CAT Lite (Windows), voir §8.

## 2. Prérequis

Le code s'exécute depuis un nœud de contrôle Linux (Ubuntu Server 24.04). Ansible n'est pas installé sur l'hôte Windows.

| Composant | Version |
|---|---|
| Ansible | 14.5.0 (ansible-core 2.21.5) |
| Python | 3.12.3 |
| pywinrm | 0.5.0 |
| ansible-lint / yamllint | 26.10.1 / 1.38.0 |
| Collections et rôle Windows | `requirements.yml` (versions épinglées) |

Réseau du laboratoire : host-only 192.168.56.0/24 (ctrl 192.168.56.10, ubuntu 192.168.56.11, win11 192.168.56.12).

## 3. Installation sur le nœud de contrôle

```bash
# 1. Ansible dans un environnement isolé
pipx install --include-deps ansible==14.5.0
pipx inject ansible pywinrm==0.5.0 yamllint==1.38.0 ansible-lint==26.10.1

# 2. Dépôt
git clone git@github.com:Luca-andoniaina/IaC-hybride.git ~/IaC-hybride
cd ~/IaC-hybride

# 3. Collections et rôle Windows épinglés
ansible-galaxy collection install -r requirements.yml
ansible-galaxy role install -r requirements.yml
```

Si le nœud de contrôle n'a pas d'accès à GitHub, télécharger le rôle Windows-11-CIS (tag 3.0.2, commit dbf6a254) depuis un poste connecté et le copier dans `~/.ansible/roles/windows_11_cis`.

**Secrets.** Les mots de passe Windows sont dans `inventory/group_vars/windows/vault.yml`, chiffré avec Ansible Vault (variables `vault_windows_password` et `vault_ansible_account_password`). Le mot de passe du coffre est lu dans `~/.vault_pass`, hors du dépôt (ignoré par Git).
```bash
echo '<mot de passe du coffre>' > ~/.vault_pass && chmod 600 ~/.vault_pass
ansible-vault edit inventory/group_vars/windows/vault.yml
```

**Accès.** Linux : clé SSH ed25519 du nœud de contrôle (`ssh-copy-id vagrant@192.168.56.11`). Windows : WinRM en HTTPS sur le port 5986, authentification NTLM, certificat auto-signé.

## 4. Organisation du dépôt

```
IaC-hybride/
├── ansible.cfg            # inventaire, rôles, journal, coffre, pipelining SSH
├── requirements.yml       # collections et rôle tiers, versions épinglées
├── .yamllint, .ansible-lint
├── inventory/
│   ├── hosts.yml          # groupes linux et windows (sous-groupe windows11)
│   └── group_vars/
│       ├── all.yml        # variables communes de la configuration de base
│       ├── linux.yml      # connexion SSH
│       └── windows/       # main.yml (WinRM), hardening.yml, vault.yml (chiffré)
├── playbooks/             # inventory.yml, baseline.yml, hardening.yml
├── roles/                 # linux_baseline, windows_baseline, linux_hardening, windows_hardening
└── reports/               # rapports d'inventaire, journal ansible.log (non versionné)
```

Le dossier `.ansible/` (collections et rôle installés localement) est ignoré par Git. Les variables de groupe doivent rester dans `inventory/group_vars/` : Ansible ne les charge pas depuis la racine du dépôt.

## 5. Utilisation

Ordre d'une campagne, pour chaque cible : accès, configuration de base, mesure initiale, snapshot de la machine, durcissement, vérification du fonctionnement, seconde exécution (idempotence), mesure finale.

```bash
# Accès
ansible ubuntu -m ping
ansible win11  -m win_ping

# Inventaire et configuration de base
ansible-playbook playbooks/inventory.yml
ansible-playbook playbooks/baseline.yml

# Durcissement Ubuntu : simulation, puis application
ansible-playbook playbooks/hardening.yml --limit ubuntu --check --diff
ansible-playbook playbooks/hardening.yml --limit ubuntu

# Durcissement Windows 11 (niveau 1 seulement ; pas de mode simulation fiable : snapshot obligatoire avant)
ansible-playbook playbooks/hardening.yml --limit win11 \
  --skip-tags level2-high-security-sensitive-data-environment,level1-bitlocker,level2-bitlocker

# Pare-feu local seul
ansible-playbook playbooks/hardening.yml --tags iac_firewall --limit <machine>
```

Rejouer la même commande doit donner `changed=0` : c'est le test d'idempotence. Exception : `inventory.yml` écrit la date de collecte dans le rapport et signale donc toujours un changement.

## 6. Ajouter un hôte

1. Déclarer la machine dans `inventory/hosts.yml`, sous le groupe `linux` ou sous `windows` (sous-groupe `windows11` pour Windows 11), avec son `ansible_host`.
2. Linux : déposer la clé du nœud de contrôle avec `ssh-copy-id`.
   Windows : activer WinRM en HTTPS (port 5986) et, si le mot de passe diffère, l'ajouter au coffre.
3. Tester avec `ansible <machine> -m ping` (ou `win_ping`), puis lancer `inventory.yml` et `baseline.yml` avec `--limit <machine>`.
4. Pour un lot de postes : sous-groupes dans l'inventaire et directive `serial`.

## 7. Ajouter, modifier ou désactiver un contrôle de durcissement

- **Windows.** Chaque contrôle CIS a une variable `win11cis_rule_<numéro>` (rôle ansible-lockdown). Pour en désactiver un, la mettre à `false` dans `inventory/group_vars/windows/hardening.yml`, puis rejouer le durcissement. Ce fichier porte aussi `win_skip_for_test`, `skip_reboot`, les noms des comptes administrateur et invité, et le texte d'avertissement légal.
- **Ubuntu.** Les variables d'`os_hardening` sont fixées dans `roles/linux_hardening/meta/main.yml` (`os_security_kernel_enable_module_loading`, `os_auditd_enabled`). Les autres gardent leur valeur par défaut.
- **Pare-feu.** Tâches `ufw` dans `roles/linux_hardening/tasks/main.yml` et `win_firewall_rule` dans `roles/windows_hardening/tasks/main.yml`, étiquette `iac_firewall`. Les adresses autorisées sont les variables `linux_hardening_firewall_*` et `windows_hardening_firewall_*` de `defaults/main.yml`.

**Avertissement WinRM.** Le contrôle CIS 18.4.1 remet `LocalAccountTokenFilterPolicy` à 0 et coupe WinRM sur une machine sans domaine. Il est désactivé (`win11cis_rule_18_4_1: false`). Ne pas le réactiver sans canal d'accès de secours. Le mode `win_skip_for_test` exclut 17 autres contrôles qui risquent de couper l'accès à distance.

## 8. Mesurer la conformité

- **Ubuntu** : `sudo lynis audit system` sur la cible ; l'indice est `hardening_index` dans `/var/log/lynis-report.dat`.
- **Windows 11** : CIS-CAT Lite (Assessor v4.67.0, benchmark Windows 11 Enterprise v5.1.0), profil `xccdf_org.cisecurity.benchmarks_profile_Level_1_L1`, lancé dans la machine (commande exacte en annexe du mémoire).

Les scores de deux outils différents ne se comparent pas ; seul le gain avant/après sur un même système a un sens.

## 9. Contrôle qualité

```bash
yamllint .
ansible-playbook playbooks/<playbook>.yml --syntax-check
ANSIBLE_COLLECTIONS_PATH=$PWD/.ansible/collections:$HOME/.ansible/collections:$HOME/.local/share/pipx/venvs/ansible/lib/python3.12/site-packages \
  ansible-lint --offline
```
Dernier résultat : 0 échec, 0 avertissement, profil `production` (23 fichiers traités sur 33).

## 10. Traçabilité

- Journal d'Ansible : `reports/ansible.log` (paramètre `log_path`, fichier non versionné).
- Rapports d'inventaire : `reports/inventaire_<machine>.md`.
- Un commit par étape, message « type : description » (chore, feat, fix).
- Preuves de la campagne du mémoire (rapports CIS-CAT, journaux Ansible, empreintes) : dossier `evidence/` du laboratoire, hors dépôt.

## 11. Résultats de la campagne du mémoire

| Système | Mesure | Avant | Après |
|---|---|---|---|
| Ubuntu 24.04 | Lynis (indice) | 61 | 72 |
| Windows 11 Enterprise 22H2 | CIS-CAT Lite niveau 1 (benchmark v5.1.0) | 22,64 % | 87,06 % |

Idempotence : seconde exécution à `changed=0` pour la configuration de base, le durcissement et le pare-feu, sur les deux systèmes.
Flux testés après les règles locales : 5 sur 5 conformes.

## 12. Limites

- Une seule machine par système, réseau plat, pas de domaine Active Directory.
- WinRM en HTTPS avec certificat auto-signé, validation désactivée (laboratoire uniquement). Un parc réel demande un certificat d'une autorité interne.
- Ubuntu 24.04 dans le laboratoire, alors que le serveur du service tourne sous 26.04.
- Windows 11 22H2, hors support Microsoft ; le rôle suit le benchmark v3.0.0 alors que CIS-CAT évalue le v5.1.0.
- Windows 10 non traité. Le partage SMB n'est pas restreint.
- Le compte `vagrant` et son mot de passe par défaut sont propres au laboratoire et doivent être remplacés sur un parc réel.
