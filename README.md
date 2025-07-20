
1. Vorbereitung:
bash# Script herunterladen und ausführbar machen
chmod +x hetzner-k8s-setup.sh

# Hetzner API Token setzen
export HCLOUD_TOKEN="dein-hetzner-api-token"
2. Komplettes Cluster aufsetzen:
bash./hetzner-k8s-setup.sh setup
3. Cluster-Status prüfen:
bash./hetzner-k8s-setup.sh status
4. Cluster löschen:
bash./hetzner-k8s-setup.sh cleanup
Was das Script automatisch macht:
✅ Infrastruktur:

Erstellt 3 Server (1 Master, 2 Worker)
Richtet privates Netzwerk ein
Erstellt und konfiguriert Load Balancer
Verwaltet SSH-Keys automatisch

✅ Kubernetes:

Installiert Docker und Kubernetes auf allen Nodes
Initialisiert Master Node mit kubeadm
Verbindet Worker Nodes automatisch
Installiert Flannel Pod Network
Richtet Hetzner Cloud Controller Manager ein

✅ Beispiel-App:

Deployed eine Nginx-Anwendung
Konfiguriert NodePort Service
Verbindet mit Load Balancer

✅ Lokale Konfiguration:

Lädt kubeconfig herunter
Richtet lokalen kubectl-Zugang ein

Anpassungen im Script:
Du kannst am Anfang des Scripts verschiedene Parameter anpassen:

SERVER_TYPE: Server-Größe (cx11, cx21, cx31...)
WORKER_COUNT: Anzahl Worker Nodes
LOCATION: Datacenter (nbg1, fsn1, hel1)
LB_TYPE: Load Balancer Typ (lb11, lb21, lb31)

Das Script ist vollständig idempotent - du kannst es mehrfach ausführen, ohne dass Probleme entstehen. Es prüft immer, ob Ressourcen bereits existieren.
Nach dem Setup kannst du direkt loslegen:
bashexport KUBECONFIG=~/.kube/hetzner-k8s-config
kubectl get nodes
