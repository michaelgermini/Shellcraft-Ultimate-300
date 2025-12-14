# Chapitre 225 - Gestion de configuration avec Ansible

> "Ansible est une merveille d'architecture. Il est simple, puissant, et extensible. C'est la façon dont l'infrastructure devrait être gérée." - Michael DeHaan, créateur d'Ansible

## Introduction : L'automatisation de la configuration

Si Terraform provisionne l'infrastructure, Ansible en configure le comportement. Cette distinction fondamentale définit leur complémentarité parfaite : Terraform crée les ressources physiques, Ansible les transforme en systèmes opérationnels.

Dans ce chapitre, nous explorerons les principes d'Ansible, son architecture sans agent, les playbooks, les rôles, et les patterns de configuration moderne.

## Section 1 : Principes fondamentaux d'Ansible

### 1.1 Architecture sans agent

**Ansible vs autres outils :**
```bash
echo "=== COMPARAISON DES OUTILS DE CONFIGURATION ==="
echo ""
echo "Ansible (sans agent) :"
echo "  ✓ Aucune installation sur les cibles"
echo "  ✓ Communication SSH standard"
echo "  ✓ Idempotent par conception"
echo "  ✓ Syntaxe déclarative simple"
echo ""
echo "Puppet/Chef (avec agent) :"
echo "  ✗ Agent à installer sur chaque nœud"
echo "  ✗ Serveur maître requis"
echo "  ✗ Complexité de déploiement"
echo "  ✓ Puissant pour configurations complexes"
echo ""
echo "Shell scripts :"
echo "  ✓ Simple et direct"
echo "  ✗ Pas idempotent"
echo "  ✗ Difficile à maintenir"
echo "  ✗ Gestion d'erreurs limitée"
```

**Avantages d'Ansible :**
- **Simplicité** : Pas d'infrastructure complexe à gérer
- **Rapidité** : Déploiement immédiat sur nouveaux serveurs
- **Sécurité** : Utilise SSH standard et sudo
- **Flexibilité** : Modules pour toutes les tâches courantes
- **Idempotence** : Exécution sécurisée multiple sans effets secondaires

### 1.2 Installation et configuration

```bash
# Installation d'Ansible
# Ubuntu/Debian
sudo apt update
sudo apt install ansible

# CentOS/RHEL
sudo yum install epel-release
sudo yum install ansible

# macOS
brew install ansible

# Windows (via WSL ou Cygwin)
# Recommandé d'utiliser WSL avec Ubuntu

# Vérification
ansible --version

# Configuration de base
mkdir -p ~/.ansible
cat > ~/.ansible.cfg << 'EOF'
[defaults]
inventory = ./inventory
remote_user = ansible
host_key_checking = False
retry_files_enabled = False
gathering = smart
fact_caching = jsonfile
fact_caching_connection = ~/.ansible/facts_cache
fact_caching_timeout = 86400

[ssh_connection]
ssh_args = -o ControlMaster=auto -o ControlPersist=60s -o ControlPath=/tmp/ansible-ssh-%h-%p-%r
pipelining = True
EOF
```

## Section 2 : Inventaire et connexion aux hôtes

### 2.1 Fichiers d'inventaire

```ini
# inventory.ini - Inventaire statique simple
[webservers]
web01 ansible_host=192.168.1.10 ansible_user=ubuntu
web02 ansible_host=192.168.1.11 ansible_user=ubuntu
web03 ansible_host=192.168.1.12 ansible_user=ubuntu

[databases]
db01 ansible_host=192.168.1.20 ansible_user=centos
db02 ansible_host=192.168.1.21 ansible_user=centos

[loadbalancers]
lb01 ansible_host=192.168.1.30 ansible_user=ubuntu

[all:vars]
ansible_python_interpreter=/usr/bin/python3
ansible_ssh_private_key_file=~/.ssh/id_rsa
ansible_ssh_common_args='-o StrictHostKeyChecking=no'

[webservers:vars]
nginx_version=1.20.0
app_port=8080

[databases:vars]
mysql_version=8.0
db_name=myapp
```

```yaml
# inventory.yml - Inventaire dynamique en YAML
all:
  children:
    webservers:
      hosts:
        web01:
          ansible_host: 192.168.1.10
          ansible_user: ubuntu
          nginx_listen_port: 80
        web02:
          ansible_host: 192.168.1.11
          ansible_user: ubuntu
          nginx_listen_port: 80
        web03:
          ansible_host: 192.168.1.12
          ansible_user: ubuntu
          nginx_listen_port: 80
      vars:
        ansible_ssh_private_key_file: ~/.ssh/web_key
        nginx_version: "1.20.0"
        app_port: 8080
    
    databases:
      hosts:
        db01:
          ansible_host: 192.168.1.20
          ansible_user: centos
        db02:
          ansible_host: 192.168.1.21
          ansible_user: centos
      vars:
        ansible_ssh_private_key_file: ~/.ssh/db_key
        mysql_version: "8.0"
        db_name: myapp
    
    loadbalancers:
      hosts:
        lb01:
          ansible_host: 192.168.1.30
          ansible_user: ubuntu
      vars:
        ansible_ssh_private_key_file: ~/.ssh/lb_key
        haproxy_version: "2.4"

  vars:
    ansible_python_interpreter: /usr/bin/python3
    ansible_ssh_common_args: '-o StrictHostKeyChecking=no'
```

### 2.2 Scripts d'inventaire dynamique

```python
#!/usr/bin/env python3
# inventory_aws.py - Script d'inventaire AWS EC2
import boto3
import json
import sys

def get_instances():
    """Récupère les instances EC2 avec leurs informations"""
    ec2 = boto3.client('ec2')
    
    response = ec2.describe_instances(
        Filters=[
            {
                'Name': 'instance-state-name',
                'Values': ['running']
            },
            {
                'Name': 'tag:Environment',
                'Values': ['prod', 'staging', 'dev']
            }
        ]
    )
    
    inventory = {
        '_meta': {
            'hostvars': {}
        },
        'all': {
            'children': []
        }
    }
    
    groups = {}
    
    for reservation in response['Reservations']:
        for instance in reservation['Instances']:
            instance_id = instance['InstanceId']
            public_ip = instance.get('PublicIpAddress', '')
            private_ip = instance.get('PrivateIpAddress', '')
            
            # Récupération des tags
            tags = {tag['Key']: tag['Value'] for tag in instance.get('Tags', [])}
            environment = tags.get('Environment', 'unknown')
            role = tags.get('Role', 'unknown')
            
            # Création des groupes
            if environment not in groups:
                groups[environment] = []
            if role not in groups:
                groups[role] = []
                
            groups[environment].append(instance_id)
            groups[role].append(instance_id)
            
            # Variables d'hôte
            hostvars = {
                'ansible_host': public_ip or private_ip,
                'ansible_user': 'ubuntu',
                'instance_id': instance_id,
                'private_ip': private_ip,
                'public_ip': public_ip,
                'instance_type': instance['InstanceType'],
                'availability_zone': instance['Placement']['AvailabilityZone'],
                'environment': environment,
                'role': role
            }
            
            inventory['_meta']['hostvars'][instance_id] = hostvars
    
    # Ajout des groupes à l'inventaire
    for group_name, hosts in groups.items():
        inventory[group_name] = {
            'hosts': {host: {} for host in hosts}
        }
        inventory['all']['children'].append(group_name)
    
    return inventory

def main():
    """Point d'entrée principal"""
    if len(sys.argv) == 2 and sys.argv[1] == '--list':
        inventory = get_instances()
        print(json.dumps(inventory, indent=2))
    elif len(sys.argv) == 3 and sys.argv[1] == '--host':
        # Pour la compatibilité, retourne un dict vide
        print(json.dumps({}))
    else:
        print("Usage: {} --list or {} --host <hostname>".format(sys.argv[0], sys.argv[0])")
        sys.exit(1)

if __name__ == '__main__':
    main()
```

```python
#!/usr/bin/env python3
# inventory_docker.py - Inventaire pour conteneurs Docker
import docker
import json
import sys

def get_containers():
    """Récupère les informations des conteneurs Docker"""
    client = docker.from_env()
    
    inventory = {
        '_meta': {
            'hostvars': {}
        },
        'all': {
            'children': []
        }
    }
    
    groups = {}
    
    for container in client.containers.list():
        container_name = container.name
        container_info = container.attrs
        
        # Récupération des labels
        labels = container_info.get('Config', {}).get('Labels', {})
        environment = labels.get('environment', 'unknown')
        role = labels.get('role', 'unknown')
        
        # Création des groupes
        if environment not in groups:
            groups[environment] = []
        if role not in groups:
            groups[role] = []
            
        groups[environment].append(container_name)
        groups[role].append(container_name)
        
        # Variables d'hôte
        hostvars = {
            'ansible_host': container_name,
            'ansible_connection': 'docker',
            'container_id': container.id,
            'container_name': container_name,
            'image': container_info['Config']['Image'],
            'environment': environment,
            'role': role
        }
        
        inventory['_meta']['hostvars'][container_name] = hostvars
    
    # Ajout des groupes à l'inventaire
    for group_name, hosts in groups.items():
        inventory[group_name] = {
            'hosts': {host: {} for host in hosts}
        }
        inventory['all']['children'].append(group_name)
    
    return inventory

def main():
    """Point d'entrée principal"""
    if len(sys.argv) == 2 and sys.argv[1] == '--list':
        inventory = get_containers()
        print(json.dumps(inventory, indent=2))
    elif len(sys.argv) == 3 and sys.argv[1] == '--host':
        print(json.dumps({}))
    else:
        print("Usage: {} --list or {} --host <hostname>".format(sys.argv[0], sys.argv[0])")
        sys.exit(1)

if __name__ == '__main__':
    main()
```

### 2.3 Test de connectivité

```bash
# Test de connectivité basique
ansible all -i inventory.ini -m ping

# Test avec un hôte spécifique
ansible web01 -i inventory.ini -m ping

# Test d'un groupe
ansible webservers -i inventory.ini -m ping

# Test avec inventaire dynamique AWS
ansible all -i inventory_aws.py -m ping

# Test de conteneurs Docker
ansible all -i inventory_docker.py -m ping

# Collecte de facts système
ansible all -i inventory.ini -m setup | head -50

# Test de connexion SSH manuelle
ssh -i ~/.ssh/id_rsa ubuntu@192.168.1.10 'echo "Connexion SSH réussie"'

# Debug de connexion
ansible web01 -i inventory.ini -m ping -vvv

# Test de sudo
ansible web01 -i inventory.ini -m command -a 'whoami' -b

# Test avec become (sudo)
ansible web01 -i inventory.ini -m command -a 'whoami' --become
```

## Section 3 : Modules et tâches

### 3.1 Modules de base

```yaml
# playbook_basics.yml - Utilisation des modules de base
---
- name: Exemples d'utilisation des modules Ansible
  hosts: webservers
  become: yes
  gather_facts: yes
  
  tasks:
    # Module command - Exécution de commandes simples
    - name: Afficher la date
      command: date
      register: date_output
      
    - name: Afficher le résultat
      debug:
        msg: "Date du serveur: {{ date_output.stdout }}"
    
    # Module shell - Exécution avec shell
    - name: Vérifier l'espace disque
      shell: df -h | grep -v tmpfs
      register: disk_usage
      
    - name: Afficher l'utilisation disque
      debug:
        var: disk_usage.stdout_lines
    
    # Module copy - Copie de fichiers
    - name: Copier un fichier de configuration
      copy:
        src: nginx.conf
        dest: /etc/nginx/nginx.conf
        owner: root
        group: root
        mode: '0644'
        backup: yes
      
    # Module template - Utilisation de templates Jinja2
    - name: Configurer nginx avec template
      template:
        src: nginx.conf.j2
        dest: /etc/nginx/sites-available/default
        owner: root
        group: root
        mode: '0644'
      notify: restart nginx
    
    # Module file - Gestion des fichiers et répertoires
    - name: Créer un répertoire
      file:
        path: /opt/myapp
        state: directory
        owner: www-data
        group: www-data
        mode: '0755'
    
    # Module lineinfile - Modification de fichiers ligne par ligne
    - name: Ajouter une ligne dans /etc/hosts
      lineinfile:
        path: /etc/hosts
        line: '192.168.1.100    api.internal.com'
        state: present
    
    # Module replace - Remplacement de texte
    - name: Modifier la configuration PHP
      replace:
        path: /etc/php/8.0/apache2/php.ini
        regexp: '^memory_limit = .*'
        replace: 'memory_limit = 256M'
    
    # Module service - Gestion des services
    - name: S'assurer que nginx est démarré
      service:
        name: nginx
        state: started
        enabled: yes
    
    # Module package - Gestion des paquets
    - name: Installer des paquets
      package:
        name: 
          - curl
          - wget
          - vim
        state: present
        update_cache: yes
    
    # Module user - Gestion des utilisateurs
    - name: Créer un utilisateur
      user:
        name: deploy
        group: www-data
        groups: sudo
        shell: /bin/bash
        create_home: yes
        ssh_key_file: .ssh/id_rsa
    
    # Module get_url - Téléchargement de fichiers
    - name: Télécharger une archive
      get_url:
        url: https://example.com/app-v1.0.tar.gz
        dest: /tmp/app.tar.gz
        checksum: sha256:abc123...
    
    # Module unarchive - Extraction d'archives
    - name: Extraire l'application
      unarchive:
        src: /tmp/app.tar.gz
        dest: /opt/myapp
        remote_src: yes
        owner: www-data
        group: www-data
    
    # Module uri - Requêtes HTTP
    - name: Tester l'API
      uri:
        url: http://localhost:8080/health
        method: GET
        status_code: 200
      register: api_response
      
    - name: Afficher la réponse API
      debug:
        var: api_response.json

  handlers:
    - name: restart nginx
      service:
        name: nginx
        state: restarted
```

### 3.2 Modules avancés

```yaml
# playbook_advanced_modules.yml - Modules avancés
---
- name: Utilisation de modules avancés
  hosts: all
  become: yes
  
  tasks:
    # Module git - Gestion de dépôts Git
    - name: Cloner un dépôt
      git:
        repo: https://github.com/myorg/myapp.git
        dest: /opt/myapp
        version: main
        update: yes
        force: yes
    
    # Module pip - Gestion des paquets Python
    - name: Installer des paquets Python
      pip:
        name:
          - requests
          - flask
          - sqlalchemy
        state: present
        executable: pip3
    
    # Module docker_container - Gestion des conteneurs Docker
    - name: Lancer un conteneur web
      docker_container:
        name: web_app
        image: nginx:alpine
        ports:
          - "80:80"
        volumes:
          - /opt/myapp:/usr/share/nginx/html:ro
        restart_policy: always
        state: started
    
    # Module mysql_db - Gestion des bases de données MySQL
    - name: Créer une base de données
      mysql_db:
        name: myapp
        state: present
        login_unix_socket: /var/run/mysqld/mysqld.sock
    
    # Module mysql_user - Gestion des utilisateurs MySQL
    - name: Créer un utilisateur MySQL
      mysql_user:
        name: app_user
        password: "{{ mysql_app_password }}"
        priv: 'myapp.*:ALL'
        state: present
        host: localhost
    
    # Module postgresql_db - Gestion PostgreSQL
    - name: Créer une base PostgreSQL
      postgresql_db:
        name: myapp
        state: present
        owner: postgres
    
    # Module postgresql_user - Utilisateurs PostgreSQL
    - name: Créer un utilisateur PostgreSQL
      postgresql_user:
        name: app_user
        password: "{{ pg_app_password }}"
        role_attr_flags: CREATEDB,NOSUPERUSER
        state: present
    
    # Module systemd - Gestion des services systemd
    - name: Créer un service systemd
      systemd:
        name: myapp
        state: started
        enabled: yes
        daemon_reload: yes
    
    # Module cron - Gestion des tâches cron
    - name: Ajouter une tâche cron
      cron:
        name: "backup database"
        minute: "0"
        hour: "2"
        job: "/usr/local/bin/backup_db.sh"
        user: root
    
    # Module firewall - Gestion du firewall
    - name: Ouvrir le port 80
      firewalld:
        port: 80/tcp
        permanent: yes
        state: enabled
      when: ansible_distribution == 'CentOS'
    
    - name: Ouvrir le port 80 (Ubuntu)
      ufw:
        rule: allow
        port: '80'
        proto: tcp
      when: ansible_distribution == 'Ubuntu'
    
    # Module selinux - Gestion SELinux
    - name: Configurer SELinux
      selinux:
        policy: targeted
        state: enforcing
    
    # Module seboolean - Gestion des booléens SELinux
    - name: Activer httpd_can_network_connect
      seboolean:
        name: httpd_can_network_connect
        state: yes
        persistent: yes
    
    # Module sysctl - Configuration du kernel
    - name: Configurer sysctl
      sysctl:
        name: net.ipv4.ip_forward
        value: '1'
        state: present
        reload: yes
    
    # Module mount - Gestion des points de montage
    - name: Monter un volume EBS
      mount:
        path: /data
        src: /dev/xvdf
        fstype: ext4
        state: mounted
```

## Section 4 : Playbooks et rôles

### 4.1 Structure d'un playbook

```yaml
# deploy_webapp.yml - Playbook complet de déploiement
---
- name: Déploiement de l'application web
  hosts: webservers
  become: yes
  gather_facts: yes
  
  vars:
    app_name: mywebapp
    app_version: "1.2.0"
    app_port: 8080
    db_host: "{{ groups['databases'][0] }}"
    db_name: myapp
    db_user: app_user
    
  vars_files:
    - secrets.yml
    
  pre_tasks:
    - name: Mettre à jour le cache des paquets
      package:
        update_cache: yes
      when: ansible_distribution in ['Ubuntu', 'Debian']
      
    - name: Installer les prérequis
      package:
        name:
          - curl
          - wget
          - unzip
        state: present
  
  roles:
    - common
    - nginx
    - nodejs
    - { role: postgresql, when: deploy_db_locally | default(false) }
  
  tasks:
    - name: Créer le répertoire de l'application
      file:
        path: "/opt/{{ app_name }}"
        state: directory
        owner: www-data
        group: www-data
        mode: '0755'
    
    - name: Télécharger l'application
      get_url:
        url: "https://releases.example.com/{{ app_name }}-{{ app_version }}.tar.gz"
        dest: "/tmp/{{ app_name }}.tar.gz"
        checksum: "{{ app_checksum }}"
    
    - name: Extraire l'application
      unarchive:
        src: "/tmp/{{ app_name }}.tar.gz"
        dest: "/opt/{{ app_name }}"
        remote_src: yes
        owner: www-data
        group: www-data
    
    - name: Configurer l'application
      template:
        src: app_config.j2
        dest: "/opt/{{ app_name }}/config.json"
        owner: www-data
        group: www-data
        mode: '0644'
    
    - name: Installer les dépendances
      command: npm install
      args:
        chdir: "/opt/{{ app_name }}"
      become_user: www-data
    
    - name: Créer le fichier de service systemd
      template:
        src: app.service.j2
        dest: "/etc/systemd/system/{{ app_name }}.service"
      notify: restart app
    
    - name: Démarrer et activer le service
      systemd:
        name: "{{ app_name }}"
        state: started
        enabled: yes
        daemon_reload: yes
    
    - name: Attendre que l'application soit disponible
      wait_for:
        host: localhost
        port: "{{ app_port }}"
        timeout: 60
    
    - name: Vérifier la santé de l'application
      uri:
        url: "http://localhost:{{ app_port }}/health"
        method: GET
        status_code: 200
      register: health_check
      
    - name: Afficher le résultat du health check
      debug:
        msg: "Application {{ app_name }} déployée avec succès"
      when: health_check.status == 200
  
  post_tasks:
    - name: Nettoyer les fichiers temporaires
      file:
        path: "/tmp/{{ app_name }}.tar.gz"
        state: absent
    
    - name: Envoyer une notification de déploiement
      command: "curl -X POST -H 'Content-type: application/json' --data '{\"text\":\"Déploiement de {{ app_name }} v{{ app_version }} terminé\"}' {{ slack_webhook_url }}"
      when: slack_webhook_url is defined
  
  handlers:
    - name: restart app
      systemd:
        name: "{{ app_name }}"
        state: restarted
    
    - name: restart nginx
      service:
        name: nginx
        state: restarted

- name: Configuration de la base de données
  hosts: databases
  become: yes
  
  tasks:
    - name: S'assurer que PostgreSQL est installé
      package:
        name: postgresql
        state: present
    
    - name: Créer la base de données
      postgresql_db:
        name: "{{ db_name }}"
        state: present
        owner: postgres
    
    - name: Créer l'utilisateur de l'application
      postgresql_user:
        name: "{{ db_user }}"
        password: "{{ db_password }}"
        role_attr_flags: CREATEDB,NOSUPERUSER
        state: present
    
    - name: Exécuter les migrations
      command: "psql -U {{ db_user }} -d {{ db_name }} -f /opt/{{ app_name }}/migrations/init.sql"
      become_user: postgres
```

### 4.2 Création de rôles

```bash
# Structure d'un rôle Ansible
mkdir -p roles/common/{tasks,handlers,templates,vars,defaults,meta}
mkdir -p roles/nginx/{tasks,handlers,templates,vars,defaults,meta}
mkdir -p roles/nodejs/{tasks,handlers,templates,vars,defaults,meta}

tree roles/
roles/
├── common
│   ├── defaults
│   │   └── main.yml
│   ├── handlers
│   │   └── main.yml
│   ├── meta
│   │   └── main.yml
│   ├── tasks
│   │   └── main.yml
│   ├── templates
│   │   └── ntp.conf.j2
│   └── vars
│       └── main.yml
└── nginx
    ├── defaults
    │   └── main.yml
    ├── handlers
    │   └── main.yml
    ├── meta
    │   └── main.yml
    ├── tasks
    │   └── main.yml
    ├── templates
    │   └── nginx.conf.j2
    └── vars
        └── main.yml
```

```yaml
# roles/common/tasks/main.yml
---
- name: Installer les outils de base
  package:
    name:
      - curl
      - wget
      - vim
      - htop
      - tree
    state: present

- name: Configurer NTP
  template:
    src: ntp.conf.j2
    dest: /etc/ntp.conf
  notify: restart ntp

- name: S'assurer que NTP est démarré
  service:
    name: ntp
    state: started
    enabled: yes
```

```yaml
# roles/common/handlers/main.yml
---
- name: restart ntp
  service:
    name: ntp
    state: restarted
```

```yaml
# roles/common/defaults/main.yml
---
common_packages:
  - curl
  - wget
  - vim
  - htop

ntp_server: pool.ntp.org
```

```yaml
# roles/nginx/tasks/main.yml
---
- name: Installer nginx
  package:
    name: nginx
    state: present

- name: Configurer nginx
  template:
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf
  notify: restart nginx

- name: Créer les répertoires de sites
  file:
    path: "{{ item }}"
    state: directory
    owner: www-data
    group: www-data
  loop:
    - /etc/nginx/sites-available
    - /etc/nginx/sites-enabled

- name: Copier la configuration du site par défaut
  template:
    src: default.j2
    dest: /etc/nginx/sites-available/default
  notify: restart nginx

- name: Activer le site par défaut
  file:
    src: /etc/nginx/sites-available/default
    dest: /etc/nginx/sites-enabled/default
    state: link
  notify: restart nginx

- name: S'assurer que nginx est démarré
  service:
    name: nginx
    state: started
    enabled: yes
```

```yaml
# roles/nginx/defaults/main.yml
---
nginx_worker_processes: auto
nginx_worker_connections: 1024
nginx_listen_port: 80
nginx_server_name: "{{ ansible_hostname }}"
```

```yaml
# roles/nginx/templates/nginx.conf.j2
user www-data;
worker_processes {{ nginx_worker_processes }};
pid /run/nginx.pid;

events {
    worker_connections {{ nginx_worker_connections }};
    use epoll;
    multi_accept on;
}

http {
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    types_hash_max_size 2048;
    
    include /etc/nginx/mime.types;
    default_type application/octet-stream;
    
    access_log /var/log/nginx/access.log;
    error_log /var/log/nginx/error.log;
    
    gzip on;
    gzip_disable "msie6";
    
    include /etc/nginx/sites-enabled/*;
}
```

```yaml
# roles/nginx/meta/main.yml
---
galaxy_info:
  author: Votre Nom
  description: Rôle pour l'installation et configuration de nginx
  license: MIT
  min_ansible_version: 2.9
  
  platforms:
    - name: Ubuntu
      versions:
        - focal
        - bionic
    - name: CentOS
      versions:
        - 8
        - 7
  
  galaxy_tags:
    - web
    - nginx
    - http

dependencies:
  - common
```

## Section 5 : Patterns avancés et bonnes pratiques

### 5.1 Gestion des secrets

```yaml
# group_vars/all/secret.yml - Secrets chiffrés
---
vault_mysql_root_password: !vault |
  $ANSIBLE_VAULT;1.1;AES256
  66386439653...
  
vault_app_secret_key: !vault |
  $ANSIBLE_VAULT;1.1;AES256
  37326635393...

# Utilisation des secrets dans les playbooks
---
- name: Configuration de l'application
  hosts: webservers
  
  vars:
    app_secret_key: "{{ vault_app_secret_key }}"
    
  tasks:
    - name: Configurer l'application
      template:
        src: config.yml.j2
        dest: /etc/myapp/config.yml
      vars:
        secret_key: "{{ app_secret_key }}"
```

```bash
# Gestion du vault Ansible
# Chiffrer un fichier
ansible-vault encrypt group_vars/all/secret.yml

# Éditer un fichier chiffré
ansible-vault edit group_vars/all/secret.yml

# Afficher le contenu d'un fichier chiffré
ansible-vault view group_vars/all/secret.yml

# Changer le mot de passe du vault
ansible-vault rekey group_vars/all/secret.yml

# Exécution avec vault
ansible-playbook deploy.yml --ask-vault-pass
ansible-playbook deploy.yml --vault-password-file ~/.vault_pass
```

### 5.2 Tests et validation

```yaml
# tests/test_webservers.yml - Tests des serveurs web
---
- name: Tests des serveurs web
  hosts: webservers
  gather_facts: no
  
  tasks:
    - name: Tester la connectivité HTTP
      uri:
        url: "http://{{ ansible_host }}"
        method: GET
        status_code: 200
      register: http_response
      
    - name: Vérifier que nginx répond
      assert:
        that:
          - http_response.status == 200
          - "'nginx' in http_response.server"
        fail_msg: "Le serveur web ne répond pas correctement"
        success_msg: "Serveur web opérationnel"
    
    - name: Tester l'application
      uri:
        url: "http://{{ ansible_host }}/health"
        method: GET
        status_code: 200
      register: app_response
      
    - name: Vérifier que l'application fonctionne
      assert:
        that:
          - app_response.status == 200
          - app_response.json.status == "healthy"
        fail_msg: "L'application ne fonctionne pas correctement"
```

```python
# test/validate_config.py - Validation de configuration Python
#!/usr/bin/env python3

import yaml
import sys
import os

def validate_inventory():
    """Valide la structure de l'inventaire"""
    inventory_file = 'inventory.yml'
    
    if not os.path.exists(inventory_file):
        print(f"ERREUR: Fichier d'inventaire manquant: {inventory_file}")
        return False
    
    try:
        with open(inventory_file, 'r') as f:
            inventory = yaml.safe_load(f)
        
        # Vérifications de base
        if 'all' not in inventory:
            print("ERREUR: Section 'all' manquante dans l'inventaire")
            return False
        
        if 'children' not in inventory['all']:
            print("ERREUR: Section 'children' manquante")
            return False
        
        print("✓ Inventaire valide")
        return True
        
    except yaml.YAMLError as e:
        print(f"ERREUR: Erreur de syntaxe YAML: {e}")
        return False

def validate_playbooks():
    """Valide la syntaxe des playbooks"""
    playbook_files = ['deploy.yml', 'test.yml']
    
    for playbook_file in playbook_files:
        if not os.path.exists(playbook_file):
            continue
            
        try:
            with open(playbook_file, 'r') as f:
                yaml.safe_load(f)
            print(f"✓ Playbook valide: {playbook_file}")
        except yaml.YAMLError as e:
            print(f"ERREUR: Syntaxe YAML invalide dans {playbook_file}: {e}")
            return False
    
    return True

def main():
    """Fonction principale"""
    print("=== Validation de la configuration Ansible ===")
    
    results = []
    results.append(validate_inventory())
    results.append(validate_playbooks())
    
    if all(results):
        print("\n✓ Toutes les validations sont passées")
        sys.exit(0)
    else:
        print("\n✗ Des erreurs ont été détectées")
        sys.exit(1)

if __name__ == '__main__':
    main()
```

### 5.3 Automatisation et intégration CI/CD

```yaml
# .github/workflows/ansible-deploy.yml
name: Ansible Deploy

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  deploy:
    runs-on: ubuntu-latest
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v3
      
    - name: Setup Python
      uses: actions/setup-python@v4
      with:
        python-version: '3.9'
    
    - name: Install dependencies
      run: |
        python -m pip install --upgrade pip
        pip install ansible ansible-lint
    
    - name: Validate syntax
      run: |
        ansible-playbook --syntax-check deploy.yml
        python test/validate_config.py
    
    - name: Lint Ansible
      run: ansible-lint
    
    - name: Test locally
      run: |
        ansible-playbook -i inventory_local.yml test.yml
    
    - name: Deploy to staging
      if: github.ref == 'refs/heads/main'
      run: |
        echo "${{ secrets.VAULT_PASSWORD }}" > ~/.vault_pass
        ansible-playbook -i inventory_staging.yml --vault-password-file ~/.vault_pass deploy.yml
    
    - name: Run tests on staging
      if: github.ref == 'refs/heads/main'
      run: |
        ansible-playbook -i inventory_staging.yml test.yml
```

```bash
# Script de déploiement automatisé
#!/bin/bash
# deploy.sh - Script de déploiement Ansible automatisé

set -euo pipefail

# Configuration
ENVIRONMENT="${1:-staging}"
PLAYBOOK="${2:-deploy.yml}"
INVENTORY="inventory_${ENVIRONMENT}.yml"

# Fonction de logging
log() {
    echo "[$(date +%Y-%m-%d %H:%M:%S)] $*" >&2
}

# Fonction de validation
validate_deployment() {
    log "Validation de la configuration..."
    
    # Vérifier que les fichiers existent
    if [[ ! -f "$PLAYBOOK" ]]; then
        log "ERREUR: Playbook manquant: $PLAYBOOK"
        exit 1
    fi
    
    if [[ ! -f "$INVENTORY" ]]; then
        log "ERREUR: Inventaire manquant: $INVENTORY"
        exit 1
    fi
    
    # Validation de la syntaxe
    ansible-playbook --syntax-check -i "$INVENTORY" "$PLAYBOOK"
    
    log "Configuration validée"
}

# Fonction de sauvegarde
create_backup() {
    local backup_dir="backups/$(date +%Y%m%d_%H%M%S)"
    
    log "Création d'une sauvegarde dans $backup_dir"
    
    mkdir -p "$backup_dir"
    
    # Sauvegarder l'inventaire et les variables
    cp "$INVENTORY" "$backup_dir/"
    cp -r group_vars/ "$backup_dir/" 2>/dev/null || true
    cp -r host_vars/ "$backup_dir/" 2>/dev/null || true
    
    echo "$backup_dir"
}

# Fonction de déploiement
deploy() {
    log "Déploiement vers $ENVIRONMENT..."
    
    # Variables d'environnement pour Ansible
    export ANSIBLE_ROLES_PATH="./roles:./galaxy_roles"
    
    # Exécution du playbook
    ansible-playbook \
        -i "$INVENTORY" \
        "$PLAYBOOK" \
        --vault-password-file ~/.vault_pass \
        --extra-vars "environment=$ENVIRONMENT" \
        --extra-vars "deploy_timestamp=$(date +%s)"
    
    log "Déploiement terminé"
}

# Fonction de tests post-déploiement
run_tests() {
    log "Exécution des tests post-déploiement..."
    
    ansible-playbook \
        -i "$INVENTORY" \
        test.yml \
        --extra-vars "environment=$ENVIRONMENT"
    
    log "Tests terminés"
}

# Fonction de rollback
rollback() {
    local backup_dir="$1"
    
    log "Rollback vers $backup_dir"
    
    if [[ -d "$backup_dir" ]]; then
        cp "$backup_dir/$(basename "$INVENTORY")" .
        cp -r "$backup_dir/group_vars/" . 2>/dev/null || true
        cp -r "$backup_dir/host_vars/" . 2>/dev/null || true
        
        log "Rollback terminé"
    else
        log "ERREUR: Répertoire de sauvegarde introuvable: $backup_dir"
        exit 1
    fi
}

# Fonction principale
main() {
    log "=== DÉPLOIEMENT ANSIBLE ==="
    log "Environnement: $ENVIRONMENT"
    log "Playbook: $PLAYBOOK"
    log "Inventaire: $INVENTORY"
    
    validate_deployment
    
    local backup_dir
    backup_dir=$(create_backup)
    
    # Gestion des erreurs avec trap
    trap 'log "ERREUR: Échec du déploiement"; rollback "$backup_dir"; exit 1' ERR
    
    deploy
    run_tests
    
    log "Déploiement réussi"
    
    # Nettoyer les anciennes sauvegardes (garder les 10 dernières)
    ls -dt backups/*/ | tail -n +11 | xargs -r rm -rf
}

# Exécution
main "$@"
```

## Conclusion : Ansible comme langage d'automatisation

Ansible transforme l'administration système en programmation déclarative, permettant de gérer des milliers de serveurs avec la même facilité qu'un seul. Son architecture sans agent et sa syntaxe intuitive en font l'outil idéal pour l'automatisation moderne.

Dans le prochain chapitre, nous explorerons les pipelines CI/CD complets, intégrant Terraform, Ansible, et d'autres outils dans des workflows de déploiement automatisé.

---

**Exercice pratique :** Créez une infrastructure complète avec Terraform et Ansible qui inclut :
1. Provisionnement d'un cluster Kubernetes avec Terraform
2. Configuration des nœuds avec Ansible (Docker, Kubernetes)
3. Déploiement d'une application multi-tiers
4. Configuration du monitoring et logging
5. Tests automatisés et rollback

**Challenge avancé :** Développez un framework d'automatisation multi-cloud qui :
- Supporte AWS, Azure et GCP simultanément
- Implémente des patterns d'orchestration avancés
- Gère les secrets et configurations sensibles
- Fournit des rapports de conformité automatiques
- S'intègre dans des pipelines GitOps avec monitoring temps réel

**Réflexion :** Comment Ansible change-t-il fondamentalement l'approche de l'administration système, et quels sont les impacts sur la productivité des équipes d'exploitation et la fiabilité des systèmes ?

