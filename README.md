1. # Repository klonen von GitHub
git clone https://github.com/luke-z/teko.git

2. # Hyper-V installieren und konfigurieren
2.1  Hyper-V bietet robuste Virtualisierungsfunktionen auf Windows, einschliesslich Netzwerkisolierung und Leistungsmanagement

# Hyper-V aktivieren über PowerShell
Enable-WindowsOptionalFeature -Online -FeatureName Microsoft-Hyper-V -All

# Hyper-V konfigurieren (Hyper-V Manager)
-Manager für virtueller Switches
-Neuer virtueller Netzwerkswitch (Intern)
-Name <MultipassSwitch>
-Internes Netzwerk
-Anwenden + OK

# IP-Adresse setzen für Switch (als Administrator in PowerShell)
New-NetIPAddress -InterfaceAlias 'vEthernet (<MultipassSwitch>)' -IPAddress 192.168.254.10 -PrefixLength 24

IPAddress         : fe80::a721:b981:8482:e670%7
InterfaceIndex    : 7
InterfaceAlias    : vEthernet (MultipassSwitch)
AddressFamily     : IPv6
Type              : Unicast
PrefixLength      : 64
PrefixOrigin      : WellKnown
SuffixOrigin      : Link
AddressState      : Preferred
ValidLifetime     : Infinite ([TimeSpan]::MaxValue)
PreferredLifetime : Infinite ([TimeSpan]::MaxValue)
SkipAsSource      : False
PolicyStore       : ActiveStore

IPAddress         : 192.168.254.10
InterfaceIndex    : 7
InterfaceAlias    : vEthernet (MultipassSwitch)
AddressFamily     : IPv4
Type              : Unicast
PrefixLength      : 24
PrefixOrigin      : Manual
SuffixOrigin      : Manual
AddressState      : Preferred
ValidLifetime     : Infinite ([TimeSpan]::MaxValue)
PreferredLifetime : Infinite ([TimeSpan]::MaxValue)
SkipAsSource      : False
PolicyStore       : ActiveStore

3. # Multipass installieren lokal auf dem Coputer (Windows)
3.1  Multipass bietet eine einfache und schnelle Möglichkeit, Linux-VMs auf Ihrem Windows-System für ein K3s-Cluster zu erstellen

https://multipass.run/download/windows

# Multipass Versionskontrolle (Visual Studio Code schliessen und neu starten)
multipass version
---
multipass   1.13.1+win
multipassd  1.13.1+win
---

# Netzwerkadapter ermitteln 
multipass networks
---
Default Switch           switch     Virtual Switch with internal networking
Ethernet                 ethernet   Intel(R) Ethernet Connection (22) I219-LM
Ethernet 2               ethernet   Intel(R) Ethernet Controller (3) I225-LMvP
MultipassSwitch          switch     Virtual Switch with internal networking
WSL (Hyper-V firewall)   switch     Virtual Switch with internal networking
---

# Netzwerkadapter setzen (vom virtuellen Switch)
multipass set local.driver=hyperv

# Netzwerkadapter überprüfen
multipass get local.driver
---
hyperv
---

4. # Virtuelle Maschinen planen und installieren (CPU, RAM und HDD angepasst auf die Bedürfnisse des Clusters und den lokalen verfügbaren Ressourcen)
multipass launch -n k3s-master -c 2 -m 4G -d 20G --network name=<MultipassSwitch>,mode=manual
multipass launch -n k3s-worker-1 -c 2 -m 4G -d 20G --network name=<MultipassSwitch>,mode=manual
multipass launch -n k3s-worker-2 -c 2 -m 4G -d 20G --network name=<MultipassSwitch>,mode=manual
---
Launched: k3s-master
Launched: k3s-worker-1
Launched: k3s-worker-2
---

# Auflisten aller virtuellen Maschienen
multipass list
---
k3s-master              Running           172.25.145.20    Ubuntu 22.04 LTS
k3s-worker-1            Running           172.25.146.17    Ubuntu 22.04 LTS
k3s-worker-2            Running           172.25.156.102   Ubuntu 22.04 LTS
---

5. # Netzwerke definieren der Nodes

# Verbinden mit den Nodes in der Kommandozeile
multipass shell k3s-master
multipass shell k3s-worker-1
multipass shell k3s-worker-2

# MAC-Adresse einsehen der Netzwerkschnittstelle
ip l

# Öffnen und bearbeiten des Netplans
sudo nano /etc/netplan/99-multipass.yaml

# Netplan erstellen

# k3s-master
network:
    ethernets:
        <eth1>:
            dhcp4: false
            match:
                macaddress: <52:54:00:b3:26:e5>
            set-name: <eth1>
            addresses: [192.168.254.11/24]
    version: 2
---

# k3s-worker-1
network:
    ethernets:
        <eth1>:
            dhcp4: false
            match:
                macaddress: <52:54:00:4d:6b:b1>
            set-name: <eth1>
            addresses: [192.168.254.12/24]
    version: 2
---

# k3s-worker-2
network:
    ethernets:
        <eth1>:
            dhcp4: false
            match:
                macaddress: <52:54:00:a1:97:a9>
            set-name: <eth1>
            addresses: [192.168.254.13/24]
    version: 2
---

# Speichern des Netplans = ctrl+X => ctrl+Y => Enter

# Ansehen des Netplans
cat /etc/netplan/99-multipass.yaml

# Anwenden des Netplans
sudo netplan apply

# Auflistung der Virtuellen Maschinen (Kontrolle der zugewisenen IP-Adresse)
multipass list
---
Name                    State             IPv4             Image
k3s-master              Running           172.30.144.52    Ubuntu 22.04 LTS
                                          172.30.152.181
                                          192.168.254.11
k3s-worker-1            Running           172.30.157.252   Ubuntu 22.04 LTS
                                          172.30.147.17
                                          192.168.254.12
k3s-worker-2            Running           172.30.153.151   Ubuntu 22.04 LTS
                                          192.168.254.13
---

6. # K3s auf dem Master-Node installieren (PowerShell als Administrator)
6.1 K3s ist eine leichtgewichtige Kubernetes-Distribution, ideal für Entwicklungsumgebungen und kleinere Produktionsumgebungen

multipass exec k3s-master -- bash -c "curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC='--node-<ip=192.168.254.11> --cluster-init --disable servicelb' sh -"

# K3s-Token vom Master-Node abrufen (PowerShell als Administrator)
multipass exec k3s-master -- sudo cat /var/lib/rancher/k3s/server/node-token
---
Token = <K1003830fad771e67dbe3b96c3d3a0f924c4229891a192a3838327409237722da7f::server:2f6df2709b1602fb1d975d799984da26>
---

# K3s auf dem Worker-Node-1 installieren (PowerShell als Administrator)
multipass exec k3s-worker-1 -- bash -c "curl -sfL https://get.k3s.io | K3S_URL=https://<192.168.254.11>:6443 K3S_TOKEN=<TOKEN> INSTALL_K3S_EXEC='--node-ip=<192.168.254.12>' sh -"

# K3s auf dem Worker-Node-2 installieren (PowerShell als Administrator)
multipass exec k3s-worker-2 -- bash -c "curl -sfL https://get.k3s.io | K3S_URL=https://<192.168.254.11>:6443 K3S_TOKEN=<TOKEN> INSTALL_K3S_EXEC='--node-ip=<192.168.254.13>' sh -"

7. # K3S-Kontext aktivieren

# Config-File anpassen lokal (öffnen in VS-Code)
C:\Users\<lokalerUser>\.kube

# Confi-File ansehen auf K3S-Master
multipass shell k3s-master
sudo cat /etc/rancher/k3s/k3s.yaml

# Anpassungen vornehmen im Config-File lokal
---
- cluster:
    certificate-authority-data: <authority-data>
    server: https://<IP k3s-master>:6443
  name: <k3s>
---

---
- context:
    cluster: <k3s>
    user: <k3s>
  name: <k3s>
current-context: <k3s>
---

---
- name: <k3s>
  user:
    client-certificate-data: <certificate-data>
    client-key-data: <key-data>
---

# Wechseln in Kontex
kubectl config set-context <NAME>

# Kontrolle ob Nodes in K3S-Kontex vorhanden sind
kubectl get nodes
---
NAME           STATUS   ROLES                       AGE   VERSION
k3s-master     Ready    control-plane,etcd,master   54m   v1.28.6+k3s2
k3s-worker-1   Ready    <none>                      39m   v1.28.6+k3s2
k3s-worker-2   Ready    <none>                      36m   v1.28.6+k3s2
---

8. Installation von Kube-VIP auf dem Master-Node
8.1 Kube-VIP ermöglicht es im Cluster, eine virtuelle IP-Adresse für den hochverfügbaren Zugriff zu nutzen

# Verbinden mit dem Master Node
multipass shell k3s-master

# Verbinden mit dem Master Node in der RootShell
sudo -i

# Umgebungsvariable setzen
export INTERFACE=eth1
export VIP=192.168.254.15

# Kontrolle
echo $VIP
---
192.168.254.15
---


# RBAC-Ressourcen für Kube-VIP installieren
8.1 Bei den RBAC-Ressourcen für kube-vip werden spezifische Zugriffsregeln im Kubernetes-Cluster eingerichtet, die definieren, welche Berechtigungen kube-vip für den Betrieb benötigt

curl -sfL https://kube-vip.io/manifests/rbac.yaml > /var/lib/rancher/k3s/server/manifests/kube-vip-rbac.yaml
kubectl apply -f /var/lib/rancher/k3s/server/manifests/kube-vip-rbac.yaml
---
serviceaccount/kube-vip created
clusterrole.rbac.authorization.k8s.io/system:kube-vip-role created
clusterrolebinding.rbac.authorization.k8s.io/system:kube-vip-binding created
---

# Kube-VIP-Daemonset-Manifest generieren und anwenden
kubectl run -it kube-vip-init --image=ghcr.io/kube-vip/kube-vip:main --restart=Never --rm -- manifest daemonset \
    --interface $INTERFACE \
    --address $VIP \
    --inCluster \
    --taint \
    --controlplane \
    --arp \
    --leaderElection > /var/lib/rancher/k3s/server/manifests/kube-vip-daemonset.yaml

# Überprüfen und Bearbeiten des generierten Manifests
nano /var/lib/rancher/k3s/server/manifests/kube-vip-daemonset.yaml

# Yaml-Datei anpassen
---
app.kubernetes.io/version: v0.6.4 (Version v0.7.0 anpassen)
app.kubernetes.io/version: v0.6.4 (Version v0.7.0 anpassen)
pod "kube-vip-init" deleted (Zeile löschen)
---

# Kube-VIP-Manifest anwenden
kubectl apply -f /var/lib/rancher/k3s/server/manifests/kube-vip-daemonset.yaml

# Status von Kube-VIP prüfen
kubectl get pods -n kube-system
---
NAME                                      READY   STATUS      RESTARTS   AGE
coredns-6799fbcd5-scj9k                   1/1     Running     0          102m
helm-install-traefik-crd-wh8v4            0/1     Completed   0          102m
helm-install-traefik-hpxfl                0/1     Completed   1          102m
kube-vip-ds-8nscx                         1/1     Running     0          6m41s
local-path-provisioner-84db5d44d9-jf9fq   1/1     Running     0          102m
metrics-server-67c658944b-zrrmj           1/1     Running     0          102m
traefik-f4564c4f4-p5r8k                   1/1     Running     0          102m
---

9. Kubeconfig Datei erhalten vom erstellten K3S-Cluster und lokalen verwenden auf dem Rechner

# Chocolatey installieren (PowerShell)
Set-ExecutionPolicy Bypass -Scope Process -Force; [System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072; iex ((New-Object System.Net.WebClient).DownloadString('https://community.chocolatey.org/install.ps1'))

# kubectx mit Chocolatey installieren (PowerShell)
choco install kubens kubectx

# Kontrolle
kubens                      kubectx
---                         ---
default                     docker-desktop
kube-node-lease             k3s
kube-public                 ---
kube-system
---

# Backup Datei erstellen
C:\Users\UserZero\.kube
config
öffnen in VS-Code
Speichern als Backup-Datei in VS-Code Explorer als .yaml Datei

10. Yaml-Files erstellen und lokal speichern

# ipaddresspool.yaml (C:\Users\UserZero\.kube)
---
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: default
  namespace: metallb-system
spec:
  addresses:
  - 192.168.254.16-192.168.254.20
---

# l2advertisement.yaml (C:\Users\UserZero\.kube)
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: default
  namespace: metallb-system
spec:
  ipAddressPools:
  - default
  interfaces:
  - eth1
---

11. MetalLB installieren (PowerShell als Administrator)
choco install kubernetes-helm
---
The install of kubernetes-helm was successful.
Software installed to 'C:\ProgramData\chocolatey\lib\kubernetes-helm\tools
---

# MetalLB Repository zu Helm hinzufügen (PowerShell als Administrator)
helm repo add metallb https://metallb.github.io/metallb
---
metallb" has been added to your repositories
---

# Helm Repository aktualisieren
helm repo update
---
Update Complete. ⎈Happy Helming!⎈
---

# MetalLB installieren mittels Helm
helm upgrade --install metallb metallb/metallb --namespace metallb-system --create-namespace
---
MetalLB is now running in the cluster
---

# ipaddresspool und l2advertisement anwenden
cd /c/Users/UserZero/.kube
kubectl apply -f ipaddresspool.yaml
kubectl apply -f l2advertisement.yaml
---
ipaddresspool.metallb.io/default created
l2advertisement.metallb.io/default created
---

# Überprüfen der Installation
kubectl get pods -n metallb-system

12. nfs-common installieren auf jeder VM

# Zugriff auf VM'S
multipass shell k3s-master
multipass shell k3s-worker-1
multipass shell k3s-worker-2

# nfs-common installieren auf VM'S
sudo apt update
sudo apt install nfs-common
---
All packages are up to date
Running kernel seems to be up-to-date
No services need to be restarted
No containers need to be restarted
No user sessions are running outdated binaries
No VM guests are running outdated hypervisor (qemu) binaries on this host
---

13. Longhorn installieren
13.1  Longhorn ermöglicht es, persistenten und hochverfügbaren Speicher für die Kubernetes-Workloads einfach zu verwalten

# Installationsbefehl
kubectl apply -f https://raw.githubusercontent.com/longhorn/longhorn/v1.5.3/deploy/longhorn.yaml

# Überprüfen der Installation
kubectl get pods -n longhorn-system

# Zugriff auf Longhorn UI (Weboberfläche)
kubectl -n longhorn-system port-forward service/longhorn-frontend 8080:80

14. Installation von Lens Desktop Personal (Übersicht des Clusters)
https://k8slens.dev/

15. Containerisierung und Deployment der Beispielapplikation

# Beispielaktion klonen auf lokalen Rechner
git clone https://github.com/SwitzerChees/ms-phonebook-example

# Dockerfile anpassen result_server
docker build -t result-server-app ./result_server
---
Sichtbar auf Docker Desktop
result-server-app
e90069824e32e4d440d706ec450a80effad9bc8e32835a86af5e6051f7e26395
---

# result-server-app Image taggen
docker tag result-server-app brunnerlan/result-server-app:latest
---
Sichtbar auf Docker Desktop
brunnerlan/result-server-app
e90069824e32e4d440d706ec450a80effad9bc8e32835a86af5e6051f7e26395
---

# Einloggen auf DockerHub
docker login
---
Authenticating with existing credentials...
Login Succeeded
---

# Image auf Docker Hub hochladen
docker push brunnerlan/result-server-app:latest
---
The push refers to repository [docker.io/brunnerlan/result-server-app]
341e7b046bfc: Pushed
cdf3ed1d2b5a: Pushed
ce4662f8aa53: Pushed
2e7a6cef9202: Pushed
6e751b5b0bee: Mounted from library/python
126cdbcad241: Mounted from library/python
01589a17de95: Mounted from library/python
84f540ade319: Mounted from library/python
9fe4e8a1862c: Mounted from library/python
909275a3eaaa: Mounted from library/python
f3f47b3309ca: Mounted from library/python
1a5fc1184c48: Mounted from library/python
latest: digest: sha256:311f95c82c571bc18ebf3f081a06efa9b0ccc5b79a68ca8315a0293668a35b18 size: 2839
---

# Dockerfile anpassen web_server 
docker build -t web-server-app ./web_server
---
Sichtbar auf Docker Desktop
web-server-app
966ef210de1c5d989e2b055578a98cd9182c3f256287241800492e7966a6c7fe
---

# web-server-app Image taggen
docker tag web-server-app brunnerlan/web-server-app:latest
---
Sichtbar auf Docker Desktop
brunnerlan/web-server-app
966ef210de1c5d989e2b055578a98cd9182c3f256287241800492e7966a6c7fe
---

# Image auf Docker Hub hochladen
docker push brunnerlan/web-server-app:latest
---
The push refers to repository [docker.io/brunnerlan/web-server-app]
73d709d9808b: Pushed
cdf3ed1d2b5a: Mounted from brunnerlan/result-server-app
ce4662f8aa53: Mounted from brunnerlan/result-server-app
2e7a6cef9202: Mounted from brunnerlan/result-server-app
6e751b5b0bee: Mounted from brunnerlan/result-server-app
126cdbcad241: Mounted from brunnerlan/result-server-app
01589a17de95: Mounted from brunnerlan/result-server-app
84f540ade319: Mounted from brunnerlan/result-server-app
9fe4e8a1862c: Mounted from brunnerlan/result-server-app
909275a3eaaa: Mounted from brunnerlan/result-server-app
f3f47b3309ca: Mounted from brunnerlan/result-server-app
1a5fc1184c48: Mounted from brunnerlan/result-server-app
latest: digest: sha256:5afa999964e1ec749899ecbdc8d60f9ab314ea6a33be781a824ebd54a089bce1 size: 2839
---

# Namespace my-apps erstellen
kubectl create namespace my-apps
---
namespace/my-apps created
---

# Kontrolle ob Namespace vorhanden ist
kubectl get namespaces my-apps
---
NAME      STATUS   AGE
my-apps   Active   76s
---

# Anwenden der deployments und services
kubectl apply -f result-server-app-deployment.yaml
kubectl apply -f result-server-app-service.yaml
kubectl apply -f web-server-app-deployment.yaml
kubectl apply -f web-server-app-service.yaml
---
deployment.apps/result-server-app created
service/result-server-app created
deployment.apps/web-server-app created
service/web-server-app created
---

# Überprüfen der Deployments
kubectl get deployments -n my-apps
---
NAME                READY   UP-TO-DATE   AVAILABLE   AGE
result-server-app   2/2     2            2           51s
web-server-app      2/2     2            2           41s
---

# Überprüfen der Services
kubectl get services -n my-apps
---
NAME                TYPE           CLUSTER-IP      EXTERNAL-IP      PORT(S)        AGE
result-server-app   LoadBalancer   10.43.129.241   192.168.254.17   80:31184/TCP   2m10s
web-server-app      LoadBalancer   10.43.107.193   192.168.254.18   80:30497/TCP   119s
---

# Überprüfen der Pods
kubectl get pods -n my-apps
---
NAME                                 READY   STATUS    RESTARTS        AGE
result-server-app-7db754f74f-hqm5d   1/1     Running   0               7m38s
result-server-app-7db754f74f-md55c   1/1     Running   0               7m38s
web-server-app-69bbdd555b-8s52n      1/1     Running   6 (2m55s ago)   7m28s <MYSQL wurde noch nicht erstellt>
web-server-app-69bbdd555b-q6d6p      1/1     Running   6 (2m55s ago)   7m28s <MYSQL wurde noch nicht erstellt>
---

16. # Erstellen des Speichers (MYSQL)

# PersistentVolumeClaim für MySQL erstellen
mysql-pvc.yaml

# Anwenden des PersistentVolumeClaims
kubectl apply -f mysql-pvc.yaml
---
persistentvolumeclaim/mysql-pvc created
---

# Überprüfen, ob der PVC erfolgreich erstellt wurde
kubectl get pvc -n my-apps
---
NAME        STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
mysql-pvc   Bound    pvc-14da02f2-171f-404b-8eab-490d35f476f0   1Gi        RWO            longhorn       36h
---

# MySQL Deployment erstellen
mysql-deployment.yaml

# MySQL-Deployment anwenden
kubectl apply -f mysql-deployment.yaml
---
deployment.apps/mysql created
---

# MySQL Service erstellen
mysql-service.yaml

# MySQL-Service anwenden
kubectl apply -f mysql-service.yaml
---
service/mysql-service created
---

# Überprüfen, ob das Deployment und der Service erfolgreich erstellt wurden
kubectl get deployments,svc -n my-apps
---
NAME                                READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/mysql               1/1     1            1           36h
deployment.apps/result-server-app   2/2     2            2           2d13h
deployment.apps/web-server-app      2/2     2            2           36h

NAME                        TYPE           CLUSTER-IP      EXTERNAL-IP      PORT(S)        AGE
service/mysql-service       ClusterIP      10.43.87.227    <none>           3306/TCP       2d13h
service/result-server-app   LoadBalancer   10.43.129.241   192.168.254.17   80:31184/TCP   2d13h
service/web-server-app      LoadBalancer   10.43.107.193   192.168.254.18   80:30497/TCP   2d13h
---

# UI-Zugriff auf Longhorn
kubectl -n longhorn-system port-forward service/longhorn-frontend 8080:80
http://localhost:8080

17. # Ingresscontroller implementieren

# web-server-ingress.yaml Datei hinzufügen und erstellen
kubectl apply -f web-server-ingress.yaml
kubectl apply -f result-server-ingress.yaml

# Host-Datei anpassen lokal (als Administrator)
C:\Windows\System32\drivers\etc\hosts
---
192.168.254.16 meinwebserver.manuel.com
192.168.254.16 meinresultserver.manuel.com
---

# Zugriffstest
curl http://meinwebserver.manuel.com
curl http://meinresultserver.manuel.com
---
Antwort von Browser
---

# web-app aufrufen im Browser
http://meinwebserver.manuel.com
http://meinresultserver.manuel.com

18. # Prometheus

# K8s-monitoring klonen auf lokalen Rechner
git clone https://github.com/SwitzerChees/k8s-monitoring

# Bitnami-Helm-Repository hinzufügen
helm repo add bitnami https://charts.bitnami.com/bitnami
---
"bitnami" has been added to your repositories
---

# Helm-Repository-Liste aktualisieren
helm repo update
---
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "metallb" chart repository
...Successfully got an update from the "bitnami" chart repository
Update Complete. ⎈Happy Helming!⎈
---

# Prometheus mit Helm installieren
helm upgrade --install kube-prometheus bitnami/kube-prometheus --namespace kube-prometheus --create-namespace -f kube-prometheus/values.yml
---
Release "kube-prometheus" does not exist. Installing it now.
NAME: kube-prometheus
LAST DEPLOYED: Tue Mar  5 20:52:47 2024
NAMESPACE: kube-prometheus
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
CHART NAME: kube-prometheus
CHART VERSION: 8.29.1
APP VERSION: 0.72.0
---

# Zugriff auf Prometheus
kubectl port-forward --namespace kube-prometheus svc/kube-prometheus-prometheus 9090:9090

# Prometheus in Webbrowser abrufen
http://127.0.0.1:9090/

19. # Grafana

# Grafana installieren
helm upgrade --install grafana bitnami/grafana --namespace grafana --create-namespace
---
Release "grafana" does not exist. Installing it now.
NAME: grafana
LAST DEPLOYED: Tue Mar  5 21:27:05 2024
NAMESPACE: grafana
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
CHART NAME: grafana
CHART VERSION: 9.11.1
APP VERSION: 10.3.3
---

# Zugriff auf Grafana
kubectl port-forward --namespace grafana svc/grafana 3000:3000

# Grafana in Webbrowser abrufen
http://localhost:3000

## Username
admin
## Password
PvoKaHw8OM

# Connections
# Add new connections
# Data source
# Prometheus server URL
http://kube-prometheus-prometheus.kube-prometheus.svc.cluster.local:9090

# Dashboard implementieren von Grafana
https://grafana.com/grafana/dashboards/

# Problematik
-heufiger Neustart der VM's

# Manuels push
git init // Initialisiert das Repository
git add . // Alle Daten hinzugügen
git commit -m "Meine Nachricht" // Kommentar hinzufügen
git remote add origin https://github.com/BrunnerLan/k3s-kubernetes-cluster.git // URL dem Repository hinzufügen
git push -u origin master // Auf GitHub pushen
# k3s-kubernetes-cluster
# k3s-kubernetes-cluster
