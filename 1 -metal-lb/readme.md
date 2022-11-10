## GitHub MetalLB: https://github.com/metallb/metallb/tree/v0.9.5
## Manifestos: https://github.com/metallb/metallb/tree/v0.9.5/manifests

# Primeiro, crie o namespace:
```bash
$ kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.9.5/manifests/namespace.yaml
```
# Faça o deployment do MetalLB:
```bash
$ kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.9.5/manifests/metallb.yaml
```
# Crie uma secret usando OpenSSL (caso não tenha, instale openssl) no namespace metallb-system criado no primeiro passo
```bash
$ kubectl create secret generic -n metallb-system memberlist --from-literal=secretkey="$(openssl rand -base64 128)"
```
# Aplique o configmap local como o exemplo:
Neste exemplo estamos separando uma faixa de IP fora de nosso DHCP na rede interna para que o MetalLB faça o balanceamento nessa faixa, para que não ocorra conflitos de IP, neste exemplo do IP ao 10.1.1.201 até o 10.1.1.220.

configmap.yaml
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
    - name: default
      protocol: layer2
      addresses:
      - 10.1.1.201-10.1.1.220        
```