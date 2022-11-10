# GitHub CertManager: https://github.com/cert-manager/cert-manager
# Adicione o repositório do JetStack para instalar o CertManager
```bash
$ helm repo add jetstack https://charts.jetstack.io
$ helm repo update
```
# Crie um namespace para o certmanager
```bash
$ kubectl create namespace cert-manager
```
# Crie um CRD (Custom resource definitions) para os certificados
```bash
$ kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.9.1/cert-manager.crds.yaml
```
# Configure os values
values.yaml
```yaml
installCRDs: false
replicaCount: 3
extraArgs:
  - --dns01-recursive-nameservers=1.1.1.1:53,9.9.9.9:53 # Os dois DNS Servers que usaremos na porta 53 (DNS)
  - --dns01-recursive-nameservers-only
podDnsPolicy: None
podDnsConfig:
  nameservers: # DNS RelayServers
    - "1.1.1.1" #Cloudflare
    - "9.9.9.9" #Quad9
```
# Instale o CertManager via Helm
```bash
$ helm install cert-manager jetstack/cert-manager --namespace cert-manager --values=values.yaml --version v1.10.0
```
# Crie uma secret com o token da Cloudflare
cloudflare-token-secret.yaml
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: cloudflare-token-secret
  namespace: cert-manager
type: Opaque
stringData:
  cloudflare-token: <Seu token super secreto da CloudFlare>
 ```
 Aplique a secret!
 ```bash
 kubectl apply -f cloudflare-token-secret.yaml
 ```
 # Crie um cluster issuer de staging para validar a sua secret criada anteriormente
 .\issuers\letsencrypt-staging.yaml
 ```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    email: juba@jubagcastro.dev #Seu email para registro no letsEncrypt (pode ser o mesmo da cloudflare)
    privateKeySecretRef:
      name: letsencrypt-staging
    solvers:
      - dns01:
          cloudflare:
            email: juba@jubagcastro.dev #Seu email de login na Cloudflare
            apiTokenSecretRef: #Referência da secret criada anteriormente
              name: cloudflare-token-secret
              key: cloudflare-token
        selector:
          dnsZones:
            - "jubagcastro.dev" #Seu Domínio
 ```
 Aplique o manifesto
 ```bash
 $ kubectl apply -f letsencrypt-staging.yaml
 ```
# Crie um certificado em staging para validar as configurações até agora
.\certificates\cert-staging.yaml
 ```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: home-juba 
  namespace: default
spec:
  secretName: home-juba #Nome que será usado como referência tls nos IngressRoutes
  issuerRef:
    name: letsencrypt-staging
    kind: ClusterIssuer
  commonName: "*.home.jubagcastro.dev" #CommonName igual ao DNS
  dnsNames:
  - "*.home.jubagcastro.dev" #DNS para o certificado, neste caso um wildcard para o subdomínio home.jubagcastro.dev
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: home-juba
  namespace: traefik
spec:
  secretName: home-juba #Nome que será usado como referência tls nos IngressRoutes
  issuerRef:
    name: letsencrypt-staging
    kind: ClusterIssuer
  commonName: "*.home.jubagcastro.dev" #CommonName igual ao DNS
  dnsNames:
  - "*.home.jubagcastro.dev" #DNS para o certificado, neste caso um wildcard para o subdomínio home.jubagcastro.dev
 ```
```bash
 $ kubectl apply -f cert-staging.yaml
 ```
## Valide a propagação do seu certificado com o comando
```bash
 $ kubectl get challenges --all-namespaces -w
 ```
 O parametro -w esperará até que ambos certificados estejam propagados corretamente.

 # Após validar em staging crie em production
 .\issuers\letsencrypt-production.yaml
 ```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-production
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: juba@jubagcastro.dev #Seu email para registro no letsEncrypt (pode ser o mesmo da cloudflare)
    privateKeySecretRef:
      name: letsencrypt-production
    solvers:
      - dns01:
          cloudflare:
            email: juba@jubagcastro.dev #Seu email de login na Cloudflare
            apiTokenSecretRef: #Referência da secret criada anteriormente
              name: cloudflare-token-secret
              key: cloudflare-token
        selector:
          dnsZones:
            - "jubagcastro.dev" #Seu Domínio
 ```
 Aplique o manifesto
 ```bash
 $ kubectl apply -f letsencrypt-production.yaml
 ```

 # Crie seus certificado em production
.\certificates\cert-production.yaml
 ```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: home-juba
  namespace: default
spec:
  secretName: home-juba #Nome que será usado como referência tls nos IngressRoutes
  issuerRef:
    name: letsencrypt-production
    kind: ClusterIssuer
  commonName: "*.home.jubagcastro.dev" #CommonName igual ao DNS
  dnsNames:
  - "*.home.jubagcastro.dev" #DNS para o certificado, neste caso um wildcard para o subdomínio home.jubagcastro.dev
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: home-juba
  namespace: traefik
spec:
  secretName: home-juba #Nome que será usado como referência tls nos IngressRoutes
  issuerRef:
    name: letsencrypt-production
    kind: ClusterIssuer
  commonName: "*.home.jubagcastro.dev" #CommonName igual ao DNS
  dnsNames:
  - "*.home.jubagcastro.dev" #DNS para o certificado, neste caso um wildcard para o subdomínio home.jubagcastro.dev
 ```
```bash
 $ kubectl apply -f cert-production.yaml
 ```
## Valide a propagação do seu certificado com o comando
```bash
 $ kubectl get challenges --all-namespaces -w
 ```
 O parametro -w esperará até que ambos certificados estejam propagados corretamente.

 # Agora o ambiente está preparado para usar os certificados emitidos nos namespaces deafult e traefik
 Como exemplo vamos emitir um certificado para o nosso jenkins que está no namespace automation
 
 Vamos emitir/copiar o certificado que já temos emitido para o namespace automation

 .\certificates\cert-automation.yaml
 ```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: home-juba
  namespace: automation
spec:
  secretName: home-juba
  issuerRef:
    name: letsencrypt-production
    kind: ClusterIssuer
  commonName: "*.home.jubagcastro.dev"
  dnsNames:
  - "home.jubagcastro.dev"
  - "*.home.jubagcastro.dev"
 ```
 E criaremos a service e a ingress route para que o traefik redirecione a rota de entrada para o nosso serviço usando o certificado criado.

 .\ingressRoutes\jenkins.yaml
 ```yaml
apiVersion: v1
kind: Service
metadata:
  name: jenkins
  namespace: automation
spec:
  selector:
    app.kubernetes.io/component: jenkins-master
    app.kubernetes.io/instance: jenkins
  ports:
  - name: http
    targetPort: 8080
    port: 8080
  - name: jnlp
    targetPort: 50000
    port: 50000 
---
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: jenkins
  namespace: automation
  annotations: 
    kubernetes.io/ingress.class: traefik
spec:
  entryPoints:
    - websecure
  routes:
    - match: Host(`www.jenkins.home.jubagcastro.dev`)
      kind: Rule
      services:
        - name: jenkins
          port: 8080
    - match: Host(`jenkins.home.jubagcastro.dev`)
      kind: Rule
      services:
        - name: jenkins
          port: 8080
      middlewares:
        - name: default-headers
  tls:
    secretName: home-juba
```
Aplique o manifesto
```bash
$ kubectl apply -f jenkins.yaml
```
Agora precisamos apenas seguir estes passos para os próximos serviços que iremos subir, na raiz desse git temos um [sample-nginx](https://github.com/Jubagcastro/MetalLB-Traefik-CertManager/tree/master/sample-nginx) que possui os manifestos necessários para usufruir do certificado gerado pelo CertManager, a rota de ingresso provida pelo Traefik, e o balanceamento de carga provida pelo MetalLB.