# GPO - Activation Wazuh Agent

Cette GPO est créée vide par Ansible. L'administrateur doit la configurer manuellement dans la console **Gestion de stratégie de groupe** afin d'activer le service de l'agent Wazuh sur les postes/serveurs ciblés.

## GPO créée par Ansible

Nom : `GPO - Activation Wazuh Agent`

Liens par défaut :

- `OU=Postes,DC=cynto,DC=tech`
- `OU=Serveurs,DC=cynto,DC=tech`

## Objectif

Activer le service Windows de l'agent Wazuh pour éviter d'avoir à le faire manuellement sur chaque machine.

## Configuration manuelle dans GPMC

1. Ouvrir **Gestion de stratégie de groupe**.
2. Développer le domaine `cynto.tech`.
3. Aller dans **Objets de stratégie de groupe**.
4. Vérifier que la GPO `GPO - Activation Wazuh Agent` existe.
5. Clic droit sur la GPO, puis **Modifier**.
6. Aller dans :

   ```text
   Configuration ordinateur
   -> Préférences
   -> Paramètres du Panneau de configuration
   -> Services
   ```

7. Clic droit dans la zone vide, puis :

   ```text
   Nouveau
   -> Service
   ```

8. Configurer le service :

   ```text
   Action : Mettre à jour
   Nom du service : WazuhSvc
   Nom d'affichage : Wazuh Agent
   Type de démarrage : Automatique
   État du service : Démarrer le service
   ```

9. Appliquer puis fermer l'éditeur GPO.
10. Sur une machine cible, lancer :

    ```cmd
    gpupdate /force
    ```

11. Redémarrer la machine si nécessaire.
12. Vérifier le service sur la machine cible :

    ```cmd
    sc query WazuhSvc
    ```

Le service doit être en état `RUNNING`.

## Remarque

Cette GPO ne déploie pas le MSI. Le déploiement MSI/MST reste dans les GPO séparées :

- `GPO - Installation Wazuh Agent - Clients`
- `GPO - Installation Wazuh Agent - Serveurs`
