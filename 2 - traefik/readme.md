# GitHub Traefik https://github.com/traefik/traefik
# Primeiro adicione Traefik nos seus repositórios do Helm
```bash
$ helm repo add traefik https://helm.traefik.io/traefik
$ helm repo update
```
# Crie o namespace para o Traefik
```bash
$ kubectl create namespace traefik
```
# Configure um values.yaml local
## Seção service no values.yaml
values.yaml
```yaml
service:
  enabled: true
  type: LoadBalancer
  annotations: {}
  labels: {}
  spec:
    loadBalancerIP: 10.1.1.201 # Este IP deve fazer parte da faixa separada para o MetalLB
  loadBalancerSourceRanges: []
  externalIPs: []
```
# Instale o treafik via helm chart apontando para o values.yaml local.
```bash
$ helm install --namespace=traefik traefik traefik/traefik --values=values.yaml
```
## Verifique a criação do serviço e o IP alocado
```bash
$ kubectl get svc --all-namespaces -o wide
```
# Crie os middlewares para cada namespace
mdw-headers-default.yaml
```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: Middleware
metadata:
  name: default-headers
  namespace: default
spec:
  headers:
    browserXssFilter: true
    contentTypeNosniff: true
    forceSTSHeader: true
    stsIncludeSubdomains: true
    stsPreload: true
    stsSeconds: 15552000
    customFrameOptionsValue: SAMEORIGIN
    customRequestHeaders:
      X-Forwarded-Proto: https   
```
Aplique o manifesto
```bash
$ kubectl apply -f mdw-headers-default.yaml
$ kubectl apply -f mdw-headers-traefik.yaml
```
# Gere uma secret estática usando OpenSSL e htpasswd para acessar o dashboard
```bash
$ htpasswd -nb $USER $PASSWD | openssl base64
anViYTokYXByMSRwT1g4aXIuUCQySWd4THZ5cGdXVmRqaS5SdTJIOWcvCgo=
```
Um resultado como acima é esperado, vamos usá-lo para criar uma secret!
# Crie a secret.yaml
secret.yaml
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: traefik-dashboard-auth
  namespace: traefik
type: Opaque
data:
  users: anViYTokYXByMSRwT1g4aXIuUCQySWd4THZ5cGdXVmRqaS5SdTJIOWcvCgo=
```
## Aplique a secret com o comando abaixo
```bash
$ kubectl apply -f secret.yaml 
```
# Crie o middleware para o nosso dashboard
```bash
$ kubectl apply -f mdw-dashboard.yaml 
```
# Faça o deploy do ingresso para o dashboard
ingress.yaml
```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: traefik-dashboard
  namespace: traefik
  annotations: 
    kubernetes.io/ingress.class: traefik
spec:
  entryPoints:
    - websecure
  routes:
    - match: Host(`traefik.home.jubagcastro.dev`) #SAN deve estar em um subdomínio ou no domínio
      kind: Rule
      middlewares:
        - name: traefik-dashboard-basicauth
          namespace: traefik
      services:
        - name: api@internal
          kind: TraefikService
  tls:
    secretName: home-juba #Deve ser o nome da secret do certificado gerado usando o CertManager
```
Aplique o ingress.yaml
```bash
$ kubectl apply -f ingress.yaml
```
# Make some Coffee