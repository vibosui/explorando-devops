\
# Atividade Guiada 01 — Registro de Empecilhos e Troubleshooting

> Registro dos problemas encontrados durante a preparação e execução da primeira atividade de DevOps, com foco no raciocínio de diagnóstico, causa, correção e validação.

## Contexto

Ambiente utilizado:

```text
[HOST]    Windows + PowerShell + Vagrant + VirtualBox
    |
    +--> [CONTROL] Ubuntu 22.04 — Ansible
    |
    +--> [NODE1]   Ubuntu 22.04 — Apache
    |
    +--> [NODE2]   Ubuntu 22.04 — Docker
```

Rede privada:

```text
CONTROL  192.168.56.10
NODE1    192.168.56.11
NODE2    192.168.56.12
```

Fluxo principal da atividade:

```text
HOST
 |
 +-- Vagrant / VirtualBox
       |
       +-- CONTROL
       |     |
       |     +-- Ansible --SSH--> NODE1
       |     |                     |
       |     |                     +-- Apache
       |     |
       |     +-- Ansible --SSH--> NODE2
       |                           |
       |                           +-- Docker
       |
       +-- NODE1/NODE2
```

---

# 1. Git: identidade do autor do commit

## Sintoma

Ao tentar criar um commit:

```text
Author identity unknown

*** Please tell me who you are.

Run

  git config --global user.email "[you@example.com]"
  git config --global user.name "Your Name"

fatal: unable to auto-detect email address
```

## O que estava acontecendo?

O Git precisava saber **quem estava criando o commit**.

O repositório estava funcionando e o arquivo estava preparado, mas o Git não tinha uma identidade configurada para associar ao commit.

O modelo mental importante é:

```text
arquivo alterado
      |
      v
git add
      |
      v
staging area
      |
      v
git commit
      |
      +--> precisa de identidade do autor
```

## Correção

Foi configurado o nome e e-mail do Git usando `git config`.

Depois disso, o commit funcionou.

## Validação

O histórico mostrou:

```text
9d134c9 (HEAD -> main, origin/main, origin/HEAD)
docs: atualizar informações sobre meu ambiente

65d168d docs: adicionar informações sobre meu ambiente
```

---

# 2. Git: risco de fazer push para o repositório do professor

## Problema conceitual

Inicialmente, o `origin` apontava para:

```text
https://github.com/tiagoheineck/explorando-devops.git
```

Ou seja, o repositório original do professor estava configurado como remoto de push.

Foi levantada a preocupação:

> "O `git push` vai afetar o repositório do professor?"

## Diagnóstico

O comando:

```powershell
git remote -v
```

mostrou:

```text
origin  https://github.com/tiagoheineck/explorando-devops.git (fetch)
origin  https://github.com/tiagoheineck/explorando-devops.git (push)
```

Isso significa que `origin` era usado tanto para buscar quanto para enviar.

## Correção

Foi criado um **fork próprio** e o `origin` foi alterado:

```powershell
git remote set-url origin https://github.com/vibosui/explorando-devops.git
```

Depois foi adicionado o repositório original como `upstream`:

```powershell
git remote add upstream https://github.com/tiagoheineck/explorando-devops.git
```

Resultado:

```text
origin
  fetch -> repositório próprio
  push  -> repositório próprio

upstream
  fetch -> repositório do professor
  push  -> repositório do professor
```

## Modelo mental

```text
                 upstream
             professor/original
                    ^
                    |
                  fetch
                    |
                 [local]
                    |
                  push
                    v
                  origin
                 seu fork
```

A principal lição foi:

> `origin` e `upstream` são apenas nomes convencionais para remotos; o importante é entender para qual URL cada um aponta.

---

# 3. Vagrant: timeout e resets durante o boot

## Sintoma

Durante `vagrant up` apareceram repetidamente:

```text
control: Warning: Connection reset. Retrying...
control: Warning: Connection aborted. Retrying...
control: Warning: Remote connection disconnect. Retrying...
```

## Primeira interpretação

O Vagrant estava tentando estabelecer a conexão SSH com a VM recém-iniciada.

Isso não significava necessariamente que a VM estivesse quebrada.

Durante o boot:

```text
VirtualBox inicia VM
       |
       v
Ubuntu inicia
       |
       v
sshd inicia
       |
       v
Vagrant consegue SSH
```

Enquanto o SSH ainda não estava pronto, o Vagrant tentava novamente.

## Evidência posterior

O processo continuou e apareceu:

```text
Vagrant insecure key detected.
Vagrant will automatically replace
this with a newly generated keypair...
```

e posteriormente:

```text
Machine booted and ready!
```

## Conclusão

Os resets eram parte do processo de espera pelo SSH durante o boot, embora o comportamento tenha parecido inicialmente um loop.

A VM acabou iniciando normalmente.

---

# 4. VirtualBox Guest Additions incompatíveis

Durante o `vagrant up` apareceu:

```text
The guest additions on this VM do not match the installed version of
VirtualBox!

Guest Additions Version: 6.0.0 r127566
VirtualBox Version: 7.2
```

## Significado

Existia uma diferença entre:

```text
Guest Additions dentro da VM
```

e:

```text
VirtualBox instalado no HOST
```

As Guest Additions são componentes instalados no guest que ajudam o VirtualBox a fornecer determinadas integrações, como pastas compartilhadas.

## Decisão

Não foi feita nenhuma alteração imediata porque o próprio Vagrant indicou que, em muitos casos, a diferença não impede o funcionamento.

E as pastas compartilhadas funcionaram:

```text
C:/Users/vinic/Documents/DevOps/explorando-devops
    =>
/vagrant
```

Portanto, naquele momento, não havia evidência de que a incompatibilidade estivesse causando o problema.

---

# 5. NODE2: dificuldade inicial para acesso SSH

## Sintoma

Em determinado momento houve dificuldade de conexão SSH com o `node2`.

Também foi observado que o acesso via terminal do Vagrant acabava funcionando.

Foi usado:

```powershell
Test-NetConnection 127.0.0.1 -Port 2201
```

Resultado:

```text
TcpTestSucceeded : True
```

## O que isso provou?

A porta TCP estava acessível no HOST:

```text
HOST
 |
 +--> 127.0.0.1:2201
             |
             v
          NODE2 SSH
```

Portanto, não era simplesmente um problema de porta fechada.

Também foi usado:

```powershell
vagrant ssh node2
```

e finalmente a sessão SSH funcionou:

```text
Welcome to Ubuntu 22.04.5 LTS
...
vagrant@ubuntu-jammy:~$
```

Posteriormente, o hostname foi corrigido após:

```powershell
vagrant reload node2
```

---

# 6. NODE2 não estava com hostname `node2`

## Sintoma

Após o primeiro provisionamento, o prompt mostrava:

```text
vagrant@ubuntu-jammy:~$
```

em vez de:

```text
vagrant@node2:~$
```

## Diagnóstico

O `Vagrantfile` continha:

```ruby
config.vm.define "node2" do |node2|
  node2.vm.hostname = "node2"
```

Mas o comando utilizado anteriormente:

```powershell
vagrant provision node2
```

executa novamente o provisionamento, não equivale a reaplicar toda a configuração da VM da mesma maneira que um reload.

## Correção

Foi executado:

```powershell
vagrant reload node2
```

Depois disso, o hostname foi aplicado corretamente.

## Conceito aprendido

Diferença entre:

```text
vagrant provision
    -> executa provisionadores

vagrant reload
    -> reinicia a VM e reaplica configurações da VM

vagrant ssh
    -> abre uma sessão SSH

vagrant halt
    -> desliga a VM

vagrant destroy
    -> remove a VM
```

---

# 7. Ansible: NODE2 ficou UNREACHABLE por chave SSH

Este foi o principal problema de troubleshooting da atividade.

## Sintoma

No `[CONTROL]`:

```bash
ansible all -m ping
```

retornou:

```text
node2 | UNREACHABLE! => {
    "changed": false,
    "msg": "Failed to connect to the host via ssh: vagrant@192.168.56.12: Permission denied (publickey,keyboard-interactive).",
    "unreachable": true
}

node1 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

## Primeira conclusão

Como `node1` funcionava:

```text
CONTROL
  |
  +--> NODE1  ✓
  |
  +--> NODE2  ✗
```

o Ansible em si estava funcionando.

O erro:

```text
Permission denied (publickey,keyboard-interactive)
```

indicava um problema de **autenticação SSH**.

Não era, naquele momento, a hipótese principal de problema de Ansible, rede ou instalação.

---

# 8. Diagnóstico por comparação das chaves

No `[CONTROL]` foi consultada a chave pública:

```bash
cat /home/vagrant/.ssh/id_ed25519_ansible.pub
```

Resultado:

```text
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIHLujB5vv6UnY7SS6ZEX7hiYnRSMGGTV5uQs4bNy5vyy vagrant@control
```

Também foi consultado:

```bash
cat /vagrant/ansible_control_key.pub
```

A chave correspondia à chave pública do CONTROL.

Depois, no `[NODE2]`:

```bash
cat /home/vagrant/.ssh/authorized_keys
```

retornou:

```text
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIMZyMM58PZ9dpgmQdb0rCJCNbimdhkFVxsPd5w3BpRVc vagrant
```

## Comparação

CONTROL:

```text
IHLujB5vv6UnY7SS6ZEX7hiYnRSMGGTV5uQs4bNy5vyy
```

NODE2:

```text
IMZyMM58PZ9dpgmQdb0rCJCNbimdhkFVxsPd5w3BpRVc
```

As chaves eram diferentes.

## Conclusão

O CONTROL estava tentando autenticar com uma chave que o NODE2 não tinha autorizado.

Modelo:

```text
CONTROL
private key A
    |
    v
public key A
    |
    X
    |
NODE2 authorized_keys
public key B
```

Por isso:

```text
Permission denied
```

---

# 9. Correção da chave SSH do NODE2

No `[NODE2]`, foi adicionada ao `authorized_keys` a chave pública correta compartilhada pelo Vagrant:

```bash
cat /vagrant/ansible_control_key.pub >> /home/vagrant/.ssh/authorized_keys
```

O operador `>>` foi importante porque ele **acrescenta** ao arquivo, sem apagar a chave existente.

Depois disso, no `[CONTROL]`:

```bash
ansible all -m ping
```

passou a funcionar para os dois nodes.

Resultado conceitual:

```text
node1 | SUCCESS | pong
node2 | SUCCESS | pong
```

## Lição principal

O diagnóstico seguiu camadas:

```text
Ansible
  |
  v
SSH
  |
  v
autenticação
  |
  v
chave privada do CONTROL
  |
  v
chave pública autorizada no NODE2
```

Em vez de reinstalar Ansible ou recriar as VMs, foi comparado o estado real.

---

# 10. Primeiro playbook Apache: execução bem-sucedida

Foi criado:

```text
~/playbooks/apache.yml
```

Conteúdo inicial:

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

A execução:

```bash
ansible-playbook ~/playbooks/apache.yml
```

produziu:

```text
PLAY RECAP
node1 : ok=5 changed=3 unreachable=0 failed=0
```

## Resultado

Apache foi instalado e configurado no NODE1.

A página foi validada com:

```bash
curl http://192.168.56.11
```

Resultado:

```html
<html>
  <body>
    <h1>Servidor provisionado pelo Ansible</h1>
    <p>node1 - Apache funcionando!</p>
  </body>
</html>
```

---

# 11. Entendimento de `ok` versus `changed`

Na primeira execução apareceram, por exemplo:

```text
TASK [Instalar o Apache]
changed: [node1]
```

Depois, em uma execução posterior:

```text
TASK [Instalar o Apache]
ok: [node1]
```

Isso demonstrou idempotência na prática.

Primeira execução:

```text
Apache não estava instalado
        |
        v
Ansible instala
        |
        v
changed
```

Execução posterior:

```text
Apache já está instalado
        |
        v
estado desejado já alcançado
        |
        v
ok
```

O mesmo princípio apareceu na criação do `index.html`.

---

# 12. Verificação do serviço Apache

No `[CONTROL]` foi executado:

```bash
ansible node1 -a "systemctl is-active apache2"
```

Resultado:

```text
node1 | CHANGED | rc=0 >>
active
```

## Observação

O `CHANGED` aqui não significa necessariamente que o Apache foi modificado.

Foi utilizado um comando remoto através do módulo de execução de comando, e esse tipo de execução pode ser reportado como `CHANGED` mesmo quando o comando apenas consulta o estado.

O dado importante foi:

```text
rc=0
active
```

Isso confirmou:

```text
systemd
   |
   v
apache2.service
   |
   v
active
```

---

# 13. Erro de indentação no YAML do handler

Ao adicionar o handler, o arquivo inicialmente ficou com:

```yaml
   handlers:
```

enquanto `tasks` estava em:

```yaml
  tasks:
```

## Problema

YAML usa indentação para definir estrutura.

O correto era:

```yaml
  tasks:
    ...

  handlers:
    ...
```

## Correção

Foi ajustado para:

```yaml
  handlers:
    - name: Reiniciar Apache
      service:
        name: apache2
        state: restarted
```

---

# 14. Uso do `--syntax-check`

Antes de executar novamente o playbook, foi utilizado:

```bash
ansible-playbook ~/playbooks/apache.yml --syntax-check
```

Resultado:

```text
playbook: /home/vagrant/playbooks/apache.yml
```

## Importância

O `--syntax-check` permitiu validar a estrutura do playbook sem executar as tasks.

Fluxo recomendado:

```text
editar
  |
  v
syntax-check
  |
  +-- erro --> corrigir
  |
  +-- OK
       |
       v
executar playbook
```

---

# 15. Handler não executado quando não houve mudança

Depois de adicionar:

```yaml
notify: Reiniciar Apache
```

à task `copy`, o playbook foi executado novamente.

Como o HTML já estava correto:

```text
TASK [Criar página HTML personalizada]
ok: [node1]
```

O handler não foi executado.

Isso confirmou a relação:

```text
task
 |
 +-- ok ------> não notifica
 |
 +-- changed -> notify -> handler
```

---

# 16. Handler executado após mudança real

O conteúdo foi alterado de:

```html
<h1>Servidor provisionado pelo Ansible</h1>
```

para:

```html
<h1>Servidor provisionado pelo Ansible - Aula 01</h1>
```

Depois:

```bash
ansible-playbook ~/playbooks/apache.yml
```

produziu:

```text
TASK [Criar página HTML personalizada]
changed: [node1]

RUNNING HANDLER [Reiniciar Apache]
changed: [node1]
```

## Raciocínio

```text
conteúdo atual
      !=
conteúdo desejado
      |
      v
copy → changed
      |
      v
notify
      |
      v
handler
      |
      v
reiniciar Apache
```

A nova página foi validada:

```bash
curl http://192.168.56.11
```

Resultado:

```html
<h1>Servidor provisionado pelo Ansible - Aula 01</h1>
```

---

# 17. Docker: primeiro container

Na Parte 2, o Docker foi instalado/configurado no NODE2 pelo playbook do roteiro.

Depois foi executado no `[NODE2]`:

```bash
docker run hello-world
```

Como a imagem ainda não existia localmente, o Docker fez o download da imagem.

Modelo observado:

```text
docker run hello-world
        |
        v
imagem não existe localmente
        |
        v
download do registry
        |
        v
imagem hello-world
        |
        v
criação do container
        |
        v
execução
```

O container `hello-world` executou corretamente.

## Conceito aprendido

Imagem:

```text
modelo/template
```

Container:

```text
instância criada a partir da imagem
```

Um container pode executar, terminar e ficar parado sem que isso represente erro.

---

# 18. Encerramento das VMs

Ao terminar a atividade, as VMs foram desligadas de forma segura com:

```powershell
vagrant halt
```

Isso é diferente de:

```powershell
vagrant destroy
```

que removeria as VMs.

Modelo:

```text
vagrant halt
    |
    v
VM desligada
    |
    v
disco/configuração preservados
```

Enquanto:

```text
vagrant destroy
    |
    v
VM removida
```

---

# Resumo dos principais problemas

| Problema | Causa | Diagnóstico | Correção |
|---|---|---|---|
| Git não criava commit | identidade não configurada | mensagem `Author identity unknown` | `git config` |
| Risco de push no professor | `origin` apontava para upstream | `git remote -v` | fork + `origin` próprio + `upstream` |
| Vagrant resetava SSH durante boot | SSH ainda não estava pronto | logs de retry | aguardar conclusão do boot |
| Guest Additions diferente | versão guest/host diferente | aviso do Vagrant | monitorar; não bloqueou o laboratório |
| NODE2 não tinha hostname correto | configuração não tinha sido reaplicada como esperado | `vagrant@ubuntu-jammy` | `vagrant reload node2` |
| Ansible não acessava NODE2 | chave pública errada em `authorized_keys` | comparar chaves | adicionar chave do CONTROL |
| YAML do handler inválido | indentação incorreta | inspeção do arquivo | alinhar `handlers` com `tasks` |
| Handler não executava | task não produzia `changed` | `copy: ok` | alterar conteúdo para demonstrar `notify` |
| Docker baixou imagem | imagem não existia localmente | saída do `docker run` | comportamento esperado |

---

# Modelo de troubleshooting utilizado

O padrão de investigação que apareceu diversas vezes foi:

```text
ERRO
  |
  v
O que exatamente significa?
  |
  v
Qual camada está falhando?
  |
  +--> Host?
  |
  +--> Virtualização?
  |
  +--> VM?
  |
  +--> Rede?
  |
  +--> SSH?
  |
  +--> Linux?
  |
  +--> Ansible?
  |
  +--> Serviço?
  |
  +--> Aplicação?
  |
  v
Teste específico
  |
  v
Resultado
  |
  v
Hipótese confirmada/refutada
  |
  v
Correção
  |
  v
Validação
```

O melhor exemplo foi:

```text
Ansible UNREACHABLE
        |
        v
Permission denied
        |
        v
SSH/authentication
        |
        v
comparar chaves
        |
        v
CONTROL != NODE2
        |
        v
authorized_keys corrigido
        |
        v
ansible all -m ping
        |
        v
SUCCESS
```

---

# Estado final da atividade

```text
[Vagrant / VirtualBox]                ✓

[CONTROL]
  Ubuntu                            ✓
  Ansible                           ✓
  inventário                        ✓
  SSH para NODE1                    ✓
  SSH para NODE2                    ✓

[NODE1]
  Apache                            ✓
  serviço ativo                     ✓
  página HTML                       ✓
  handler                           ✓
  HTTP                              ✓

[NODE2]
  Docker                            ✓
  docker run hello-world            ✓
```

A atividade também deixou uma base para a próxima evolução arquitetural:

```text
site.yml
   |
   +-- composição de playbooks
   |
   +-- grupos de hosts
   |
   +-- configuração base
   |
   +-- configuração específica por host
   |
   +-- perfis de acesso de usuários
```

> **Nota:** este documento registra os eventos, comandos e saídas observados durante a atividade. Algumas explicações de causa são inferências técnicas baseadas nos sintomas e nas correções realizadas; quando não houve prova direta da causa interna do Vagrant/VirtualBox, o texto evita tratá-la como certeza.
