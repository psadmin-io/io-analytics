# psadmin.io Analytics

Podman (Docker) Compose project to process PeopleSoft log data.

## OEL 8 Installation

```bash
sudo dnf install podman podman-compose buildah skopeo -y
```

(Optional) If you want to store the containers on a different mount than the root drive:

```
# Create the storage location
sudo su - opc
container_location=/u01/app/containers
sudo mkdir -p $container_location
sudo chown opc:opc $container_location

# Symlink /var/lib/containers
sudo rm -rf /var/lib/containers
sudo ln -s $container_location /var/lib/

# Configure podman to use location - rootless default is ~/.local/containers
mkdir -p ~/.config/containsers
tee ~/.config/containers/storage.conf <<EOF 
[storage]
driver    = "overlay"
graphroot = "/var/lib/containers/storage"
EOF
```

Clone the repo and build the cluster

```bash
sudo su - opc
cd /var/lib/containers
git clone https://github.com/psadmin-io/io-analytics.git
cd io-analytics
podman-compose up
```

## Deploy Filebeat

```bash
sudo rpm --import https://packages.elastic.co/GPG-KEY-elasticsearch
sudo tee /etc/yum.repos.d/beats.repo <<EOF
[elastic-8.x]
name=Elastic repository for 8.x packages
baseurl=https://artifacts.elastic.co/packages/oss-8.x/yum
gpgcheck=1
gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
enabled=1
autorefresh=1
type=rpm-md
EOF

sudo dnf install filebeat -y
sudo systemctl enable filebeat

sudo tee /etc/filebeat/filebeat.yml <<EOF
---
filebeat.inputs:
- type: filestream
  id: peoplesoft-activity
  paths:
    - <PIA_DOMAIN>/servers/PIA/logs/activity.csv
  fields:
    source: appsian
    tier: dev
    application: hr
  fields_under_root: true

output.logstash:
  hosts: ["<logstash-container>:5044"]
EOF

sudo systemctl start filebeat
sudo systemctl status filebeat
```

## Validate Logstash Config

```bash
curl -L -O https://github.com/breml/logstash-config/releases/download/v0.5.3/mustache_0.5.3_Linux_x86_64.tar.gz
tar xvf mustache_0.5.3_Linux_x86_64.tar.gz
chmod +x mustache
./mustache check config/logstash.conf
```