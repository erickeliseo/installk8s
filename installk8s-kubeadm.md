



# Copiar la llave en formato PEM proporcionada hacia el servidor student-0-aio y colocarle los permisos adecuados
chmod 0600 student-0-private_key.pem

# Establecer Conexión por medio de SSH con el servidor student-0-master
ssh student@student-0-master -i student-0-private_key.pem


# Forwarding IPv4 and letting iptables see bridged traffic
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# sysctl params required by setup, params persist across reboots
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# Apply sysctl params without reboot
sudo sysctl --system

# Eliminar posibles instalaciones anteriores de Docker CE.
sudo yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-engine

# Crear el repositorio para paqueteria de Kubernetes
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-\$basearch
enabled=1
gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
exclude=kubelet kubeadm kubectl
EOF

# Set SELinux in permissive mode (effectively disabling it)
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

# Detener firewalld
sudo systemctl stop firewalld

# Instalación de paquetería necesaria
sudo yum makecache --refresh
sudo yum -y install iproute-tc
sudo yum install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes

# Configuración de bash auto-completion
sudo yum install bash-completion -y
echo "source <(kubectl completion bash)" >> ~/.bashrc
source ~/.bashrc

# Habilitación de Servicios
sudo systemctl enable --now kubelet
sudo systemctl enable containerd
sudo systemctl enable --now docker
sudo systemctl start docker
sudo systemctl start kubelet
sudo systemctl start containerd

#You need CRI support enabled to use containerd with Kubernetes. Make sure that cri is not included in thedisabled_plugins list within /etc/containerd/config.toml; if you made changes to that file, also restart containerd.

sudo vi /etc/containerd/config.toml
#disabled_plugins = ["cri"]

# Reiniciar servicio containerd
sudo systemctl restart containerd

# Ejecutar todos los pasos anteriores para el servidor student-0-worker

# Creación de nuevo Cluster, cambiar la variable IPADDR por la IP de su servidor Master
export IPADDR="10.138.0.33"
export NODENAME=$(hostname -s)
export POD_CIDR="192.168.0.0/16"
sudo kubeadm init --apiserver-advertise-address=$IPADDR  --apiserver-cert-extra-sans=$IPADDR  --pod-network-cidr=$POD_CIDR --node-name $NODENAME --ignore-preflight-errors Swap

# Guardar la línea que comienza con: kubeadm join .....

# Ejecutar los siguientes comandos como root.
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
  
kubectl get nodes
kubectl get pods -A

# Instalación de CNI Cilium
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/master/stable.txt)
CLI_ARCH=amd64
if [ "$(uname -m)" = "aarch64" ]; then CLI_ARCH=arm64; fi
curl -L --fail --remote-name-all https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
sha256sum --check cilium-linux-${CLI_ARCH}.tar.gz.sha256sum
sudo tar xzvfC cilium-linux-${CLI_ARCH}.tar.gz /usr/local/bin
rm cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}

cilium install

cilium status


# Agregar nuevo nodo: Conectarse por SSH y ejecutar el siguiente comando:
ssh student@student-0-worker -i student-0-private_key.pem

# Ejecutar el siguiente comando para incorporar el nuevo servidor
kubeadm join 10.138.0.33:6443 --token 0ulgah.lm5q88vwg398bk2r --discovery-token-ca-cert-hash sha256:0a3704551e4819102ee481f21a1a5c46f32eafdca8228986038dba85f4f70165

# Regresar al servidor Master y verificar el nuevo Nodo con los siguientes comandos:
kubectl get nodes
kubectl get pods -A

# Aagregar la etiqueta de Worker al nuevo servidor
kubectl label node student-0-worker node-role.kubernetes.io/worker=worker

# Instalación de la utilidad HELM
mkdir ~/bin
cd ~/
wget https://get.helm.sh/helm-v3.3.1-linux-amd64.tar.gz
tar -xf helm-v3.3.1-linux-amd64.tar.gz
mv linux-amd64/helm  ~/bin/helm
chmod +x  ~/bin/helm

# Instalación de Ingress Controller
kubectl create ns ingress-nginx
helm install ingress-nginx ingress-nginx --repo https://kubernetes.github.io/ingress-nginx --namespace ingress-nginx

# Create a deployment named my-dep that runs the nginx image with 3 replicas
kubectl create deployment demo-dep-k8s --image=nginx --replicas=3

