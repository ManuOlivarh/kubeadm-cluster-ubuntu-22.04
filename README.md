# Guía Kubernetes clúster KUBEADM - Ubuntu 22.04

Esta guía te ayudará a instalar un cluster de Kubernetes utilizando kubeadm en Ubuntu 22.04. Kubeadm es una herramienta que facilita la creación de clusters de Kubernetes de forma rápida y sencilla en sistemas Ubuntu 22.04.

A lo largo de esta guía, te llevaré a través de los pasos necesarios para configurar un entorno de Kubernetes en tu sistema Ubuntu 22.04 utilizando kubeadm, basado en la versión 1.28 de Kubernetes, que es la versión actual para el examen Certified Kubernetes Administrator (CKA). Esto incluirá la instalación de los componentes necesarios, la configuración de los nodos maestros y trabajadores, y la verificación de que tu cluster de Kubernetes esté funcionando correctamente

### Habilitar los módulos del núcleo y habilitar bridge traffic en iptables

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

```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
```

```bash
sudo sysctl --system
```

### Deshabilitar swap

```bash
sudo swapoff -a
```

```bash
(crontab -l 2>/dev/null; echo "@reboot /sbin/swapoff -a") | crontab - || true
```


### Instalar Container Runtime - CRI-O
Una ventaja para la version 1.28 es que CRI-O se puede instalar de otra forma diferente a la que sea habia instalado hasta ahora, ya que se estan introduciendo los repositorios de paquetes de propiedad de la comunidad de Kubernetes

[CRI-O is moving towards pkgs.k8s.io](https://kubernetes.io/blog/2023/08/15/pkgs-k8s-io-introduction/)
```md
"La comunidad de Kubernetes anunció recientemente que sus repositorios de paquetes heredados están congelados y ahora han comenzado a introducir repositorios de paquetes de propiedad comunitaria respaldados por el OpenBuildService (OBS). CRI-O tiene una larga historia de utilizar OBS para la construcción de paquetes, pero hasta ahora, todos los esfuerzos de empaquetado se han realizado de forma manual.
La comunidad de CRI-O ama absolutamente a Kubernetes, lo que significa que están encantados de anunciar que:
¡Todos los futuros paquetes de CRI-O se enviarán como parte de la infraestructura oficialmente respaldada por Kubernetes alojada en pkgs.k8s.io!

Habrá una fase de descontinuación para los paquetes existentes, la cual se está discutiendo actualmente en la comunidad de CRI-O. La nueva infraestructura solo admitirá versiones de CRI-O >= v1.28.2 y las ramas de lanzamiento más nuevas que release-1.28."
```


Instalamos dependencias para poder agregar repositorios
```bash
apt-get update
apt-get install -y software-properties-common curl
```
Agregamos repositorio de CRI-O
```bash
curl -fsSL https://pkgs.k8s.io/addons:/cri-o:/prerelease:/main/deb/Release.key |
    gpg --dearmor -o /etc/apt/keyrings/cri-o-apt-keyring.gpg
echo "deb [signed-by=/etc/apt/keyrings/cri-o-apt-keyring.gpg] https://pkgs.k8s.io/addons:/cri-o:/prerelease:/main/deb/ /" |
    tee /etc/apt/sources.list.d/cri-o.list
```
### Agregamos el repositorio de Kuberenetes
```bash
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key |
    gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /" |
    tee /etc/apt/sources.list.d/kubernetes.list
```


### Instalamos los paquetes (Kubeadm - Kubelet - Kubectl - CRI-O)
```bash
sudo apt-get update
sudo apt-get install -y cri-o kubelet kubeadm kubectl
```
```bash
sudo apt-mark hold kubelet kubeadm kubectl
```
```bash
systemctl start crio.service
```

Instalamos JQ
```bash
sudo apt-get install -y jq
```

### Incializamos Kubeadm en el Master Node para configurar el Control Plane

Agregamos variables antes de inicializar kubeadm

```bash
IPADDR="192.168.5.11"  
NODENAME=$(hostname -s)
POD_CIDR="192.168.0.0/16"
```

```bash
sudo kubeadm init --apiserver-advertise-address=$IPADDR  --apiserver-cert-extra-sans=$IPADDR  --pod-network-cidr=$POD_CIDR --node-name $NODENAME --ignore-preflight-errors Swap
```
Tras una inicialización exitosa de kubeadm, deberías recibir una salida con la ubicación del archivo de kubeconfig y el comando de unión con el token. Cópialo y guárdalo en un archivo, ya que lo necesitaremos para unir el nodo worker al nodo mater.

Salimos de sudo y cambiamos a usuario regular y ejecutamos el siguiente comando:

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Verificamos los pods

```bash
kubectl get po -n kube-system
```
