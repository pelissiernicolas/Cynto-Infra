# Cynto-Infra — Infrastructure Ansible unifiée

Projet Ansible qui automatise le déploiement complet de l'infrastructure **cynto.tech** :
Active Directory, pare-feu pfSense, Nextcloud, GLPI et WSUS — le tout orchestré depuis un
dépôt unique, avec des rôles partagés factorisés et les secrets chiffrés via Ansible Vault.

| Service | Rôle dans l'infra | Cible |
|---|---|---|
| **Active Directory** | Identité (forêt, DC, DNS, OUs, groupes, GPO) | Windows Server / WinRM |
| **pfSense** | Pare-feu, VLANs, règles, OpenVPN, LDAP | pfSense / SSH |
| **Nextcloud** | Plateforme collaborative (LDAPS, AppAPI) | Ubuntu / SSH |
| **GLPI** | Gestion de parc / ITSM (LDAPS, agent GPO) | Ubuntu / SSH |
| **Zabbix** | Supervision (agents Linux/Windows, SNMP, API) | Ubuntu / SSH |
| **Switches Cisco SG500** | Configuration SNMP des commutateurs | SSH / SNMP |
| **WSUS** | Mises à jour Windows centralisées | Windows Server / WinRM |

> Ce dépôt est issu de la fusion de : `Cynto-Ansible-AD`, `Cynto-Ansible-pfsense`,
> `Cynto-Ansible-nextcloud-ldaps`, `Cynto-Ansible-GLPI`.

---

## Architecture réseau

```
                        Internet
                           │
                      ┌────┴────┐
                      │ pfSense │  fw-pfsense.cynto.tech
                      │  (WAN)  │  OpenVPN :1194/UDP
                      └────┬────┘
                           │ vtnet1 (trunk 802.1Q)
          ┌────────────────┼──────────────────────────────────┐
          │                │                │                  │
    VLAN10_ADMIN     VLAN20_USERS    VLAN40_SERVERS     VLAN99_MGMT
    10.8.10.0/28    10.8.20.0/26    10.8.40.0/27       10.8.99.0/28
                                         │
                    ┌────────────────────┼────────────────────┐
                    │                    │                     │
             srv-ad-01              srv-nextcloud          srv-glpi
             srv-ad-02              10.8.40.12             10.8.40.13
             10.8.40.10/11
             (DC + DNS + CA)
```

### VLANs

| ID | Nom | Passerelle | Plage | Masque |
|---|---|---|---|---|
| 10 | ADMIN | 10.8.10.1 | 10.8.10.0 | /28 |
| 20 | USERS | 10.8.20.1 | 10.8.20.0 | /26 |
| 30 | DEVELOPMENT | 10.8.30.1 | 10.8.30.0 | /27 |
| 40 | SERVERS | 10.8.40.1 | 10.8.40.0 | /27 |
| 60 | WiFi | 10.8.60.1 | 10.8.60.0 | /26 |
| 70 | IMPRESSION / IoT | 10.8.70.1 | 10.8.70.0 | /28 |
| 99 | MANAGEMENT | 10.8.99.1 | 10.8.99.0 | /28 |

### Hôtes

| Hostname | IP | Rôle |
|---|---|---|
| srv-ad-01 | 10.8.40.10 | Contrôleur de domaine principal, DNS, AD CS |
| srv-ad-02 | 10.8.40.11 | Contrôleur de domaine secondaire |
| srv-nextcloud | 10.8.40.12 | Nextcloud (Apache + MariaDB + Redis) |
| srv-glpi | 10.8.40.13 | GLPI (Apache + MariaDB) |
| srv-zabbix | 10.8.40.x | Zabbix Server + frontend (Apache + MariaDB) |
| srv-wsus-01 | 10.8.40.16 | WSUS |
| Proxmox / qdevice | — | Supervisés par Zabbix Agent 2 (agent only) |
| Switches Cisco SG500 | — | Configurés via SNMP, supervisés via Zabbix |

---

## Arborescence

```
Cynto-Infra/
├── ansible.cfg                 # Config unique (inventaire, roles_path, vault…)
├── requirements.yml            # Collections Galaxy requises
├── site.yml                    # Orchestration maître (tous les services)
├── inventories/
│   └── prod/
│       ├── hosts.yml           # Inventaire unifié (tous les groupes)
│       ├── group_vars/
│       │   ├── all/            # Variables globales : domain.yml, dns.yml, vault.yml
│       │   ├── windows/        # Connexion WinRM, réseau Windows
│       │   ├── ad/             # AD CS / LDAPS
│       │   ├── pfsense/        # base/ dns/ firewall/ ldap/ network/ vpn/
│       │   ├── nextcloud/      # application/ database/ ldap/ php/ server/ ssl/ appapi/
│       │   ├── glpi/           # application/ database/ ldap/ server/ ssl/ web/
│       │   └── zabbix/         # server/ api/ agents/ snmp/
│       └── host_vars/          # srv-ad-01, srv-ad-02, srv-wsus-01, srv-zabbix
├── playbooks/
│   ├── ad.yml                  # Service Active Directory
│   ├── ldaps-prep.yml          # Étape partagée : AD CS + export CA (une seule fois)
│   ├── pfsense.yml             # Service pfSense
│   ├── nextcloud.yml           # Service Nextcloud
│   ├── glpi.yml                # Service GLPI
│   ├── zabbix.yml              # Service Zabbix (serveur + agents Linux/Windows + API)
│   └── switches.yml            # Configuration SNMP des switches Cisco SG500
├── playbooks/artifacts/ad-ca/  # CA exportée à l'exécution (ignorée par Git)
│   ├── cynto-root-ca.cer       # Format DER (Windows)
│   └── cynto-root-ca.pem       # Format PEM (Linux / pfSense)
└── roles/
    ├── shared/                 # Rôles mutualisés entre plusieurs services
    │   ├── adcs_ldaps/         # Installe AD CS, émet le cert LDAPS, exporte la CA
    │   ├── base_server/        # Base Ubuntu : hostname, netplan, timezone, mises à jour
    │   └── mariadb/            # MariaDB générique (utilisé par Nextcloud et GLPI)
    ├── ad/
    │   ├── ad_forest/          # Création de la forêt AD
    │   ├── ad_domain_controller/  # Promotion DC secondaire
    │   ├── ad_dns/             # Zones DNS, enregistrements A/PTR
    │   ├── ad_ou/              # Structure d'OUs
    │   ├── ad_groups/          # Groupes de sécurité
    │   ├── win_base/           # Base Windows : WinRM, timezone, DNS
    │   └── win_network/        # Configuration réseau Windows
    ├── pfsense/
    │   ├── base_config/        # Config de base pfSense
    │   ├── vlans/              # Création des VLANs
    │   ├── interfaces/         # Affectation IP par interface
    │   ├── aliases/            # Alias IP / réseau
    │   ├── firewall_rules/     # Règles d'autorisation et de blocage
    │   ├── dns_resolver/       # Résolveur DNS Unbound
    │   ├── ldap_auth/          # Authentification LDAP via AD
    │   ├── pfsense_ad_ca/      # Import de la CA AD dans pfSense
    │   ├── openvpn_server/     # Serveur OpenVPN (TLS + auth AD)
    │   ├── openvpn_firewall/   # Règles firewall OpenVPN
    │   ├── openvpn_users/      # Comptes VPN
    │   └── openvpn_client_export/  # Export des profils clients
    ├── nextcloud/
    │   ├── nextcloud/          # Installation et configuration Nextcloud (occ)
    │   ├── https_internal/     # VHost Apache HTTPS interne
    │   ├── nextcloud_ldaps/    # Intégration LDAPS Active Directory
    │   ├── nextcloud_php_tuning/   # Tuning PHP (memory, upload…)
    │   ├── nextcloud_postconfig/   # Configuration post-install (mail, cron…)
    │   ├── nextcloud_mail/     # Configuration SMTP
    │   ├── nextcloud_access_policy/  # Politique d'accès
    │   ├── nextcloud_appapi_daemon/  # Démon AppAPI (Docker-less)
    │   └── nextcloud_appapi_harp/    # Proxy AppAPI HARP
    ├── glpi/
    │   ├── glpi/               # Installation GLPI
    │   ├── apache_php/         # Apache + PHP pour GLPI
    │   ├── apache_https/       # VHost Apache HTTPS
    │   ├── glpi_postinstall/   # Configuration post-install GLPI
    │   ├── glpi_ldap/          # Intégration LDAPS Active Directory
    │   ├── glpi_ad_groups/     # Synchronisation des groupes AD dans GLPI
    │   └── gpo_glpi_agent/     # Déploiement de l'agent GLPI via GPO
    ├── zabbix/
    │   ├── zabbix_server/      # Serveur Zabbix + frontend Apache (MariaDB, schéma, PHP)
    │   ├── zabbix_https/       # VHost Apache HTTPS pour Zabbix
    │   ├── zabbix_agent2/      # Agent Zabbix 2 pour Linux (Nextcloud, GLPI, Proxmox…)
    │   ├── zabbix_agent2_windows/  # Agent Zabbix 2 pour Windows (domain controllers)
    │   ├── zabbix_api_hosts/   # Enregistrement des hôtes via l'API REST Zabbix
    │   ├── zabbix_api_ldaps/   # Configuration LDAPS dans Zabbix via API
    │   ├── zabbix_api_mail/    # Configuration des alertes mail via API
    │   ├── zabbix_api_dashboard/   # Création des dashboards via API
    │   ├── zabbix_api_maps/    # Création des cartes réseau via API
    │   └── zabbix_snmp_test/   # Test SNMP de connectivité (switches)
    └── switches/
        └── sg500_snmp/         # Configuration SNMP des commutateurs Cisco SG500
```

> **`roles_path` multi-dossiers.** Les rôles étant rangés par sous-dossier de service,
> `ansible.cfg` déclare explicitement chaque chemin :
> ```ini
> roles_path = roles/ad:roles/pfsense:roles/nextcloud:roles/glpi:roles/shared
> ```

---

## Rôles partagés

Les projets d'origine dupliquaient plusieurs rôles. Ils ont été **mutualisés** dans `roles/shared/` :

| Rôle | Provenait de | Description |
|---|---|---|
| `shared/adcs_ldaps` | pfSense, Nextcloud, GLPI (3 copies identiques) | Installe AD CS, publie le template Kerberos, émet le cert LDAPS du DC, exporte la CA racine. Exécuté **une seule fois** via `playbooks/ldaps-prep.yml`. |
| `shared/base_server` | Nextcloud, GLPI (2 copies identiques) | Préparation Ubuntu : hostname, netplan, timezone, mises à jour. |
| `shared/mariadb` | Nextcloud (embarqué) + GLPI (dédié) | MariaDB générique, paramétré via `mariadb_databases`. Utilisé par Nextcloud **et** GLPI. |

La récupération + conversion PEM de la CA AD (également dupliquée) est centralisée dans
`playbooks/ldaps-prep.yml`. Les artefacts (`cynto-root-ca.cer` / `.pem`) sont déposés dans
`playbooks/artifacts/ad-ca/` et consommés par `pfsense_ad_ca`, `nextcloud_ldaps` et `glpi_ldap`.

---

## Prérequis

### Machine de contrôle

| Outil | Version minimale | Usage |
|---|---|---|
| Ansible | ≥ 2.15 | Exécution des playbooks |
| Python | ≥ 3.10 | Dépendance Ansible |
| openssl | toute version récente | Conversion DER → PEM de la CA AD |

### Accès réseau

| Protocole | Port | Cible | Usage |
|---|---|---|---|
| WinRM HTTP | 5985 | Serveurs Windows | Connexion Ansible → AD / WSUS |
| SSH | 22 | Ubuntu, pfSense | Connexion Ansible → Nextcloud / GLPI / pfSense |

### Collections Ansible Galaxy

```bash
ansible-galaxy collection install -r requirements.yml
```

Collections utilisées : `ansible.windows`, `microsoft.ad`, `community.mysql`,
`community.general`, `pfsensible.core`.

---

## Secrets (Ansible Vault)

Toutes les variables sensibles sont préfixées `vault_` et stockées dans
`inventories/prod/group_vars/all/vault.yml` (chiffré).

```bash
# Initialisation depuis l'exemple fourni
cp inventories/prod/group_vars/all/vault.yml.example \
   inventories/prod/group_vars/all/vault.yml

# Chiffrement
ansible-vault encrypt inventories/prod/group_vars/all/vault.yml

# Édition
ansible-vault edit inventories/prod/group_vars/all/vault.yml
```

### Variables Vault à renseigner

| Variable | Description |
|---|---|
| `vault_windows_admin_password` | Mot de passe `Administrateur` Windows |
| `vault_ad_safe_mode_password` | Mot de passe DSRM Active Directory |
| `vault_nextcloud_admin_user` | Compte admin Nextcloud |
| `vault_nextcloud_admin_password` | Mot de passe admin Nextcloud |
| `vault_nextcloud_db_password` | Mot de passe MariaDB Nextcloud |
| `vault_glpi_db_password` | Mot de passe MariaDB GLPI |
| `vault_pfsense_password` | Mot de passe admin pfSense |
| `vault_zabbix_db_password` | Mot de passe MariaDB Zabbix |
| `vault_zabbix_api_password` | Mot de passe du compte API Zabbix |
| `vault_snmp_community` | Communauté SNMP des switches |

---

## Déploiement

### Déploiement complet

```bash
ansible-playbook site.yml --ask-vault-pass
```

### Déploiement service par service

Les playbooks doivent être exécutés dans l'ordre des dépendances :

```bash
# 1. Active Directory (forêt, DNS, OUs, groupes)
ansible-playbook playbooks/ad.yml --ask-vault-pass

# 2. LDAPS — à exécuter UNE SEULE FOIS, avant tout service qui consomme l'AD
#    (génère playbooks/artifacts/ad-ca/cynto-root-ca.{cer,pem})
ansible-playbook playbooks/ldaps-prep.yml --ask-vault-pass

# 3. pfSense (VLANs, règles, OpenVPN, LDAP)
ansible-playbook playbooks/pfsense.yml --ask-vault-pass

# 4. Nextcloud
ansible-playbook playbooks/nextcloud.yml --ask-vault-pass --ask-pass --ask-become-pass

# 5. GLPI
ansible-playbook playbooks/glpi.yml --ask-vault-pass --ask-pass --ask-become-pass

# 6. Switches Cisco SG500 (SNMP)
ansible-playbook playbooks/switches.yml --ask-vault-pass

# 7. Zabbix (serveur en premier, puis agents et configuration API)
ansible-playbook playbooks/zabbix.yml --ask-vault-pass --ask-pass --ask-become-pass
```

### Exécution ciblée (tags / limit)

```bash
# Rejouer uniquement les règles firewall pfSense
ansible-playbook playbooks/pfsense.yml --ask-vault-pass --tags firewall_rules

# Rejouer sur un seul hôte
ansible-playbook playbooks/nextcloud.yml --ask-vault-pass --limit srv-nextcloud
```

> **Ubuntu récent (sudo.ws)** : si `become` échoue, ajouter
> `--become-method=sudo -e "ansible_become_exe=sudo.ws"`.

---

## ⚠️ Points de vigilance avant déploiement

La fusion des quatre projets d'origine a nécessité d'harmoniser des valeurs qui divergeaient.
Vérifiez qu'elles correspondent à votre environnement réel avant de lancer :

1. **Réseau des serveurs unifié sur `10.8.40.0/27`.**
   GLPI et le résolveur DNS pfSense utilisaient `192.168.1.x` ; ces références ont été
   réécrites en `10.8.40.x` (cherchez les commentaires `HARMONISATION FUSION`).

2. **Adresses IP de l'inventaire** (`inventories/prod/hosts.yml`) :
   vérifiez `fw-pfsense` (interface d'admin), `srv-wsus-01` (`10.8.40.16`).

3. **Chemin d'export de la CA AD** : valeur retenue `C:\Temp\ca-export\`
   (GLPI utilisait `C:\Temp\CYNTO-CA\`). Voir `group_vars/ad/ldaps.yml`.

4. **Comptes de connexion SSH** : harmonisés dans `group_vars/` par groupe.
   Vérifiez que `ansible_user` est cohérent avec vos serveurs réels.

---

## Dépannage

**`ldaps-prep.yml` échoue sur la conversion PEM**
→ Vérifier qu'`openssl` est installé sur la machine de contrôle et que
`playbooks/artifacts/ad-ca/cynto-root-ca.cer` a bien été récupéré à l'étape précédente.

**WinRM refusé sur les serveurs Windows**
→ S'assurer que WinRM est activé (`winrm quickconfig`) et que le pare-feu autorise le port 5985
depuis la machine de contrôle. Vérifier `ansible_winrm_transport: basic` dans
`group_vars/windows/connection.yml`.

**`pfsensible.core` introuvable**
→ Relancer `ansible-galaxy collection install -r requirements.yml`. La collection pfSensible
n'est pas incluse par défaut dans Ansible.

**Nextcloud déjà installé, le playbook re-joue l'installation**
→ Normal : la tâche est protégée par `when: "'installed: true' not in nextcloud_status.stdout"`.
Si le statut n'est pas détecté, vérifier que `occ` répond bien sur le serveur.

---

## Notes de fusion

- Les `00-site.yml` d'origine étaient partiellement désynchronisés (étapes manquantes ou noms
  de fichiers obsolètes). Les playbooks par service ont été reconstruits proprement et incluent
  tous les rôles existants (y compris `nextcloud_mail` et `nextcloud_access_policy`).
- L'installation MariaDB embarquée dans le rôle `nextcloud` a été extraite vers `shared/mariadb`.
- Les fichiers parasites `*:Zone.Identifier` (flux ADS Windows) sont exclus par `.gitignore`.
- Les artefacts de CA (`playbooks/artifacts/`) sont exclus de Git et régénérés à chaque exécution
  de `ldaps-prep.yml`.
