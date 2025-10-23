# Vagrantfile definitivo - 3 VMs: monitor (Prometheus+Grafana), webserver (Nginx+node_exporter), control (Ansible)
Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/focal64"

  # ----------------------------
  # VM: monitor (Prometheus + Grafana)
  # ----------------------------
  config.vm.define "monitor" do |monitor|
    monitor.vm.hostname = "monitor"
    monitor.vm.network "private_network", ip: "192.168.56.101"
    # Forward para acceso desde el host
    monitor.vm.network "forwarded_port", guest: 9090, host: 9090
    monitor.vm.network "forwarded_port", guest: 3000, host: 3000

    monitor.vm.provider "virtualbox" do |vb|
      vb.name = "vagrant-monitor"
      vb.memory = 4096
      vb.cpus = 2
      vb.gui = false
    end

    monitor.vm.provision "shell", inline: <<-SHELL
      #!/usr/bin/env bash
      set -e
      export DEBIAN_FRONTEND=noninteractive

      # 1) crear swap de 2GB si no existe (evita OOM durante apt/dpkg)
      if ! swapon --show | grep -q '/swapfile'; then
        if command -v fallocate >/dev/null 2>&1; then
          fallocate -l 2G /swapfile || dd if=/dev/zero of=/swapfile bs=1M count=2048
        else
          dd if=/dev/zero of=/swapfile bs=1M count=2048
        fi
        chmod 600 /swapfile
        mkswap /swapfile
        swapon /swapfile
        echo '/swapfile none swap sw 0 0' >> /etc/fstab
      fi

      apt-get update -y
      apt-get install -y --no-install-recommends wget curl tar gnupg apt-transport-https

      # 2) Instalar Prometheus (release tarball)
      PROM_VERSION="2.48.0"
      PROM_TARBALL="prometheus-${PROM_VERSION}.linux-amd64.tar.gz"
      PROM_URL="https://github.com/prometheus/prometheus/releases/download/v${PROM_VERSION}/${PROM_TARBALL}"

      useradd --no-create-home --shell /usr/sbin/nologin prometheus || true
      mkdir -p /etc/prometheus /var/lib/prometheus
      cd /tmp
      if [ ! -f /tmp/${PROM_TARBALL} ]; then
        wget -q ${PROM_URL} -O /tmp/${PROM_TARBALL}
      fi
      tar xzf /tmp/${PROM_TARBALL} -C /tmp
      PROMDIR=$(tar -tf /tmp/${PROM_TARBALL} | head -1 | cut -f1 -d"/")
      cp /tmp/${PROMDIR}/prometheus /usr/local/bin/
      cp /tmp/${PROMDIR}/promtool /usr/local/bin/
      cp -r /tmp/${PROMDIR}/consoles /etc/prometheus
      cp -r /tmp/${PROMDIR}/console_libraries /etc/prometheus
      # basic config (prometheus + node exporter on webserver)
      cat > /etc/prometheus/prometheus.yml <<'EOF'
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']
  - job_name: 'node_exporter'
    static_configs:
      - targets: ['192.168.56.102:9100']
EOF

      chown -R prometheus:prometheus /etc/prometheus /var/lib/prometheus || true
      chmod +x /usr/local/bin/prometheus /usr/local/bin/promtool || true

      cat > /etc/systemd/system/prometheus.service <<'EOF'
[Unit]
Description=Prometheus
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/prometheus --config.file=/etc/prometheus/prometheus.yml --storage.tsdb.path=/var/lib/prometheus

[Install]
WantedBy=multi-user.target
EOF

      systemctl daemon-reload
      systemctl enable prometheus || true
      systemctl start prometheus || true

      # 3) Instalar Grafana desde repo oficial
      # (apt-key puede avisar deprecación, pero funciona en focal)
      wget -q -O - https://packages.grafana.com/gpg.key | apt-key add - || true
      echo "deb https://packages.grafana.com/oss/deb stable main" > /etc/apt/sources.list.d/grafana.list
      apt-get update -y
      apt-get install -y --no-install-recommends grafana

      systemctl daemon-reload
      systemctl enable grafana-server || true
      systemctl start grafana-server || true

      echo "=== monitor provision complete ==="
      echo "Prometheus: http://192.168.56.101:9090  Grafana: http://192.168.56.101:3000"
    SHELL
  end

  # ----------------------------
  # VM: webserver (Nginx + node_exporter)
  # ----------------------------
  config.vm.define "webserver" do |web|
    web.vm.hostname = "webserver"
    web.vm.network "private_network", ip: "192.168.56.102"
    web.vm.network "forwarded_port", guest: 80, host: 8080

    web.vm.provider "virtualbox" do |vb|
      vb.name = "vagrant-webserver"
      vb.memory = 2048
      vb.cpus = 1
      vb.gui = false
    end

    web.vm.provision "shell", inline: <<-SHELL
      #!/usr/bin/env bash
      set -e
      export DEBIAN_FRONTEND=noninteractive

      apt-get update -y
      apt-get install -y --no-install-recommends nginx wget

      # Página simple
      cat > /var/www/html/index.html <<'EOF'
<html><body><h1>Nginx en vagrant-webserver</h1><p>IP: 192.168.56.102</p></body></html>
EOF

      systemctl enable nginx
      systemctl restart nginx

      # Instalar node_exporter (para Prometheus)
      NODE_VER="1.6.1"
      NODE_TAR="node_exporter-${NODE_VER}.linux-amd64.tar.gz"
      NODE_URL="https://github.com/prometheus/node_exporter/releases/download/v${NODE_VER}/${NODE_TAR}"
      cd /tmp
      if [ ! -f /tmp/${NODE_TAR} ]; then
        wget -q ${NODE_URL} -O /tmp/${NODE_TAR}
      fi
      tar xzf /tmp/${NODE_TAR} -C /tmp
      NODE_DIR=$(tar -tf /tmp/${NODE_TAR} | head -1 | cut -f1 -d"/")
      cp /tmp/${NODE_DIR}/node_exporter /usr/local/bin/
      useradd --no-create-home --shell /usr/sbin/nologin node_exporter || true
      mkdir -p /var/lib/node_exporter
      chown node_exporter:node_exporter /var/lib/node_exporter || true

      cat > /etc/systemd/system/node_exporter.service <<'EOF'
[Unit]
Description=Node Exporter
Wants=network-online.target
After=network-online.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/usr/local/bin/node_exporter

[Install]
WantedBy=multi-user.target
EOF

      systemctl daemon-reload
      systemctl enable node_exporter || true
      systemctl start node_exporter || true

      echo "=== webserver provision complete ==="
      echo "Nginx: http://192.168.56.102 (host: http://localhost:8080)  node_exporter: 192.168.56.102:9100"
    SHELL
  end

  # ----------------------------
  # VM: control (Ansible)
  # ----------------------------
  config.vm.define "control" do |control|
    control.vm.hostname = "control"
    control.vm.network "private_network", ip: "192.168.56.103"
    control.vm.provider "virtualbox" do |vb|
      vb.name = "vagrant-control"
      vb.memory = 2048
      vb.cpus = 2
      vb.gui = false
    end

    control.vm.provision "shell", inline: <<-SHELL
      #!/usr/bin/env bash
      set -e
      export DEBIAN_FRONTEND=noninteractive

      apt-get update -y
      apt-get install -y --no-install-recommends software-properties-common sshpass wget

      # Instalar Ansible (PPA)
      apt-add-repository --yes --update ppa:ansible/ansible
      apt-get update -y
      apt-get install -y --no-install-recommends ansible

      # Generar clave SSH para vagrant user (si no existe)
      if [ ! -f /home/vagrant/.ssh/id_rsa ]; then
        sudo -u vagrant ssh-keygen -t rsa -N "" -f /home/vagrant/.ssh/id_rsa
      fi
      chown -R vagrant:vagrant /home/vagrant/.ssh

      # Crear inventario simple en /vagrant/ansible_inventory.ini (hostnames + ip)
      mkdir -p /vagrant/ansible
      cat > /vagrant/ansible/inventory.ini <<'EOF'
[monitor]
192.168.56.101

[web]
192.168.56.102

[all:vars]
ansible_user=vagrant
ansible_ssh_common_args='-o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null'
EOF

      echo "=== control provision complete ==="
      echo "Inventory at /vagrant/ansible/inventory.ini. Desde 'vagrant ssh control' puedes ejecutar ansible-playbook -i /vagrant/ansible/inventory.ini <playbook.yml>"
    SHELL
  end
end
