# Cynto-Infra — Infrastructure Ansible unifiée

Ce dépôt fusionne en un seul projet Ansible les quatre projets Cynto d'origine, afin de
déployer l'ensemble de l'infrastructure de manière cohérente, avec des **services bien
segmentés** et des **rôles partagés factorisés** :

| Service | Rôle dans l'infra | Cible |
|--------|--------------------|-------|
| **Active Directory** | Identité (forêt, DC, DNS, objets) | Windows / WinRM |
| **pfSense** | Pare-feu, VLAN, LDAP, OpenVPN | pfSense / SSH |
| **Nextcloud** | Plateforme collaborative (LDAPS) | Ubuntu / SSH |
| **GLPI** | Gestion de parc / ITSM (LDAPS) | Ubuntu / SSH |

Projets d'origine : `Cynto-Ansible-AD`, `Cynto-Ansible-pfsense`,
`Cynto-Ansible-nextcloud-ldaps`, `Cynto-Ansible-GLPI`.

---

## Arborescence

```
Cynto-Infra/
├── ansible.cfg                 # config unique (inventaire, roles_path multi-dossiers, ...)
├── requirements.yml            # collections (ansible.windows, microsoft.ad, pfsensible.core, ...)
├── site.yml                    # orchestration MAÎTRE (tous les services)
├── README.md
├── .gitignore
├── inventories/
│   └── prod/
│       ├── hosts.yml           # inventaire UNIFIÉ (tous les groupes)
│       ├── group_vars/
│       │   ├── all/            # variables globales + vault.yml(.example)
│       │   ├── windows/ domain_controllers / wsus_servers / member_servers
│       │   ├── ad/             # variables AD CS / LDAPS (consolidées)
│       │   ├── pfsense/        # base / dns / firewall / ldap / network / vpn
│       │   ├── nextcloud/      # serveur / nextcloud / php / ssl / appapi / ldap / db
│       │   └── glpi/           # serveur / apache / glpi / ldap / db
│       └── host_vars/          # srv-ad-01, srv-ad-02, srv-wsus-01
├── playbooks/
│   ├── ad.yml                  # service Active Directory
│   ├── ldaps-prep.yml          # ÉTAPE PARTAGÉE : AD CS + export CA (une seule fois)
│   ├── pfsense.yml             # service pfSense
│   ├── nextcloud.yml           # service Nextcloud
│   └── glpi.yml                # service GLPI
└── roles/                      # rôles RANGÉS PAR SERVICE (un sous-dossier par service)
    ├── ad/
    │   ├── ad_dns/  ad_domain_controller/  ad_forest/  ad_groups/  ad_ou/  ad_users/
    │   └── win_base/  win_network/
    ├── pfsense/
    │   ├── aliases/  base_config/  dns_resolver/  firewall_rules/  interfaces/  ldap_auth/
    │   ├── openvpn_client_export/  openvpn_firewall/  openvpn_server/  openvpn_users/
    │   └── pfsense_ad_ca/  vlans/
    ├── nextcloud/
    │   ├── nextcloud/  https_internal/  nextcloud_postconfig/  nextcloud_php_tuning/
    │   ├── nextcloud_appapi_daemon/  nextcloud_appapi_harp/  nextcloud_mail/
    │   └── nextcloud_ldaps/  nextcloud_access_policy/
    ├── glpi/
    │   ├── glpi/  apache_php/  apache_https/  glpi_postinstall/
    │   └── glpi_ldap/  glpi_ad_groups/  gpo_glpi_agent/
    └── shared/                 # rôles mutualisés entre plusieurs services
        ├── adcs_ldaps/  base_server/  mariadb/
```

> **Important — `roles_path`.** Comme les rôles sont rangés dans des sous-dossiers, Ansible ne
> les trouve plus avec le `roles_path` par défaut. `ansible.cfg` déclare donc explicitement
> chaque dossier de service :
>
> ```ini
> roles_path = roles/ad:roles/pfsense:roles/nextcloud:roles/glpi:roles/shared
> ```
>
> Les rôles restent référencés par leur **nom** dans les playbooks (ex. `glpi_ldap`,
> `nextcloud`, `mariadb`) ; les noms étant uniques, la résolution est sans ambiguïté.

---

## Segmentation des services

L'inventaire `inventories/prod/hosts.yml` regroupe tous les hôtes par fonction :

- `pfsense` — le pare-feu
- `windows`, `domain_controllers`, `primary_dc`, `secondary_dcs`, `wsus_servers`, `member_servers` — le périmètre Windows/AD
- `ad` — alias ciblant le DC principal pour les opérations AD CS / LDAPS
- `nextcloud` — le serveur Nextcloud
- `glpi` — le serveur GLPI

Chaque service possède son **playbook dédié** (`playbooks/<service>.yml`), son **dossier de
rôles** (`roles/<service>/`) et son **dossier de variables** (`group_vars/<service>/`).
Le `site.yml` enchaîne les services dans l'ordre des dépendances.

---

## Rôles partagés (factorisation)

Les projets d'origine dupliquaient certains rôles. Ils ont été **mutualisés** dans
`roles/shared/`, en une seule copie :

| Rôle partagé | Provenait de | Remarque |
|--------------|--------------|----------|
| `shared/adcs_ldaps` | pfSense, Nextcloud, GLPI (3 copies **identiques**) | Installe AD CS sur le DC, émet le certificat LDAPS, exporte la CA racine. Exécuté **une seule fois** via `playbooks/ldaps-prep.yml`. |
| `shared/base_server` | Nextcloud, GLPI (2 copies **identiques**) | Préparation de base des serveurs Ubuntu (hostname, netplan, timezone, MAJ). |
| `shared/mariadb` | GLPI (rôle dédié) + Nextcloud (installation embarquée) | Rôle générique : installe le moteur MariaDB et provisionne les bases/utilisateurs via la variable `mariadb_databases`. Utilisé par Nextcloud **et** GLPI. |

La récupération de la CA AD (`fetch` + conversion PEM), elle aussi dupliquée dans les trois
projets, a été centralisée dans `playbooks/ldaps-prep.yml`. Les artefacts sont déposés dans
`playbooks/artifacts/ad-ca/` (`.cer` et `.pem`) et consommés par `pfsense_ad_ca`,
`nextcloud_ldaps` et `glpi_ldap`.

---

## Prérequis

- Ansible (≥ 2.15 recommandé) sur la machine de contrôle
- Accès WinRM aux serveurs Windows et SSH aux serveurs Linux / pfSense
- `openssl` sur la machine de contrôle (conversion de la CA en PEM)

Installer les collections :

```bash
ansible-galaxy collection install -r requirements.yml
```

## Secrets (Ansible Vault)

```bash
cp inventories/prod/group_vars/all/vault.yml.example inventories/prod/group_vars/all/vault.yml
ansible-vault encrypt inventories/prod/group_vars/all/vault.yml
ansible-vault edit  inventories/prod/group_vars/all/vault.yml   # renseigner les mots de passe
```

Le fichier `vault.yml.example` liste **toutes** les variables `vault_*` nécessaires.

---

## Déploiement

Déploiement complet (ordre des dépendances géré par `site.yml`) :

```bash
ansible-playbook site.yml --ask-vault-pass --ask-pass --ask-become-pass
```

Déploiement service par service :

```bash
ansible-playbook playbooks/ad.yml          --ask-vault-pass
ansible-playbook playbooks/ldaps-prep.yml  --ask-vault-pass     # avant tout service LDAPS
ansible-playbook playbooks/pfsense.yml     --ask-vault-pass
ansible-playbook playbooks/nextcloud.yml   --ask-vault-pass --ask-pass --ask-become-pass
ansible-playbook playbooks/glpi.yml        --ask-vault-pass --ask-pass --ask-become-pass
```

> Note Ubuntu récent (sudo.ws) : si nécessaire, ajouter
> `--become-method=sudo -e "ansible_become_exe=sudo.ws"`.

---

## ⚠️ À vérifier avant déploiement

La fusion a nécessité d'**harmoniser des valeurs qui divergeaient** entre les projets d'origine.
Vérifiez qu'elles correspondent à votre environnement réel :

1. **Réseau des serveurs unifié sur `10.8.40.0/24`.**
   GLPI et le résolveur DNS pfSense utilisaient `192.168.1.x` ; ces références ont été
   réécrites en `10.8.40.x` (recherchez les commentaires `HARMONISATION FUSION`) :
   - `srv-glpi` → `10.8.40.13`
   - DC AD → `10.8.40.10` / `10.8.40.11`
2. **Adresses IP de l'inventaire** (`inventories/prod/hosts.yml`) :
   `fw-pfsense` (interface d'admin), `srv-wsus-01` (`10.8.40.16`), masques `/27` vs `/24`.
3. **Chemin d'export de la CA AD** : valeur retenue `C:\Temp\ca-export\` (GLPI utilisait
   `C:\Temp\CYNTO-CA\`). Voir `group_vars/ad/ldaps.yml`.
4. **Comptes de connexion** : `ansible_user` diffère (`cyntoadmin` pour Nextcloud,
   `nicolas` pour GLPI, `Administrateur` pour Windows).

## Notes de fusion

- Les rôles sont rangés par service sous `roles/<service>/` ; `roles_path` est adapté en
  conséquence dans `ansible.cfg`.
- Les `00-site.yml` d'origine étaient partiellement désynchronisés (étapes manquantes ou noms de
  fichiers obsolètes). Les playbooks par service ont été reconstruits proprement et incluent
  **tous** les rôles existants (y compris `nextcloud_mail` et `nextcloud_access_policy`).
- Des fichiers parasites `*:Zone.Identifier` (flux ADS Windows) présents dans GLPI ont été exclus.
- L'installation de MariaDB embarquée dans le rôle `nextcloud` a été retirée au profit du rôle
  partagé `shared/mariadb` (exécuté avant le rôle `nextcloud` dans `playbooks/nextcloud.yml`).
