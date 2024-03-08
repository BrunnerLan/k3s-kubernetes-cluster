# Semesterarbeit DannyUndManuel

## K3s Kubernete Cluster

### Anleitung
0. # Hyper-V  aktivieren (Systemsteuerung)

## Hyper-V als Hypervisor konfigurieren
### Manager für virtuelle Switches
#### Neuer virtueller Netzwerkswitch (intern)
##### Switch benennen (multipass)

1. # Multipass installieren lokal auf dem Coputer (Windows)
https://multipass.run/download/windows

2. # Welche Multipass Version ist installiert? (Kontrolle)
multipass version
## multipass   1.13.0+win
## multipassd  1.13.0+win

3. # Netzwerkadapter ermitteln 
multipass networks
## Default Switch           switch     Virtual Switch with internal networking
## Ethernet 8               ethernet   Surface Ethernet Adapter #3
## WSL (Hyper-V firewall)   switch     Virtual Switch with internal networking
## multipass                switch     Virtual Switch with internal networking

4. # Netzwerkadapter setzen (vom virtuellen Switch) über Powershell
multipass set local.driver=hyperv

5. # Netzwerkadapter überprüfen
multipass get local.driver
## hyperv

6. # Virtuelle Maschinen planen (CPU, RAM und HDD angepasst auf die Bedürfnisse des Clusters so, wie die lokalen verfügbaren Ressourcen sind)

7. # VM-MasterNode erstellen -c=CPU, -m=RAM, -d=Festplattenspeicher
multipass launch -n k3s-master -c 2 -m 2G -d 20G --bridged

8. # VM-WorkerNode1 erstellen -c=CPU, -m=RAM, -d=Festplattenspeicher
multipass launch -n k3s-worker-1 -c 2 -m 2G -d 20G --bridged

9. # VM-WorkerNode2 erstellen -c=CPU, -m=RAM, -d=Festplattenspeicher
multipass launch -n k3s-worker-2 -c 2 -m 2G -d 20G --bridged

10. # Auflisten aller virtuellen Maschienen (Name, Status, IP, Image)
multipass list

11. # Netzwerke definieren der Nodes

## Verbinden mit dem MasterNode in der Kommandozeile
multipass shell k3s-master

## MAC-Adresse einsehen der Netzwerkschnittstelle
ip l

## Öffnen und bearbeiten des Netplans
sudo nano /etc/netplan/master-multipass.yaml

## Netplan erstellen (Vorsicht "<>" Platzhalter)
## k3s-master
network:
    ethernets:
        eth1:
            dhcp4: false
            match:
                macaddress: 52:54:00:39:17:05
            set-name: eth1
            addresses: [192.168.10.10/24]
    version: 2

## ctrl+X => ctrl+Y => Enter = Speichern des Netplans

## Ansehen des Netplans
cat /etc/netplan/master-multipass.yaml

## Anwenden des Netplans
sudo netplan apply

### Verbinden mit dem WorkerNode1 in der Kommandozeile
multipass shell k3s-worker-1

### IP-Adresse einsehen der Netzwerkschnittstelle
ip a

### MAC-Adresse einsehen der Netzwerkschnittstelle
ip l

### Öffnen und bearbeiten des Netplans
sudo nano /etc/netplan/master-multipass.yaml

### Netplan erstellen (Vorsicht "<>" Platzhalter)
### k3s-worker-1
network:
    ethernets:
        eth1:
            dhcp4: false
            match:
                macaddress: 52:54:00:3c:b7:23
            set-name: eth1
            addresses: [192.168.10.11/24]
    version: 2

### ctrl+X => ctrl+Y => Enter = Speichern des Netplans

### Öffnen Netplans
cat /etc/netplan/master-multipass.yaml

### Anwenden des Netplans
sudo netplan apply

#### Verbinden mit dem WorkerNode2 in der Kommandozeile
multipass shell k3s-worker-2

#### IP-Adresse einsehen der Netzwerkschnittstelle
ip a

#### MAC-Adresse einsehen der Netzwerkschnittstelle
ip l

#### Öffnen und bearbeiten des Netplans
sudo nano /etc/netplan/master-multipass.yaml

#### Netplan erstellen (Vorsicht "<>" Platzhalter)
#### k3s-worker-2
network:
    ethernets:
        eth1:
            dhcp4: false
            match:
                macaddress: 52:54:00:24:39:1e
            set-name: eth1
            addresses: [192.168.10.12/24]
    version: 2

#### ctrl+X => ctrl+Y => Enter = Speichern des Netplans

#### Öffnen Netplans
cat /etc/netplan/master-multipass.yaml

#### Anwenden des Netplans
sudo netplan apply

#### Auflistung der Virtuellen Maschinen (Nodes)
multipass list

###
Name                    State             IPv4             Image
k3s-master              Running           172.25.138.135   Ubuntu 22.04 LTS
                                          172.25.139.155
                                          192.168.10.10
k3s-worker-1            Running           172.25.135.2     Ubuntu 22.04 LTS
                                          172.25.130.73
                                          192.168.10.11
k3s-worker-2            Running           172.25.141.6     Ubuntu 22.04 LTS
                                          172.25.130.87
                                          192.168.10.12
###

12. # k3s installieren auf dem MasterNode

# Diese Vorgänge müssen in Powershell ausgeführt werden!
# Installiert und initialisiert k3s als Kubernetes-Master auf einer Multipass-VM namens k3s-master, wobei die IP-Adresse des Masters auf 192.168.10.10 gesetzt und der eingebaute Service Load Balancer deaktiviert wird
multipass exec k3s-master -- bash -c "curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC='--node-ip=[192.168.10.10] --cluster-init --disable servicelb' sh -"

## K3S Token abrufen
multipass exec k3s-master sudo cat /var/lib/rancher/k3s/server/node-token

## Erhaltener Token
K1011cea11d8e40c184e01100f5d5f9d7f902678eeea346feee8aad6c1e87ab81e1::server:157aad372f4ad99d0f8513a90407e7b0

## K3S auf allen Worker installieren
multipass exec k3s-worker-* -- bash -c "curl -sfL https://get.k3s.io | K3S_URL=https://<IP_MASTER>:6443 K3S_TOKEN=<TOKEN> INSTALL_K3S_EXEC='--node-ip=<IP_WORKER>' sh -"

## Platzhalter ersetzt mit unseren Attributen (MasterNode => WorkerNode-1)
multipass exec [k3s-worker-1] -- bash -c "curl -sfL https://get.k3s.io | K3S_URL=https://[192.168.10.10]:6443 K3S_TOKEN=[K1011cea11d8e40c184e01100f5d5f9d7f902678eeea346feee8aad6c1e87ab81e1::server:157aad372f4ad99d0f8513a90407e7b0] INSTALL_K3S_EXEC='--node-ip=[192.168.10.11]' sh -"

## Platzhalter ersetzt mit unseren Attributen (MasterNode => WorkerNode-2)
multipass exec [k3s-worker-2] -- bash -c "curl -sfL https://get.k3s.io | K3S_URL=https://[192.168.10.10]:6443 K3S_TOKEN=[K1011cea11d8e40c184e01100f5d5f9d7f902678eeea346feee8aad6c1e87ab81e1::server:157aad372f4ad99d0f8513a90407e7b0] INSTALL_K3S_EXEC='--node-ip=[192.168.10.12]' sh -"

13. # Installieren von kube-vip auf der MasterNode
# Bei der Installation von kube-vip auf der MasterNode wird eine hochverfügbare virtuelle IP-Adresse (VIP) konfiguriert, die als einzelner Netzwerkendpunkt für den Zugriff auf den Kubernetes API-Server dient, um Ausfallzeiten bei der Verwaltung des Clusters zu minimieren

## Root-Shell auf MasterNode öffnen
sudo -i

## Netzwerkschnittstelle bestimmen (eth1)
ip a

## Schnittstellenname exportieren als Umgebungsvariable
export INTERFACE=<ubuntu network interface>
export INTERFACE=eth1

## Virtuelle IP-Adresse (VIP) exportieren als Umgebungsvariable
export VIP=<Ihre-VIP>
export VIP=192.168.10.5

14. # Installieren von RBAC-Ressourcen für kube-vip
# Beim Installieren von RBAC-Ressourcen für kube-vip werden spezifische Zugriffsregeln im Kubernetes-Cluster eingerichtet, die definieren, welche Berechtigungen kube-vip für den Betrieb benötigt

## RBAC-Manifest-Datei für kube-vip herunterladen
curl https://kube-vip.io/manifests/rbac.yaml > /var/lib/rancher/k3s/server/manifests/kube-vip-rbac.yaml

## RBAC-Manifest anwenden
kubectl apply -f /var/lib/rancher/k3s/server/manifests/kube-vip-rbac.yaml

15. # DaemonSet-Manifest für kube-vip
# Das DaemonSet-Manifest für kube-vip definiert eine Gruppe von Pods, die auf jedem Master-Knoten des Kubernetes-Clusters laufen sollen, um die kube-vip-Funktionalität bereitzustellen, die für das Load Balancing und die Hochverfügbarkeit des API-Servers erforderlich ist

## DaemonSet-Manifest für kube-vip erzeugen
kubectl run -it kube-vip-init  --image=ghcr.io/kube-vip/kube-vip:main --restart=Never --rm -- manifest daemonset \
    --interface $INTERFACE \
    --address $VIP \
    --inCluster \
    --taint \
    --controlplane \
    --arp \
    --leaderElection > /var/lib/rancher/k3s/server/manifests/kube-vip-daemonset.yaml

## DaemonSet-Manifest für kube-vip öffnen
nano /var/lib/rancher/k3s/server/manifests/kube-vip-daemonset.yaml

## DaemonSet-Manifest für kube-vip ändern (Versionanpassung von v0.7.0 auf v0.6.4)
# ctrl+X => ctrl+Y => Enter = Speichern des Manifests
apiVersion: apps/v1
kind: DaemonSet
metadata:
  creationTimestamp: null
  labels:
    app.kubernetes.io/name: kube-vip-ds
    app.kubernetes.io/version: v0.6.4
  name: kube-vip-ds
  namespace: kube-system
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: kube-vip-ds
  template:
    metadata:
      creationTimestamp: null
      labels:
        app.kubernetes.io/name: kube-vip-ds
        app.kubernetes.io/version: v0.6.4
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: node-role.kubernetes.io/master
                operator: Exists
            - matchExpressions:
              - key: node-role.kubernetes.io/control-plane
                operator: Exists
      containers:
      - args:
        - manager
        env:
        - name: vip_arp
          value: "true"
        - name: port
          value: "6443"
        - name: vip_interface
          value: eth1
        - name: vip_cidr
          value: "32"
        - name: dns_mode
          value: first
        - name: cp_enable
          value: "true"
        - name: cp_namespace
          value: kube-system
        - name: vip_leaderelection
          value: "true"
        - name: vip_leasename
          value: plndr-cp-lock
        - name: vip_leaseduration
          value: "5"
        - name: vip_renewdeadline
          value: "3"
        - name: vip_retryperiod
          value: "1"
        - name: address
          value: 192.168.10.5
        - name: prometheus_server
          value: :2112
        image: ghcr.io/kube-vip/kube-vip:v0.6.4
        imagePullPolicy: Always
        name: kube-vip
        resources: {}
        securityContext:
          capabilities:
            add:
            - NET_ADMIN
            - NET_RAW
      hostNetwork: true
      serviceAccountName: kube-vip
      tolerations:
      - effect: NoSchedule
        operator: Exists
      - effect: NoExecute
        operator: Exists
  updateStrategy: {}
status:
  currentNumberScheduled: 0
  desiredNumberScheduled: 0
  numberMisscheduled: 0
  numberReady: 0
        
## DaemonSet-Manifest für kube-vip anwenden
kubectl apply -f /var/lib/rancher/k3s/server/manifests/kube-vip-daemonset.yaml

16. # Installieren von kubectx und kubens mit Chocolatey auf der lokalen Maschine mit Power Shell
# Hier werden Kommandozeilenwerkzeuge zur Vereinfachung des Wechsels zwischen verschiedenen Kubernetes-Clustern und Namespaces auf dem Entwicklerrechner hinzugefügt

## chocolatey (https://docs.chocolatey.org/en-us/choco/setup) => Paketmanager
Set-ExecutionPolicy Bypass -Scope Process -Force; [System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072; iex ((New-Object System.Net.WebClient).DownloadString('https://community.chocolatey.org/install.ps1'))

## kubectx (Erleichterter Wechsel in verschiedenen Cluster und deren Kontext) install auf PowerShell
choco install kubens kubectx

## Kubeconfig ansehen auf der SSH MasterNode
sudo cat /etc/rancher/k3s/k3s.yaml

### Config-Backup-08.02.2024.yaml
### Config-Backup erstellen und bestehende IP-Adresse ändern <127.0.0.1> auf
<server: https://192.168.10.10:6443>

17. # Erstellen der Datei "ipadresspool.yaml" mit dem einem Range von 3 reservierten Adressen
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: default
  namespace: metallb-system
spec:
  addresses:
  - <192.168.10.20>-<192.168.10.25>


18. # Erstellen der Datei "l2advertisement.yaml" mit eigener Netzwerkschnittstelle
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: default
  namespace: metallb-system
spec:
  ipAddressPools:
  - default
  interfaces:
  - <eth1>
-----------------------------------------------------------------------------------
19. # MetalLB installieren (LoadBalancer)

## Helm mit Chocolatey installieren (Power Shell)
choco install kubernetes-helm

## MetalLB-Helm-Repository hinzufügen und aktualisieren
helm repo add metallb https://metallb.github.io/metallb

## Helm-Repositories aktualisieren
helm repo update

## MetalLB mit Helm installieren
helm upgrade --install metallb metallb/metallb --namespace metallb-system --create-namespace

20. # IP-Adresspool anwenden (IP-Range LoadBalancer)
kubectl apply -f ipaddresspool.yaml

21. # L2-Netzwerkschnittstellenzuweisung anwenden (Steuert die IP-Adressen und Pool auf NW-Ebene)
kubectl apply -f l2advertisement.yaml

## Installationsüberprüfung MetalLB und zugehörigkeiten
kubectl get pods -n metallb-system

## Überprüfung IP-Adressen im Cluster
multipass list

#
Name                    State             IPv4             Image
k3s-master              Running           172.25.138.135   Ubuntu 22.04 LTS
                                          172.25.132.116
                                          192.168.10.5
                                          10.42.0.0
                                          10.42.0.1
k3s-worker-1            Running           172.25.135.2     Ubuntu 22.04 LTS
                                          172.25.130.73
                                          192.168.10.11
k3s-worker-2            Running           172.25.141.6     Ubuntu 22.04 LTS
                                          172.25.130.87
                                          192.168.10.12
#


22. # Installation von nfs-common 
## Network File System ist ein Protokoll, das es ermöglicht, Dateien über ein Netzwerk gemeinsam zu nutzen

# Verbinden mit den Nodes in der Kommandozeile
multipass shell k3s-master
multipass shell k3s-worker-1
multipass shell k3s-worker-2

## nfs-common auf jeder VM-installieren
sudo apt update
sudo apt install nfs-common

23. # Longhorn installieren (Verteiltes Speichersystem)
kubectl apply -f https://raw.githubusercontent.com/longhorn/longhorn/v1.5.3/deploy/longhorn.yaml

24. # Mit diesem Befehl werden alle Ressourcen aufgelistet, die sich im Namespace "metallb-system" befinden, einschliesslich Pods, Services, Deployments, ConfigMaps, und andere Ressourcen, die im Namespace erstellt wurden
kubectl get all -n metallb-system

##
NAME                                            READY   STATUS    RESTARTS   AGE
pod/metallb-controller-648b76f565-k2pc8         1/1     Running   0          11m
pod/metallb-speaker-q6kjw                       4/4     Running   0          11m

NAME                                            TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
service/metallb-webhook-service                 ClusterIP   10.111.228.165   <none>        443/TCP   11m

NAME                                            DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
daemonset.apps/metallb-speaker                  1         1         1       1            1          
 kubernetes.io/os=linux   11m

NAME                                            READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/metallb-controller              1/1     1            1           11m

NAME                                            DESIRED   CURRENT   READY   AGE      
replicaset.apps/metallb-controller-648b76f565   1         1         1       11m      
##

25. # Containerisierung und Deployment der Beispielapplikation

## Klone das Repository der Beispielapplikation auf deinem lokalen Rechner (ms-phonebook-example)
git clone https://github.com/SwitzerChees/ms-phonebook-example

### Bearbeiten des Dockerfiles
Siehe: ms_phonebook-example/reult_server/templates/Dockerfile

#### Docker-Image erstellen
docker build -t result_server .

#### Docker-Container starten basierend auf dem gebauten Image
docker run -p 8080:80 result_server

#### Verbindung kontrollieren
http://localhost:8080

#### Alle Container anzeigen
docker ps -a
CONTAINER ID   IMAGE          COMMAND           CREATED             STATUS                     PORTS  NAMES
472fa7053cb6   result_server  "python app.py"   About a minute ago  Exited (0) 35 seconds ago         serene_jackson

##### Docker Image taggen
docker tag result_server  manuelbrunner/result_server:latest

##### Docker Image auf Docker Hub pushen
docker push manuelbrunner/result_server:latest
#
The push refers to repository [docker.io/manuelbrunner/result_server]
7315d78f95e9: Pushed
96a5e07e6ea5: Pushed
878030f1fd1e: Pushed
6b4c0dede09b: Pushed
2c41385f0400: Pushed
b2414301083a: Mounted from library/python
e67a1acfc698: Mounted from library/python
7701f8551e70: Mounted from library/python
da5d55102092: Mounted from library/python
fb1bd2fc5282: Mounted from library/python
latest: digest: sha256:c978c741b53f10cb5d0a26492a47edc059140302c3208d67765502a686af49ad size: 2415
#

# Bearbeiten des Dockerfiles
Siehe: ms_phonebook-example/web_server/templates/Dockerfile

## Docker-Image erstellen
docker build -t web_server .

### Docker-Container starten basierend auf dem gebauten Image
docker run -p 5000:80 web_server

#### Verbindung kontrollieren
http://localhost:5000

#### Alle Container anzeigen
docker ps -a

##### Docker Image taggen
docker tag web_server  manuelbrunner/web_server:latest

##### Docker Image auf Docker Hub pushen
docker push manuelbrunner/web_server:latest
#The push refers to repository [docker.io/manuelbrunner/web_server]
7315d78f95e9: Mounted from manuelbrunner/result_server
96a5e07e6ea5: Mounted from manuelbrunner/result_server
878030f1fd1e: Mounted from manuelbrunner/result_server
6b4c0dede09b: Mounted from manuelbrunner/result_server
2c41385f0400: Mounted from manuelbrunner/result_server
b2414301083a: Mounted from manuelbrunner/result_server
e67a1acfc698: Mounted from manuelbrunner/result_server
7701f8551e70: Mounted from manuelbrunner/result_server
da5d55102092: Mounted from manuelbrunner/result_server
fb1bd2fc5282: Mounted from manuelbrunner/result_server
latest: digest: sha256:c978c741b53f10cb5d0a26492a47edc059140302c3208d67765502a686af49ad size: 2415
#

26. Nodes integrieren in Multi-Node-Cluster

# Kubeconfig von K3s-Master abrufen
multipass exec k3s-master -- sudo cat /etc/rancher/k3s/k3s.yaml

## KubeConfig-K3S-Master.yaml Datei erstellen

### KubeConfig-K3S-Master.yaml Datei exportieren
export KUBECONFIG=/c/K3S_Kubernetes_Cluster/Auftrag_K3S/KubeConfig-K3S-Master.yaml

#### KubeConfig-K3S-Master.yaml Datei anwenden
kubectl apply -f KubeConfig-K3S-Master.yaml

##### Worker-1 integrieren 
multipass exec k3s-worker-1 -- sudo k3s agent --server https://192.168.10.10:6443 --token K102cf50145cc10205da134cba90717e95d2e086a6c99dc1b39ad38e789653628b2::server:74bd96c372fbc077bbcb5f8bbaf22c51

###### Worker-2 integrieren 
multipass exec k3s-worker-2 -- sudo k3s agent --server https://192.168.10.10:6443 --token K102cf50145cc10205da134cba90717e95d2e086a6c99dc1b39ad38e789653628b2::server:74bd96c372fbc077bbcb5f8bbaf22c51

27. # Erstellung der YAML-Files (Deployments und Services)
-web-server-deployment.yaml (Deployment für den web_server)
-web-server-service.yaml (Service für den web_server)
-result-server-deployment.yaml (Deployment für den result_server)
-result-server-service.yaml (Service für den result_server)
-mysql-deployment.yaml (MySQL Deployment)
-mysql-service.yaml (MySQL Service)
-mysql-pvc.yaml (PersistentVolumeClaim für MySQL)
-KubeConfig-K3S-Master.yaml (Clusterintegration)

## Anwendungen auf Cluster anwenden
kubectl apply -f web-server-deployment.yaml
kubectl apply -f web-server-service.yaml
kubectl apply -f result-server-deployment.yaml
kubectl apply -f result-server-service.yaml
kubectl apply -f mysql-deployment.yaml
kubectl apply -f mysql-service.yaml
kubectl apply -f mysql-pvc.yaml
kubectl apply -f KubeConfig-K3S-Master.yaml

### Nodes labeln
kubectl label nodes k3s-master node-role=k3s-master
kubectl label nodes k3s-worker-1 node-role=k3s-worker-1
kubectl label nodes k3s-worker-2 node-role=k3s-worker-2

28. # Grobtesting

## Kontrolle mit dem Ping
k3s-master nach k3s-worker-1 192.168.10.11
k3s-master nach k3s-worker-2 192.168.10.12
k3s-worker-1 nach k3s-worker-2 192.168.10.12
k3s-worker-1 nach k3s-master 192.168.10.10
k3s-worker-2 nach k3s-worker-1 192.168.10.11
k3s-worker-2 nach k3s-master 192.168.10.10

### Status der deployments überprüfen
kubectl get deployments
NAME                       READY   UP-TO-DATE   AVAILABLE   AGE
mysql                      1/1     1            1           4h9m
result-server-deployment   2/2     2            2           4h7m
web-server-deployment      2/2     2            2           4h7m

#### Status der Services überprüfen
kubectl get services
NAME                    TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
kubernetes              ClusterIP      10.43.0.1       <none>        443/TCP        6d23h        
mysql-service           ClusterIP      10.43.130.194   <none>        3306/TCP       103m
result-server-service   LoadBalancer   10.43.138.141   <pending>     80:30277/TCP   103m
web-server-service      LoadBalancer   10.43.137.243   <pending>     80:31680/TCP   4h8m

##### Status der pods überprüfen
kubectl get pods
NAME                                        READY   STATUS    RESTARTS   AGE
mysql-68f8cf4656-ndzcj                      1/1     Running   0          4h10m
result-server-deployment-585454c444-2b9mp   1/1     Running   0          4h8m
result-server-deployment-585454c444-th4xc   1/1     Running   0          4h8m
web-server-deployment-85bd489c44-8tcf6      1/1     Running   0          4h8m
web-server-deployment-85bd489c44-zcwcd      1/1     Running   0          4h8m

###### Nodes auflistung
multipass list
#
Name                    State             IPv4             Image
k3s-master              Running           172.25.138.135   Ubuntu 22.04 LTS
                                          172.25.132.116
                                          192.168.10.5
                                          10.42.0.0
                                          10.42.0.1
k3s-worker-1            Running           172.25.135.36    Ubuntu 22.04 LTS
                                          192.168.10.11
                                          10.42.1.0
                                          10.42.1.1
k3s-worker-2            Running           172.25.141.6     Ubuntu 22.04 LTS
                                          172.25.130.87
                                          192.168.10.12
                                          10.42.2.0
                                          10.42.2.1
#    
                                      
###### Überprüfen der Logs
kubectl logs <Pod-Name>

###### Dauerhafte Speicherung überprüfen
kubectl delete pod <MySQL-Pod-Name>


###### Nodes überprüfen:
kubectl get nodes
NAME           STATUS   ROLES                       AGE     VERSION
k3s-master     Ready    control-plane,etcd,master   7d      v1.28.6+k3s2
k3s-worker-1   Ready    <none>                      5h18m   v1.28.6+k3s2
k3s-worker-2   Ready    <none>                      5h16m   v1.28.6+k3s2

###### Detaillierte Ansicht der Pods
kubectl get pods -o wide

###### Detaillierte Liste aller Pods in allen Namespaces des Kubernetes-Clusters
kubectl get pods --all-namespaces -o wide

10. Überprüfung der Node-Details
kubectl describe node <nodename>


## zugriff lokal?
# Diverses 192.168.10.20 und 192.168.10.21 ????????????????? Geschäft geht nicht
# Localhost mit Portforwarding?
kubectl port-forward service/<service-name> <local-port>:<service-port>
## Einfügen wenn ich versioniere auf github
## Wie mache ich das in der <CLI>?