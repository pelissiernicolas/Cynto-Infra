Place the Wazuh Windows agent installer and both transform files here before running the playbook:

- wazuh-agent.msi
- wazuh-agent-clients.mst
- wazuh-agent-servers.mst

The playbook copies all three files to the domain controller share and creates/links two GPOs:

- clients -> OU=Postes,DC=cynto,DC=tech
- servers -> OU=Serveurs,DC=cynto,DC=tech

The GPO software package configuration is intentionally manual. In GPMC, edit each GPO and add the MSI under:

Computer Configuration -> Policies -> Software Settings -> Software Installation

Use the UNC MSI path and add the matching MST in the Modifications tab.
