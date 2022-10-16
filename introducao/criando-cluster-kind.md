# Criando um cluster local com Kind

Para começar os estudos de istio, criaremos um cluster local de Kubernetes utilizando o kind. Nesse exemplo, estarei considerando que você já tenha o ```kubectl``` e ```docker``` instalado em sua máquina local.


Para instalá-lo, basta seguir o seguinte passo:

```bash
cd /tmp
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.16.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind
```

Para validar se o binário do kind foi instalado corretamente, utilize:

```bash
kind --version
# Se ocorreu tudo OK, o comando retornará a versão atual do Kind.
kind version 0.16.0
```

Agora, você deve criar um arquivo ```kind.yaml``` conforme o abaixo: 

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
networking:
  disableDefaultCNI: true # Isso vai desabilitar o CNI padrão do Kind, para então podermos instalar o Calico
  podSubnet: 192.168.0.0/16
nodes:
- role: control-plane
- role: control-plane
- role: worker
- role: worker
- role: worker
``` 

E em seguida, digite o seguinte comando para criar um cluster:

```bash
kind create cluster --config kind.yaml

# e para testar se seu cluster foi criado corretamente, utilize:
kubectl get nodes

# A saída deverá ser a seguinte: 
NAME                  STATUS     ROLES           AGE   VERSION
kind-control-plane    NotReady   control-plane   99s   v1.25.2
kind-control-plane2   NotReady   control-plane   87s   v1.25.2
kind-worker           NotReady   <none>          7s    v1.25.2
kind-worker2          NotReady   <none>          7s    v1.25.2
kind-worker3          NotReady   <none>          13s   v1.25.2
```

Agora, vamos instalar a CNI do calico para termos conectividade dentro do Cluster:

```bash
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
kubectl -n kube-system set env daemonset/calico-node FELIX_IGNORELOOSERPF=true
```

Esse processo pode demorar ~10min...


```bash
kubectl get nodes
NAME                  STATUS   ROLES           AGE     VERSION
kind-control-plane    Ready    control-plane   8m59s   v1.25.2
kind-control-plane2   Ready    control-plane   8m42s   v1.25.2
kind-worker           Ready    <none>          7m27s   v1.25.2
kind-worker2          Ready    <none>          7m28s   v1.25.2
kind-worker3          Ready    <none>          7m28s   v1.25.2


kubectl get pods -n kube-system | grep calico-node
calico-node-7hrdw                             1/1     Running   0               6m51s
calico-node-hj246                             1/1     Running   0               6m51s
calico-node-vgdj9                             1/1     Running   0               6m51s
calico-node-xsk2f                             1/1     Running   0               6m51s
calico-node-zm7k9                             1/1     Running   0               6m51s
```
Pronto, agora que o seu cluster local está pronto, podemos passar para a parte da instalação do Istio.

