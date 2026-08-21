# IaC-hybride

Depot Ansible du memoire - Mise en oeuvre et evaluation d'une approche
Infrastructure as Code pour le deploiement automatise et le durcissement
d'une infrastructure hybride Windows/Linux.

Consomme le lab (VMs Vagrant) via inventaire Ansible. Depot independant
de vagrant-lab (justification dans le memoire).

## Structure

- ansible/inventory/ - inventaire des hotes
- ansible/roles/ - roles Ansible
- ansible/playbooks/ - playbooks
- ansible/group_vars/ - variables par groupe
- ansible/keys/ - cles SSH locales (non versionnees)