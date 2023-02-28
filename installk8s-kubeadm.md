# Creación de un clúster de Kubernetes con kubeadm
Con kubeadm, puede crear un clúster de Kubernetes mínimo que se ajuste a las prácticas recomendadas. De hecho, puede usar kubeadm para configurar un clúster que pasará las pruebas de conformidad de Kubernetes . kubeadm también es compatible con otras funciones del ciclo de vida del clúster, como tokens de arranque y actualizaciones del clúster.

La kubeadmherramienta es buena si necesitas:

* Una forma sencilla de probar Kubernetes, posiblemente por primera vez.
* Una forma para que los usuarios existentes automaticen la configuración de un clúster y prueben su aplicación.
* Un bloque de construcción en otro ecosistema y/o herramientas de instalación con un alcance mayor.
* Puede instalarlo y usarlo kubeadmen varias máquinas: su computadora portátil, un conjunto de servidores en la nube, una Raspberry Pi y más. Ya sea que esté implementando en la nube o en las instalaciones OnPremise, puede integrarse kubeadmen sistemas de aprovisionamiento como Ansible o Terraform.

## Copiar la llave en formato PEM proporcionada hacia el servidor student-0-aio y colocarle los permisos adecuados
```bash
chmod 0600 student-0-private_key.pem
```

## Establecer Conexión por medio de SSH con el servidor student-0-master, ejecute el siguiente comando desde el servidor student-0-aio con el usuario student
```bash
ssh student@student-0-master -i student-0-private_key.pem
```

## Forwarding IPv4 and letting iptables see bridged traffic
```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
```

```bash
sudo modprobe overlay
sudo modprobe br_netfilter
```

## sysctl params required by setup, params persist across reboots
```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
```

## Apply sysctl params without reboot
```bash
sudo sysctl --system
```

## Eliminar posibles instalaciones anteriores de Docker CE.
```bash
sudo yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-engine
```

## Agregar al sistema operativo el repositorio de Docker CE
```bash
sudo dnf config-manager --add-repo=https://download.docker.com/linux/centos/docker-ce.repo
```

## Crear el repositorio para paqueteria de Kubernetes
```bash
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-\$basearch
enabled=1
gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
exclude=kubelet kubeadm kubectl
EOF
```

## Set SELinux in permissive mode (effectively disabling it)
```bash
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
```

## Detener firewalld
```bash
sudo systemctl stop firewalld
```

## Instalación de paquetería necesaria
```bash
sudo yum makecache --refresh
sudo yum -y install iproute-tc
sudo yum install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
```

## Configuración de bash auto-completion
```bash
sudo yum install bash-completion -y
echo "source <(kubectl completion bash)" >> ~/.bashrc
source ~/.bashrc
```

## Habilitación de Servicios
```bash
sudo systemctl enable --now kubelet
sudo systemctl enable containerd
sudo systemctl enable --now docker
```

## Iniciar los Servicios
```bash
sudo systemctl start docker
sudo systemctl start kubelet
sudo systemctl start containerd
```

## You need CRI support enabled to use containerd with Kubernetes. Make sure that cri is not included in thedisabled_plugins list within /etc/containerd/config.toml; if you made changes to that file, also restart containerd.
```bash
sudo cp -p /etc/containerd/config.toml /etc/containerd/config.toml_respaldo
sudo vi /etc/containerd/config.toml
#disabled_plugins = ["cri"]
```

## Reiniciar servicio containerd
```bash
sudo systemctl restart containerd
```

## Ejecutar todos los pasos anteriores para el servidor student-0-worker

## Identifique la IP del servidor student-0-master, ejecutando el siguiente comando en el servidor student-0-master
```bash
sudo ip a
```

## Creación de nuevo Cluster, cambiar la variable IPADDR por la IP de su servidor Master, reemplace la variable IPADDR, por la IP identificada anteriormente
```bash
export IPADDR="10.138.0.33"
export NODENAME=$(hostname -s)
export POD_CIDR="192.168.0.0/16"
sudo kubeadm init --apiserver-advertise-address=$IPADDR  --apiserver-cert-extra-sans=$IPADDR  --pod-network-cidr=$POD_CIDR --node-name $NODENAME --ignore-preflight-errors Swap
```

## Guardar en un archivo de texto toda la salida de la ejecución anterior, ya que será utilizada mas adelante

# Ejecutar los siguientes comandos como root.
```bash
sudo -i
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
```bash
kubectl get nodes
kubectl get pods -A
```

## Instalación de CNI Cilium como solución CNI
```bash
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/master/stable.txt)
CLI_ARCH=amd64
if [ "$(uname -m)" = "aarch64" ]; then CLI_ARCH=arm64; fi
curl -L --fail --remote-name-all https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
sha256sum --check cilium-linux-${CLI_ARCH}.tar.gz.sha256sum
sudo tar xzvfC cilium-linux-${CLI_ARCH}.tar.gz /usr/local/bin
rm cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
```
```bash
cilium install
```
```bash
cilium status
```
```bash
kubectl get nodes
kubectl get pods -A
```

## Agregar nuevo nodo al cluster de Kubernetes: Conectarse por SSH y ejecutar el siguiente comando:
```bash
ssh student@student-0-worker -i student-0-private_key.pem
```

## Ejecutar el siguiente comando para incorporar el nuevo servidor, reemplazar esta línea con los valores obtenidos anteriormente
```bash
kubeadm join 10.138.0.33:6443 --token 0ulgah.lm5q88vwg398bk2r --discovery-token-ca-cert-hash sha256:0a3704551e4819102ee481f21a1a5c46f32eafdca8228986038dba85f4f70165
```

## Regresar al servidor Master y verificar el nuevo Nodo con los siguientes comandos:
```bash
kubectl get nodes
kubectl get pods -A
```

## Aagregar la etiqueta de Worker al nuevo servidor
```bash
kubectl label node student-0-worker node-role.kubernetes.io/worker=worker
```

## Instalación de la utilidad HELM
```bash
mkdir ~/bin
cd ~/
wget https://get.helm.sh/helm-v3.3.1-linux-amd64.tar.gz
tar -xf helm-v3.3.1-linux-amd64.tar.gz
mv linux-amd64/helm  ~/bin/helm
chmod +x  ~/bin/helm
```

## Instalación de Ingress Controller
```bash
kubectl create ns ingress-nginx
helm install ingress-nginx ingress-nginx --repo https://kubernetes.github.io/ingress-nginx --namespace ingress-nginx
```

## Create a deployment named my-dep that runs the nginx image with 3 replicas
```bash
kubectl create deployment demo-dep-k8s --image=nginx --replicas=3
```
