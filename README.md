Repo B (Infrastructure-as-Code)

This repository contains:
- Ansible playbooks used by the CD Jenkins pipeline to provision/configure AWS and deploy your Docker image.
- A CD Jenkinsfile that runs the playbook.

Important (edit these before running the CD pipeline):
- Update the AWS deployment variables in `playbooks/provision_and_deploy.yml` (or pass them via Jenkins extra-vars).
- Create the Jenkins credentials required by the `Jenkinsfile`.
- Create/encrypt an Ansible Vault file (placeholder provided in `vars/vault.yml`).

