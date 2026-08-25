Vagrant.configure("2") do |config|
  # Box padrão do Ubuntu 22.04 LTS para todas as VMs
  config.vm.box = "ubuntu/jammy64"
  config.ssh.insert_key = true # habilita a chave padrão do Vagrant para login via vagrant ssh

  # 1. Nó de Controle (Onde o Ansible será executado)
  config.vm.define "control" do |control|
    control.vm.hostname = "control"
    control.vm.network "private_network", ip: "192.168.56.10"
    
    control.vm.provider "virtualbox" do |vb|
      vb.memory = "1024"
      vb.cpus = 1
    end

    # Provisionamento: Instala o Ansible e cria arquivos de configuração base
    control.vm.provision "shell", inline: <<-SHELL
      sudo apt-get update
      sudo apt-get install -y software-properties-common ansible sshpass

      # Gera uma chave dedicada para o Ansible no nó de controle
      if [ ! -f /home/vagrant/.ssh/id_ed25519_ansible ]; then
        sudo -u vagrant ssh-keygen -t ed25519 -N "" -f /home/vagrant/.ssh/id_ed25519_ansible
      fi

      # Publica a chave pública no diretório compartilhado entre as VMs
      cp /home/vagrant/.ssh/id_ed25519_ansible.pub /vagrant/ansible_control_key.pub
      chmod 644 /vagrant/ansible_control_key.pub

      # Cria um arquivo de inventário com os IPs dos nós alvo
      cat > /home/vagrant/inventory <<'EOF'
[targets]
node1 ansible_host=192.168.56.11
node2 ansible_host=192.168.56.12

[targets:vars]
ansible_user=vagrant
ansible_ssh_private_key_file=/home/vagrant/.ssh/id_ed25519_ansible
ansible_python_interpreter=/usr/bin/python3
EOF

      # Cria um ansible.cfg para ignorar verificação de chave SSH (ideal para laboratórios)
      echo -e "[defaults]\nhost_key_checking = False\ninventory = /home/vagrant/inventory" > /home/vagrant/ansible.cfg
      
      chown vagrant:vagrant /home/vagrant/inventory /home/vagrant/ansible.cfg
    SHELL
  end

  # 2. Nó Alvo 01
  config.vm.define "node1" do |node1|
    node1.vm.hostname = "node1"
    node1.vm.network "private_network", ip: "192.168.56.11"
    node1.vm.provider "virtualbox" do |vb|
      vb.memory = "512"
    end

    # Garante login SSH por senha para ambiente de laboratório com Ansible
    node1.vm.provision "shell", inline: <<-SHELL
      echo "vagrant:vagrant" | sudo chpasswd
      # Em imagens cloud, arquivos em sshd_config.d podem sobrescrever sshd_config
      sudo tee /etc/ssh/sshd_config.d/99-vagrant-lab.conf >/dev/null <<'EOF'
PasswordAuthentication yes
KbdInteractiveAuthentication yes
ChallengeResponseAuthentication yes
EOF

      # Autoriza a chave pública do control para acesso sem senha
      install -d -m 700 -o vagrant -g vagrant /home/vagrant/.ssh
      touch /home/vagrant/.ssh/authorized_keys
      chown vagrant:vagrant /home/vagrant/.ssh/authorized_keys
      chmod 600 /home/vagrant/.ssh/authorized_keys
      if [ -f /vagrant/ansible_control_key.pub ]; then
        grep -qxF "$(cat /vagrant/ansible_control_key.pub)" /home/vagrant/.ssh/authorized_keys || cat /vagrant/ansible_control_key.pub >> /home/vagrant/.ssh/authorized_keys
      fi

      sudo systemctl restart ssh
    SHELL
  end

  # 3. Nó Alvo 02
  config.vm.define "node2" do |node2|
    node2.vm.hostname = "node2"
    node2.vm.network "private_network", ip: "192.168.56.12"
    node2.vm.provider "virtualbox" do |vb|
      vb.memory = "1024"
    end

    # Garante login SSH por senha para ambiente de laboratório com Ansible
    node2.vm.provision "shell", inline: <<-SHELL
      echo "vagrant:vagrant" | sudo chpasswd
      # Em imagens cloud, arquivos em sshd_config.d podem sobrescrever sshd_config
      sudo tee /etc/ssh/sshd_config.d/99-vagrant-lab.conf >/dev/null <<'EOF'
PasswordAuthentication yes
KbdInteractiveAuthentication yes
ChallengeResponseAuthentication yes
EOF

      # Autoriza a chave pública do control para acesso sem senha
      install -d -m 700 -o vagrant -g vagrant /home/vagrant/.ssh
      touch /home/vagrant/.ssh/authorized_keys
      chown vagrant:vagrant /home/vagrant/.ssh/authorized_keys
      chmod 600 /home/vagrant/.ssh/authorized_keys
      if [ -f /vagrant/ansible_control_key.pub ]; then
        grep -qxF "$(cat /vagrant/ansible_control_key.pub)" /home/vagrant/.ssh/authorized_keys || cat /vagrant/ansible_control_key.pub >> /home/vagrant/.ssh/authorized_keys
      fi

      sudo systemctl restart ssh
    SHELL
  end
end