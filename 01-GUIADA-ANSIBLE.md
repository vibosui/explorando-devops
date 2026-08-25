# Atividade Guiada 01 — Provisionamento com Ansible

## Objetivo

Utilizar o Ansible, a partir do nó de controle, para provisionar automaticamente:
- **node1** → Servidor web Apache
- **node2** → Docker Engine

---

## Pré-requisitos

- Ambiente Vagrant iniciado e provisionado (`vagrant up`)
- Comunicação SSH entre `control` e os nós funcionando

Verifique a conectividade antes de começar:

```bash
vagrant ssh control
ansible all -m ping
```

Saída esperada:
```
node1 | SUCCESS => { "ping": "pong" }
node2 | SUCCESS => { "ping": "pong" }
```

---

## Estrutura de arquivos

Dentro do nó de controle (`vagrant ssh control`), crie a seguinte estrutura:

```
~/
├── inventory          (já criado pelo Vagrantfile)
├── ansible.cfg        (já criado pelo Vagrantfile)
└── playbooks/
    ├── apache.yml
    └── docker.yml
```

```bash
mkdir -p ~/playbooks
```

---

## Parte 1 — Provisionando o Apache no node1

### 1.1 Crie o playbook

```bash
nano ~/playbooks/apache.yml
```

Cole o conteúdo abaixo:

```yaml
---
- name: Instalar e configurar Apache no node1
  hosts: node1
  become: true

  tasks:
    - name: Atualizar cache de pacotes
      apt:
        update_cache: true

    - name: Instalar o Apache
      apt:
        name: apache2
        state: present

    - name: Garantir que o Apache está iniciado e habilitado
      service:
        name: apache2
        state: started
        enabled: true

    - name: Criar página HTML personalizada
      copy:
        dest: /var/www/html/index.html
        content: |
          <html>
            <body>
              <h1>Servidor provisionado pelo Ansible</h1>
              <p>node1 - Apache funcionando!</p>
            </body>
          </html>
        owner: www-data
        group: www-data
        mode: "0644"
```

### 1.2 Execute o playbook

```bash
ansible-playbook ~/playbooks/apache.yml
```

### 1.3 Valide o resultado

```bash
curl http://192.168.56.11
```

Saída esperada: página HTML com a mensagem personalizada.

---

## Parte 2 — Provisionando o Docker no node2

### 2.1 Crie o playbook

```bash
nano ~/playbooks/docker.yml
```

Cole o conteúdo abaixo:

```yaml
---
- name: Instalar Docker no node2
  hosts: node2
  become: true

  tasks:
    - name: Atualizar cache de pacotes
      apt:
        update_cache: true

    - name: Instalar dependências do Docker
      apt:
        name:
          - ca-certificates
          - curl
          - gnupg
        state: present

    - name: Criar diretório para chaves APT
      file:
        path: /etc/apt/keyrings
        state: directory
        mode: "0755"

    - name: Adicionar chave GPG oficial do Docker
      shell: >
        curl -fsSL https://download.docker.com/linux/ubuntu/gpg |
        gpg --dearmor -o /etc/apt/keyrings/docker.gpg
      args:
        creates: /etc/apt/keyrings/docker.gpg

    - name: Adicionar repositório do Docker
      shell: >
        echo "deb [arch=amd64 signed-by=/etc/apt/keyrings/docker.gpg]
        https://download.docker.com/linux/ubuntu
        $(lsb_release -cs) stable" > /etc/apt/sources.list.d/docker.list
      args:
        creates: /etc/apt/sources.list.d/docker.list

    - name: Atualizar cache após adicionar repositório
      apt:
        update_cache: true

    - name: Instalar Docker Engine
      apt:
        name:
          - docker-ce
          - docker-ce-cli
          - containerd.io
        state: present

    - name: Iniciar e habilitar o serviço Docker
      service:
        name: docker
        state: started
        enabled: true

    - name: Adicionar usuário vagrant ao grupo docker
      user:
        name: vagrant
        groups: docker
        append: true
```

### 2.2 Execute o playbook

```bash
ansible-playbook ~/playbooks/docker.yml
```

### 2.3 Valide o resultado

```bash
ansible node2 -m command -a "docker --version"
ansible node2 -m command -a "systemctl is-active docker"
```

Ou acesse o node2 diretamente:

```bash
exit   # sair do control
vagrant ssh node2
docker run hello-world
```

---

## Desafio Extra

Após concluir as etapas guiadas, tente:

1. **Combinar os dois playbooks em um único arquivo** `site.yml` usando `import_playbook`.
2. **Criar um handler** no playbook do Apache que reinicie o serviço somente quando a página HTML for alterada.
3. **Adicionar uma task** no playbook do Docker que execute o container `nginx` mapeando a porta 80 do node2.

### Passo a passo para reproduzir o desafio

Dentro do nó de controle (`control`), crie os arquivos abaixo:

#### 1. Playbook do Apache com handler

```bash
nano ~/playbooks/apache.yml
```

```yaml
---
- name: Instalar e configurar Apache no node1
  hosts: node1
  become: true

  tasks:
    - name: Atualizar cache de pacotes
      apt:
        update_cache: true

    - name: Instalar o Apache
      apt:
        name: apache2
        state: present

    - name: Garantir que o Apache está iniciado e habilitado
      service:
        name: apache2
        state: started
        enabled: true

    - name: Criar página HTML personalizada
      copy:
        dest: /var/www/html/index.html
        content: |
          <html>
            <body>
              <h1>Servidor provisionado pelo Ansible</h1>
              <p>node1 - Apache funcionando!</p>
            </body>
          </html>
        owner: www-data
        group: www-data
        mode: "0644"
      notify: Reiniciar Apache

  handlers:
    - name: Reiniciar Apache
      service:
        name: apache2
        state: restarted
```

#### 2. Playbook do Docker com container nginx

```bash
nano ~/playbooks/docker.yml
```

```yaml
---
- name: Instalar Docker no node2
  hosts: node2
  become: true

  tasks:
    - name: Atualizar cache de pacotes
      apt:
        update_cache: true

    - name: Instalar dependências do Docker
      apt:
        name:
          - ca-certificates
          - curl
          - gnupg
        state: present

    - name: Criar diretório para chaves APT
      file:
        path: /etc/apt/keyrings
        state: directory
        mode: "0755"

    - name: Adicionar chave GPG oficial do Docker
      shell: >
        curl -fsSL https://download.docker.com/linux/ubuntu/gpg |
        gpg --dearmor -o /etc/apt/keyrings/docker.gpg
      args:
        creates: /etc/apt/keyrings/docker.gpg

    - name: Adicionar repositório do Docker
      shell: >
        echo "deb [arch=amd64 signed-by=/etc/apt/keyrings/docker.gpg]
        https://download.docker.com/linux/ubuntu
        $(lsb_release -cs) stable" > /etc/apt/sources.list.d/docker.list
      args:
        creates: /etc/apt/sources.list.d/docker.list

    - name: Atualizar cache após adicionar repositório
      apt:
        update_cache: true

    - name: Instalar Docker Engine
      apt:
        name:
          - docker-ce
          - docker-ce-cli
          - containerd.io
        state: present

    - name: Iniciar e habilitar o serviço Docker
      service:
        name: docker
        state: started
        enabled: true

    - name: Adicionar usuário vagrant ao grupo docker
      user:
        name: vagrant
        groups: docker
        append: true

    - name: Garantir que o container nginx esteja em execução
      shell: |
        if ! docker ps -a --format '{{ "{{.Names}}" }}' | grep -qx 'nginx-lab'; then
          docker run -d --name nginx-lab -p 80:80 nginx:latest
        fi
      args:
        executable: /bin/bash
```

#### 3. Arquivo principal com `import_playbook`

```bash
nano ~/playbooks/site.yml
```

```yaml
---
- import_playbook: apache.yml
- import_playbook: docker.yml
```

#### 4. Executar a solução completa

```bash
ansible-playbook ~/playbooks/site.yml
```

#### 5. Validar o resultado

```bash
curl http://192.168.56.11
ansible node2 -m command -a "docker ps --format '{{ '{{.Names}}' }}'"
```

> O primeiro comando deve retornar a página HTML do Apache e o segundo deve listar o container `nginx-lab` em execução no `node2`.

---

## Referências

- [Documentação oficial do Ansible](https://docs.ansible.com/)
- [Módulo `apt`](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/apt_module.html)
- [Módulo `service`](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/service_module.html)
- [Módulo `copy`](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/copy_module.html)
- [Instalação do Docker no Ubuntu](https://docs.docker.com/engine/install/ubuntu/)
