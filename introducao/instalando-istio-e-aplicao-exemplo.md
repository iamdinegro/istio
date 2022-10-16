# Instalando o Istio e implementando um app exemplo

No momento em que escrevo essa documentação, o Istio está na sua versão ```1.15.2``` , então será ela que iremos utilizar junto ao Kubernetes na versão ```1.25.2```.
## Instalando o Istio client
Para começar, iremos fazer o Download do istioctl e dos manifestos da aplicação exemplo que utilizaremos:

```bash
mkdir -p ~/istio/intro && cd ~/istio/intro
curl -L https://istio.io/downloadIstio | ISTIO_VERSION=1.15.2 TARGET_ARCH=x86_64 sh -
cd istio-1.15.2
export PATH=$PWD/bin:$PATH #<- Adicione esse export no seu  ~/.zshrc ou ~/.bashrc
```

## Instalando o istio no Kubernetes

Agora seguindo com a instalação do Istio no Kubernetes, devemos saber que há diversas maneiras diferentes de instalá-lo. O ```istioctl```, binário de instalação e administração do istio suporta alguns perfis de instalação do istio, são esses:

```default, demo, minimal, external, empty, preview```

Veja mais sobre esses perfis e configurações avançadas de instalação como ```hpaSpec```, ```affinity``` entre outras ```KubernetesResourcesSpec``` em:

https://istio.io/latest/docs/setup/additional-setup/config-profiles/


Seguindo com a instalação do istio no K8s, utilizaremos o perfil ```demo```. Já com o kubectl configurado em sua máquina, utilize o seguinte comando para instalar os componentes do istio:

```bash
istioctl install --set profile=demo -y
```

O comando deverá retorna uma saída como a seguinte:
```bash
✔ Istio core installed                                                                                                                                                                                                                                                                    
✔ Istiod installed                                                                                                                                                                                                                                                                        
✔ Egress gateways installed                                                                                                                                                                                                                                                               
✔ Ingress gateways installed                                                                                                                                                                                                                                                              
✔ Installation complete                                                                                                                                                                                                                                                                   Making this installation the default for injection and validation.

Thank you for installing Istio 1.15.  Please take a few minutes to tell us about your install/upgrade experience!  https://forms.gle/SWHFBmwJspusK1hv6

```
Caso tenha dado algum problema na instalação do seu istio, é importante que você adicione sua experiência no formulário que o comando retorna :)

Para garantir que a instalação do Istio ocorreu bem, utilize o seguinte comando para confirmar que os pods estão rodando:

```bash
kubectl get pods -n istio-system

NAME                                    READY   STATUS    RESTARTS   AGE
istio-egressgateway-fffc799cf-88r8j     1/1     Running   0          4m38s
istio-ingressgateway-7d68764b55-7wgx9   1/1     Running   0          4m38s
istiod-5456fd558d-bgfwz                 1/1     Running   0          5m7s
```

Para garantir que o istio adicione automaticamente um sidecar proxy quando você fizer o deploy de sua aplicação, utilize o seguinte comando para aplicar a label ```istio-injection=enabled``` em sua namespace:

```bash
kubectl label namespace default istio-injection=enabled

namespace/default labeled
```

## Instalando o app de exemplo:

Para instalar a aplicação de exemplo do istio, utilize o seguinte comando:

```bash
kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml

service/details created
serviceaccount/bookinfo-details created
deployment.apps/details-v1 created
service/ratings created
serviceaccount/bookinfo-ratings created
deployment.apps/ratings-v1 created
service/reviews created
serviceaccount/bookinfo-reviews created
deployment.apps/reviews-v1 created
deployment.apps/reviews-v2 created
deployment.apps/reviews-v3 created
service/productpage created
serviceaccount/bookinfo-productpage created
deployment.apps/productpage-v1 created
```

Após alguns minutos, você pode utilizar o comando:

```bash
kubectl get pods

NAME                             READY   STATUS    RESTARTS   AGE
details-v1-5ffd6b64f7-8vwcc      2/2     Running   0          4m48s
productpage-v1-979d4d9fc-lmtjs   2/2     Running   0          4m47s
ratings-v1-5f9699cfdf-bpfrq      2/2     Running   0          4m48s
reviews-v1-569db879f5-fnwvj      2/2     Running   0          4m48s
reviews-v2-65c4dc6fdc-bh8j9      2/2     Running   0          4m48s
reviews-v3-c9c4fb987-wbsl2       2/2     Running   0          4m47s
```

Agora que os pods da aplicação exemplo estão rodando. Você pode utilizar o seguinte comando para testá-la:

```bash
kubectl exec "$(kubectl get pod -l app=ratings -o jsonpath='{.items[0].metadata.name}')" -c ratings -- curl -sS productpage:9080/productpage | grep -o "<title>.*</title>"

# Retorno esperado:

<title>Simple Bookstore App</title>
```

## Configurando ingressGateway e virtualService

A aplicação teste foi deployada porém não há acesso externo. Para deixarmos com acesso externo, vamos criar o Ingress Gateway e VirtualService:

```bash
kubectl apply -f samples/bookinfo/networking/bookinfo-gateway.yaml
```

Para conseguir o IP e Porta do IngressGateway, utilize os seguinte comando:

```bash
export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].nodePort}')
export SECURE_INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="https")].nodePort}')

export GATEWAY_URL=$INGRESS_HOST:$INGRESS_PORT
```

Agora, utilize o seguinte comando para conseguir o endereço externo da aplicação Exemplo:

```bash
echo "http://$GATEWAY_URL/productpage"
```
Isso irá retornar algo como: http://172.18.0.5:30289/productpage, onde você poderá acessar pelo seu browser e verificar a aplicação:

![Alt text](./img/screen-bookinfo.png?raw=true "Bookinfo App")

### Agora preste atenção!!

Repare que, ao você dar ```f5``` na página várias vezes, as informações da página irá atualizar. Isso ocorre graças ao algoritmo de round robin que faz um balanceamento de carga entre os 3 deployments do microsservice ```reviews```:

```bash
kubectl get deployment | grep reviews

reviews-v1       1/1     1            1           24m
reviews-v2       1/1     1            1           24m
reviews-v3       1/1     1            1           24m
```
Massa né?!

## Adicionando uma dashboard do Kiali

Ainda na pasta com os manifestos do exemplo, utilize o seguinte comando para adicionar os addons:

```bash
kubectl apply -f samples/addons

serviceaccount/grafana created
configmap/grafana created
service/grafana created
deployment.apps/grafana created
configmap/istio-grafana-dashboards created
configmap/istio-services-grafana-dashboards created
deployment.apps/jaeger created
service/tracing created
service/zipkin created
service/jaeger-collector created
serviceaccount/kiali created
configmap/kiali created
clusterrole.rbac.authorization.k8s.io/kiali-viewer created
clusterrole.rbac.authorization.k8s.io/kiali created
clusterrolebinding.rbac.authorization.k8s.io/kiali created
role.rbac.authorization.k8s.io/kiali-controlplane created
rolebinding.rbac.authorization.k8s.io/kiali-controlplane created
service/kiali created
deployment.apps/kiali created
serviceaccount/prometheus created
configmap/prometheus created
clusterrole.rbac.authorization.k8s.io/prometheus created
clusterrolebinding.rbac.authorization.k8s.io/prometheus created
service/prometheus created
deployment.apps/prometheus created
```
```bash
kubectl rollout status deployment/kiali -n istio-system

Waiting for deployment "kiali" rollout to finish: 0 of 1 updated replicas are available...
deployment "kiali" successfully rolled out

```

Agora, vamos acessar a dashboard do kiali:

```bash
istioctl dashboard kiali
```

Abrirá essa dashboardo do kiali e então, vamos gerar tráfego para nossa app:

![Alt text](./img/screen-kiali.png?raw=true "Bookinfo App")

```bash
for i in $(seq 1 100); do curl -s -o /dev/null "http://$GATEWAY_URL/productpage"; done
```

Ao acessar a aplicação reviews na dashboard do kiali, você verá algo semelhante à isso aqui:

![Alt text](./img/kiali-dash.png?raw=true "Bookinfo App")

MASSAAAAA NÉ?

Estude, explore as métricas e conheça a dashboard do Kiali e nos vemos ná próxima :)