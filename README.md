Este projeto é um ambiente criado para o curso de Ciência da Computação do IFC Videira.

# Pré-requisitos:

- Virtual Box
- Vagrant



# Vagrant

## Subindo as máquinas virtuais

Abra o terminal na pasta do projeto e execute:

```
vagrant up #na segunda vez recomendasse usar --provision como parâmetro.
```

> Qual a diferença entre usar o parâmetro --provision


> Nota: Na primeira vez, o Vagrant irá baixar a imagem base do Ubuntu (ubuntu/jammy64), o que pode levar alguns minutos dependendo da sua conexão de internet.

Você consegue acessar a máquina virtual da seguinte forma:

```
vagrant ssh control
```

Aí para testar a conectividade das máquinas, dentro da VM que tem Ansible instalado você roda o seguinte

```
ansible all -m ping
```

O provisionamento já cria automaticamente uma chave SSH no `control` e adiciona essa chave nos nós, permitindo acesso sem senha.

Caso você tenha subido as VMs antes desta configuração, reprovisione para aplicar as mudanças:

```
vagrant reload control --provision
vagrant reload node1 --provision
vagrant reload node2 --provision

```
## Meu ambiente

Este projeto faz parte dos meus estudos DevOps.