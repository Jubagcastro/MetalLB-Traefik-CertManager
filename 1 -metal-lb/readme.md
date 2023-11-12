## GitHub MetalLB: https://github.com/metallb/metallb/tree/v0.9.5
## Manifestos: https://github.com/metallb/metallb/tree/v0.9.5/manifests

# Faça o deployment do MetalLB:
```bash
❯ kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.7/config/manifests/metallb-native.yaml
```
# Aplique o ipAddressPool local como o exemplo:
Neste exemplo estamos separando uma faixa de IP fora de nosso DHCP na rede interna para que o MetalLB faça o balanceamento nessa faixa, para que não ocorra conflitos de IP, neste exemplo do IP ao 10.1.1.201 até o 10.1.1.220.

configmap.yaml
```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: config
  namespace: metallb-system
spec:
  addresses:
  - 10.244.0.50-10.244.0.150
```
# Make some Coffee