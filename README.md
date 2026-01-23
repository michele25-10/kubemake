# Kubemake

## Documentazione

- [Documentazione kubemake](/documentations/kubemake.md)
- [Documentazione groups e roles](/documentations/roles.md)

---

# Configurazione

## File di inventory: `hosts.yml`

```yml
[masters]
master ansible_user=ubuntu ansible_host=192.168.1.1 \
kubelet_node_ip=192.168.1.1 \              # IP che il kubelet userà come suo indirizzo nel cluster
apiserver_advertise_address=192.168.1.1 \ # IP che l’API server annuncia agli altri nodi
apiserver_cert_extra_sans=172.16.100.100  # SAN extra per il certificato TLS dell’API server (utile se accedi da altro IP)

[workers]
worker-1 ansible_user=ubuntu ansible_host=192.168.1.2 \
kubelet_node_ip=192.168.1.2 \             # IP del nodo worker nel cluster
crio_os=xUbuntu_22.04 \                   # Variante OS per CRI-O (serve ai ruoli CRI-O)
cni_arch=amd64                            # Architettura della CNI, utile per scaricare binari corretti

worker-2 ansible_user=ubuntu ansible_host=192.168.1.3 \
kubelet_node_ip=192.168.1.3 \
crio_os=xUbuntu_22.04 \
cni_arch=amd64

[all:vars]
ansible_ssh_private_key_file=/path/to/your/pem   # Chiave SSH privata per connettersi ai nodi
crio_version="1.28"                              # Versione CRI-O da installare
cri_socket="unix:/run/crio/crio.sock"            # Socket usato da kubelet per parlare con CRI-O
cni_version="v1.4.0"                             # Versione del CNI plugin da installare
kubernetes_version_repo="v1.29"                  # Versione Kubernetes da scaricare dai repo ufficiali
kubernetes_os="deb"                              # OS packaging, qui Debian/Ubuntu (.deb)
kubernetes_version_pkg="1.29.9-1.1"             # Versione specifica dei pacchetti kubeadm, kubelet, kubectl

[masters:vars]
pod_network_cidr=10.244.0.0/16                  # CIDR della rete pod (necessario per flannel)
flannel_iface_regex=[eth1|eth0]                 # Interfacce da usare per flannel, regex per scegliere la corretta
```

## (Un)Make

Per installare un cluster Kubernetes, lancia:

```bash
ansible-playbook site.yaml -i hosts.ini
```

Per rimuovere tutto ciò che è stato installato dal playbook precedente, usa il rollback:

```bash
ansible-playbook rollback-site.yaml -i hosts.ini
```

Questo annulla tutte le modifiche apportate ai nodi dal playbook “site.yaml”, riportandoli allo stato iniziale.

## Tag

Se vuoi eseguire solo un sottoinsieme di operazioni, puoi usare i tag definiti nei playbook:

| Tag     | Cosa fa                                                             |
| ------- | ------------------------------------------------------------------- |
| `setup` | Installa tutti i pacchetti e componenti necessari per Kubernetes    |
| `init`  | Inizializza il master Kubernetes con kubeadm e networking (flannel) |
| `join`  | Fa sì che i worker si uniscano al master                            |

```bash
ansible-playbook rollback-site.yml -i hosts -t setup,init,join,chaos

ansible-playbook site.yaml -i hosts.ini --tags setup
ansible-playbook site.yaml -i hosts.ini --tags init
ansible-playbook site.yaml -i hosts.ini --tags join
```

# Playbook

### `site.yml`

```yml
---
- hosts:
    - masters
    - workers
  become: yes
  tags:
    - setup
  roles:
    - common
    - crio
    - k8s/common
    - cni

- hosts: masters
  become: yes
  tags:
    - init
  roles:
    - k8s/init
    - flannel

- hosts: workers
  become: yes
  tags:
    - join
  roles:
    - k8s/join
```

### `rollback-site.yml`

```yml
---
- hosts: workers
  become: yes
  tags:
    - join
  roles:
    - rollback/k8s/join

- hosts: masters
  become: yes
  tags:
    - init
  roles:
    - rollback/k8s/init
    - rollback/flannel

- hosts: all
  become: yes
  tags:
    - setup
  roles:
    - rollback/cni
    - rollback/k8s/common
    - rollback/crio
    - rollback/common
```

---

# Risultato atteso

Dopo aver lanciato:

```bash
ansible-playbook site.yaml -i hosts.ini
```

Se ti colleghi via SSH sul master e controlli i pod:

```bash
kubectl get pods --all-namespaces
```

Risultato atteso:

```text
kube-flannel   kube-flannel-ds-xxxxxxx   1/1     Running   <...>
kube-flannel   kube-flannel-ds-yyyyyyy   1/1     Running   <...>
kube-system    coredns-xxxxxxx           1/1     Running   <...>
kube-system    coredns-yyyyyyy           1/1     Running   <...>
kube-system    etcd-master               1/1     Running   <...>
kube-system    kube-apiserver-master     1/1     Running   <...>
kube-system    kube-controller-manager   1/1     Running   <...>
kube-system    kube-proxy-xxxxxx         1/1     Running   <...>
kube-system    kube-proxy-yyyyyy         1/1     Running   <...>
kube-system    kube-scheduler-master     1/1     Running   <...>
```

Spiegazione:

- kube-flannel → il daemonset del networking Flannel per la comunicazione tra pod
- kube-system/coredns → servizio DNS del cluster
- etcd, kube-apiserver, controller-manager, scheduler → componenti core del master
- kube-proxy → gestisce il networking sui nodi worker

### Verifica dei nodi

Controlla lo stato dei nodi:

```bash
kubectl get nodes -o wide
```

Risultato atteso:

```text
NAME      STATUS   ROLES    AGE   VERSION    INTERNAL-IP
master    Ready    master   ...   v1.29.9   192.168.1.1
worker-1  Ready    <none>   ...   v1.29.9   192.168.1.2
worker-2  Ready    <none>   ...   v1.29.9   192.168.1.3
```
