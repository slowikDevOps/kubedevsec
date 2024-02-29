# KubeDevSec! Wdrożenie on-premise z wykorzystaniem Kubespray, MetalLB, Vault
##### Autor: Szymon Słowicki

https://devops.sl0vik.online


Wymagania wstępne:

#### ansible, helm, python, kubectl, git.

Jeżeli bedzie brakować jakiś paczek, kubespray upomni się o nie. 


#### Rekomendacje dla użytkowników bez doświadczenia z Linuksem:

- Windows: WSL2
- MacOS: ZSH

### Kilka słów od autora



*Niechciałbym aby ten projekt, był **"projektem-corowym"** dla wszystkich, którzy zdecydują się go wdrożyć. Zaprojektowałem go tak, aby nic innego poza plikiem `index.html` nie było montowane przez **Vault init container**. Jak rozwinąć swój projekt?*

- *Monitoring (prometheus + grafana) i  mailowe alerty. Jeżeli jesteś bardziej ambitny, możesz pokusić się o monitorowanie Twojej aplikacji. np. wykorzystanie CPU/RAM* 

- *Potrzebujesz supportu? Pisz na LI, chętnie pomogę! itd :) 
Szukasz/nie możesz znaleźć pracy? Napisz, podkręcimy Twoje CV, sprawdzę jakich skilli Ci brakuje.*

### BONUS: Gdy pójdzie coś nie tak -> restart klastra:


`ansible-playbook -i inventory/mycluster/hosts.yaml  --become --become-user=root reset.yml`

Przy restarcie musiałem restartować VM z poziomu panelu Hetznera i puścić playbook jeszcze raz.

#### Postawienie klastra (nie pomyl z restartem )

`ansible-playbook -i inventory/mycluster/hosts.yaml  --become --become-user=root cluster.yml`


## PIERWSZY KOMPONENT 1. KUBESPRAY DO STAWIANIA KLASTRA

Pamiętaj, żeby użyć własnych adresów IP w kroku nr IV. 

```
1. git clone https://github.com/kubernetes-sigs/kubespray.git

2. cd kubespray

3. cp -rfp inventory/sample inventory/mycluster

4. declare -a IPS=(135.181.194.175 37.27.32.146 65.108.156.34)

5. CONFIG_FILE=inventory/mycluster/hosts.yaml python3 contrib/inventory_builder/inventory.py ${IPS[@]}

```

*otrzymałem błąd, musiałem dorzucić brakujący pakiet i wykonać powyższą komendę ponownie.*

`pip3 install ruamel.yaml`

6. W lokalizacji `kubespray/inventory/mycluster/group_vars/k8s_cluster` modyfikujemy `plik k8s-cluster.yml` w lini 70!

Zmieniamy **calico** na **cilium**!

Tutaj można sprawdzić dlaczego calico gryzię się z MetalLB:
https://metallb.universe.tf/installation/network-addons/
https://metallb.universe.tf/configuration/calico/


7. Nie chcemy dwóch masterów, z lokalizacji `kubespray/inventory/mycluster` modyfikujemy linię 19 w pliku `hosts.yaml`, komentując ją. 

*# node2:*

8. #### Playbook musi być odpalony jako root! 

Moja struktura pliku config w katalogu .ssh
```
Host *
        IdentityFile ~/.ssh/id_rsa
        User root

```
9. Uruchamiamy playbook'a, który postawi klaster:

`ansible-playbook -i inventory/mycluster/hosts.yaml  --become --become-user=root cluster.yml`


10. Kopiuje do siebie .kubeconfig



`scp root@135.181.194.175:/root/.kube/config /home/slovik/.kube/moje-configi/testowy-kubedevsec` # pamiętaj, że będziesz miał inne IP :) 


`cat moje-configi/testowy-kubedevsec > config`

`podmieniam IP: 135.181.194.175` # pamiętaj, że będziesz miał inne IP :)  - nie będę o tym przypominał w kolejnych krokach! 


## DRUGI KOMPONENT 2. NGINX INGRESS CONTROLLER 


1.
```
helm upgrade --install ingress-nginx ingress-nginx \
  --repo https://kubernetes.github.io/ingress-nginx \
  --namespace ingress-nginx --create-namespace
```


## TRZECI KOMPONENT 3. METALLB 


`helm install metallb metallb/metallb --create-namespace -n metallb-system`

1. config, przechodzimy do katalogu official-docs/metealLB

`cd /official-docs/metealLB`

`kubectl apply -f config.yaml`


## CZWARTY KOMPONENT 4. VAULT 

Przechodzimy do katalogu:
`cd vault-manifests`

Ogarniamy pv i storage class i ingressa:

`kubectl create namespace vault` 

`kubectl apply -f pv.yaml`

`kubectl apply -f storageclass.yaml`

`kubectl apply -f ingress.yaml`

**tworzę sekrety z certem:**

`cd wildcard/sl0vik.online`

`kubectl create secret generic sl0vik-wildcard --from-file=tls.crt=fullchain.pem --from-file=tls.key=privkey.pem --namespace vault`


Nie używaj moich plików, tylko wygeneruj własne! Jak? 

https://www.linkedin.com/pulse/wildcard-certificates-using-lets-encrypt-certbot-pallavi-udhane/

Odpalam vaulta z helma, w values.yaml wskazuje na storageclass **vault** i mam włączony **vault-ui**:

`cd official-docs/vault-helm`

`helm upgrade --install vault hashicorp/vault --create-namespace -n vault -f values.yaml` 

**Teraz mamy do wykonania komendy w vaulcie!**


`vault operator init`

W wyniku czego otrzymamy:

```
Unseal Key 1: X869Z0ogUSCrhW1tqweqweqweqweqweqweqw0cGO6iT9
Unseal Key 2: Wtoi0EuvPiqweqweqweqweqweqweq/6dHzBapYxQUjki
Unseal Key 3: Dvsadasdasdasdasdadadadadadadadadadadadadssx
Unseal Key 4: gdfgdfgdfgdfgdfgdfgdfgdfgdfgdfgdfgdfgdfgoQO2
Unseal Key 5: fsdfsdfsfw3ef23rfwefwf23fsdfsdf24t24gegfdaak

*Initial Root Token: hvs.b6Ni7vdasdasdasdasdasdasdI*
```

*Otrzymasz powyższe dane! Zapisz je w bezpiecznym miejscu i nikomu nie udostępniaj! Powyższe są przykładowe :) 
Jeżeli otrzymasz błąd, sprawdź na którym worker node uruchomił się vault, i dla katalogu /vault wykonaj:*

`chmod -R 777 /vault` i spróbuj ponownie! 

następnie `vault operator unseal` (musisz wykonać trzy razy podając za każdym razem inny **Unseal Key**)

`vault login` (klikasz enter i następnie wprowadzasz token) np. hvs.b6Ni7vdasdasdasdasdasdasdI


Vault jest dostępny pod URL: vault.sl0vik.online





## PIĄTY KOMPONENT 5. APLIKACJA 


**W pierwszej kolejności działamy w podzie vaulta**


`vault operator unseal`
Komendę wykonujemy trzy razy podając inny klucz! 

`vault login`


`cat <<EOF > /home/vault/app-policy.hcl
path "secret*" {
  capabilities = ["read"]
}
EOF`

`vault policy write app /home/vault/app-policy.hcl`


`vault auth enable kubernetes`

`vault write auth/kubernetes/config \
   token_reviewer_jwt="$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" \
   kubernetes_host=https://${KUBERNETES_PORT_443_TCP_ADDR}:443 \
   kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt`

`vault write auth/kubernetes/role/myapp \
   bound_service_account_names=app \
   bound_service_account_namespaces=stronka \
   policies=app \
   ttl=1h`


`vault secrets enable -path=secret kv-v2`

`vault kv put secret/html index=value1`


`cd stronka`

następnie `kubectl apply -f  stronka.yaml`


`cd wildcard/sl0vik.online`

`kubectl create secret generic sl0vik-wildcard --from-file=tls.crt=fullchain.pem --from-file=tls.key=privkey.pem --namespace stronka`


Finito! Jeżeli masz pytania, wiesz gdzie mnie szukać! 
