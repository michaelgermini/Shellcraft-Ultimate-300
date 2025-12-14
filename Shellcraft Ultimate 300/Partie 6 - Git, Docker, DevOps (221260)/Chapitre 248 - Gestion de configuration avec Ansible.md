# Chapitre 248 - Gestion de configuration avec Ansible

## Table des matières
- [Introduction](#introduction)
- [Playbooks de base](#playbooks-de-base)
- [Roles et réutilisabilité](#roles-et-réutilisabilité)
- [Inventaires dynamiques](#inventaires-dynamiques)
- [Conclusion](#conclusion)

## Introduction

Ansible automatise la configuration et la gestion de serveurs de manière idempotente et déclarative.

## Playbooks de base

**Playbook simple** :
```yaml
# playbook.yml
---
- name: Configuration serveur web
  hosts: web_servers
  become: yes
  tasks:
    - name: Installer Apache
      apt:
        name: apache2
        state: present
        update_cache: yes
    
    - name: Démarrer Apache
      systemd:
        name: apache2
        state: started
        enabled: yes
    
    - name: Copier la configuration
      template:
        src: apache.conf.j2
        dest: /etc/apache2/apache2.conf
      notify: restart apache
  
  handlers:
    - name: restart apache
      systemd:
        name: apache2
        state: restarted
```

## Roles et réutilisabilité

**Structure de role** :
```yaml
# roles/web/tasks/main.yml
---
- name: Installer les dépendances
  apt:
    name: "{{ item }}"
    state: present
  loop: "{{ web_packages }}"

- name: Configurer le service
  template:
    src: service.conf.j2
    dest: /etc/web/service.conf

# Utilisation du role
- hosts: web_servers
  roles:
    - role: web
      vars:
        web_packages:
          - apache2
          - php
```

## Inventaires dynamiques

**Inventaire dynamique** :
```bash
#!/bin/bash
# Inventaire dynamique Ansible

# Inventaire depuis AWS
cat > inventory_aws.json << 'EOF'
{
    "_meta": {
        "hostvars": {}
    },
    "web": {
        "hosts": ["web1.example.com", "web2.example.com"]
    },
    "db": {
        "hosts": ["db1.example.com"]
    }
}
EOF

# Utiliser l'inventaire
ansible-playbook -i inventory_aws.json playbook.yml
```

## Conclusion

Ansible simplifie la gestion de configuration à grande échelle avec une syntaxe claire et des playbooks réutilisables.

