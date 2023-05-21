require 'yaml'

$root_dir = File.dirname(File.expand_path(__FILE__))
$settings = YAML.load_file(File.join($root_dir, "settings.yaml"))
$workaround = "false"
$runtime_type = ""
$HOSTPort=5566

def detect_runtime
  case $settings['kubernetes_version']
  when /v1.2[2345].*(k3s|rke2).*/
    $workaround = "true"
  when /v1\.21.*(k3s|rke2).*/
  else
    puts "Unsupported Kubernetes runtime #{$settings['kubernetes_version']}"
    exit(1)
  end

  case $settings['kubernetes_version']
  when /.*k3s.*/
    $runtime_type = "k3s"
  when /.*rke2.*/
    $runtime_type = "rke2"
  else
    puts "Unsupported Kubernetes runtime #{$settings['kubernetes_version']}"
    exit(1)
  end

  File.open(File.join($root_dir, "runtime"), "w") { |f| f.write $runtime_type }
end

detect_runtime

# open-iscsi is the dependency for longhorn
$provision_prepare = <<-PROVISION_PREPARE
zypper ref
zypper in -y apparmor-parser iptables wget open-iscsi

PROVISION_PREPARE

$provision_server_config = <<-PROVISION_SERVER_CONFIG
cat > /etc/bash.bashrc.local <<EOF
if [ -z "$KUBECONFIG" ]; then
    if [ -e /etc/rancher/rke2/rke2.yaml ]; then
        export KUBECONFIG="/etc/rancher/rke2/rke2.yaml"
    else
        export KUBECONFIG="/etc/rancher/k3s/k3s.yaml"
    fi
fi
export PATH="${PATH}:/var/lib/rancher/rke2/bin"
if [ -z "$CONTAINER_RUNTIME_ENDPOINT" ]; then
    export CONTAINER_RUNTIME_ENDPOINT=unix:///var/run/k3s/containerd/containerd.sock
fi
if [ -z "$IMAGE_SERVICE_ENDPOINT" ]; then
    export IMAGE_SERVICE_ENDPOINT=unix:///var/run/k3s/containerd/containerd.sock
fi

# For ctr
if [ -z "$CONTAINERD_ADDRESS" ]; then
    export CONTAINERD_ADDRESS=/run/k3s/containerd/containerd.sock
fi
EOF


mkdir -p /etc/rancher/rancherd
cat > /etc/rancher/rancherd/config.yaml << EOF
role: cluster-init
token: somethingrandom
kubernetesVersion: #{$settings['kubernetes_version']}
rancherVersion: #{$settings['rancher_version']}
rancherInstallerImage: rancher/system-agent-installer-rancher:v2.7.3
rancherValues:
  noDefaultAdmin: false
  privateCA: true
  bootstrapPassword: #{$settings['rancher_admin_passwd']}
  features: multi-cluster-management=true,multi-cluster-management-agent=false
  global:
    cattle:
      psp:
        enabled: false
  hostPort: #{$HOSTPort}
EOF

mkdir -p /etc/rancher/rke2/config.yaml.d/
cat > /etc/rancher/rke2/config.yaml.d/99-vagrant-rancherd.yaml << EOF
cni: multus,canal
disable: rke2-ingress-nginx

EOF

PROVISION_SERVER_CONFIG


$provision_worker_config = <<-PROVISION_WORKER_CONFIG

mkdir -p /etc/rancher/rancherd
cat > /etc/rancher/rancherd/config.yaml << EOF
role: agent 
token: somethingrandom
server: https://#{$settings['server_ip']}:8443
EOF

mkdir -p /etc/rancher/rke2/config.yaml.d/
cat > /etc/rancher/rke2/config.yaml.d/99-vagrant-rancherd.yaml << EOF
cni: multus,canal
disable: rke2-ingress-nginx

EOF

PROVISION_WORKER_CONFIG


$provision_rancherd = <<-PROVISION_RANCHERD
if [ "#{$workaround}" = "true" ]; then
  curl -fL https://raw.githubusercontent.com/rancher/rancherd/harvester-dev/install.sh | sh -
else
  curl -fL https://raw.githubusercontent.com/rancher/rancherd/master/install.sh | sh -
fi

PROVISION_RANCHERD


$create_tls_ca = <<-CREATE_TLS_CA

echo "Check the TLS_TA..."
  retries=0
  while [ $retries -lt 60 ]; do
    source /etc/bash.bashrc.local
    kubectl get secrets tls-ca -n cattle-system
    if [[ $? != 0 ]]; then
      rm -rf /etc/rancher/ssl/cacerts.pem || true
      echo "tls-ca not found, creating..."
      mkdir -p /etc/rancher/ssl/
      cat > /etc/rancher/ssl/cacerts.pem <<EOF
-----BEGIN CERTIFICATE-----
MIIC8DCCAdigAwIBAgIUJhBQ8fZsCtWmjqP6e0jPFdi2JtIwDQYJKoZIhvcNAQEL
BQAwEjEQMA4GA1UEAwwHdGVzdC1jYTAeFw0yMzA1MTkxMTIyMDlaFw0yMzA3MTgx
MTIyMDlaMBcxFTATBgNVBAMMDGZyZWV6ZWlvLmNvbTCCASIwDQYJKoZIhvcNAQEB
BQADggEPADCCAQoCggEBAMjQTJ5ILWBEFWWCYg7m6oFFFZ6/Qi5m/Rg5x3VYyvRl
PzVICEgIC6vEH0BFT9/zDTWZmE4s0k5gpiQ+OoVG9fRfqTj60yG/YY4cHfXlDyrI
c0bUc+RK8sPnhUNrDvyI0J3EJiuHYvwSyXSm+IqfU/VawfKtXCDsDIgZyi20gS2i7
S59OVlgXNmCbQHPZzLwLbCgvNDfOdKPQmJF8+OPzBEtdSaRoiHK812fA2A0t1Kci
0o37okYvy98t6EO2bqUKhxxb+pGF8NsQgOeJdCfUgY0NRV8hrPOwnue4N1B8UEmY
PpknZadoxR3T6SZznGxt+wOcgn03f6HqkqxIcS1Tp+ECAwEAAaM5MDcwCQYDVR0T
BAIwADALBgNVHQ8EBAMCBeAwHQYDVR0lBBYwFAYIKwYBBQUHAwIGCCsGAQUFBwMB
MA0GCSqGSIb3DQEBCwUAA4IBAQBi1zRh6S8l/Crky8tIG+wU0rl62f0BxTNV6JeS
at7H1v8oOvhW7wL3Hf67ESAiVg75W1pCmVFHmJpedCmx0kVF+D9dlGkNaX4u/zaT
O8g0e0ar0+wxV3HWXOPsZz+TrVpggGO8velFgKJ2wfFB/A5yuche/bh85vb2+CKb
uD28yGtGig7B1eeOeL4KxZbkYkA7jdUm+6sjQom0dGMC/QbEekTXOwCUPtkMkCUd
gdkGNZz49k9kCZ+gcBPg7L24FrwmGOqJVuZdz8KpTzm2FPcum4rFnh6vNUuHwTEX
HkLdg2cC9EN6DADkQaNxBgd2/SlozelVlqAjaXoSkitdPJ1v
-----END CERTIFICATE-----
EOF

      curl --insecure -fL https://localhost:#{$HOSTPort}/cacerts >> /etc/rancher/ssl/cacerts.pem
      kubectl -n cattle-system create secret generic tls-ca --from-file=/etc/rancher/ssl/cacerts.pem
    else
      echo "tls-ca found!"
      exit 0
    fi
    sleep 10
    retries=$((retries+1))
  done
  echo "try to create tls-ca failed. Break here"
  exit 1

CREATE_TLS_CA
    

$wait_rancherd_bootstrapped = <<-WAIT_RANCHERD

echo "Waiting for Rancherd bootstrapped..."
  retries=0
  while [ $retries -lt 120 ]; do
    bootstrapped=$(sudo cat /var/lib/rancher/rancherd/bootstrapped 2>/dev/null || true)

    echo "."
    sleep 10

    curl --insecure -fL https://localhost:#{$HOSTPort}/cacerts
    if [[ $? == 0 ]]; then
      echo "update cacerts chain"
      curl --insecure -fL https://localhost:#{$HOSTPort}/cacerts >> /etc/rancher/ssl/cacerts.pem
      kubectl -n cattle-system create secret generic tls-ca --from-file=/etc/rancher/ssl/cacerts.pem --dry-run --save-config -o yaml | kubectl apply -f -
      kubectl rollout restart deploy/rancher -n cattle-system
      exit 0
    fi
    retries=$((retries+1))
  done
  echo "Rancherd can't bootstrap, you can check log with:"
  echo '$ vagrant ssh node1'
  echo '$ sudo journalctl -u rancherd'
  exit 1
WAIT_RANCHERD


Vagrant.configure("2") do |config|
  config.vm.box = "opensuse/Leap-15.3.x86_64"
  config.vm.synced_folder ".", "/vagrant", disabled: true

  (1..4).each do |i|
    config.vm.define "node#{i}" do |node|
      node.vm.hostname = "node#{i}"
      node.vm.network 'private_network', libvirt__network_name: 'harvester'
      node.vm.provider "libvirt" do |lv|
        lv.driver = $settings['driver']
        lv.connect_via_ssh = false
        lv.qemu_use_session = false
        lv.storage_pool_name = 'general'


        if $settings['driver'] == 'kvm'
          lv.cpu_mode = 'host-passthrough'
        end

        lv.memory = 16384
        lv.cpus = 8

        if $settings['ci']
          lv.management_network_name = 'vagrant-libvirt-ci'
          lv.management_network_address = '192.168.124.0/24'
        end

        lv.storage :file, :size => '50G', :device => 'vdb'
        lv.graphics_ip = '0.0.0.0'
      end

      #node.vm.provision:shell, inline: <<-SHELL
      #  echo "root:p@ssw0rd" | sudo chpasswd
      #SHELL
      node.vm.provision "shell", inline: $provision_prepare
      # The first node is server, others are workers
      if i > 1
        node.vm.provision "shell", inline: $provision_worker_config
        node.vm.provision "shell", inline: $provision_rancherd
      else
        node.vm.provision "shell", inline: $provision_server_config
        node.vm.provision "shell", inline: $provision_rancherd
        node.vm.provision "shell", inline: $create_tls_ca
        node.vm.provision "shell", inline: $wait_rancherd_bootstrapped
      end
    end
  end
end
