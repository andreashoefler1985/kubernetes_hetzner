# 🚀 Hetzner Cloud Kubernetes Setup

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Kubernetes](https://img.shields.io/badge/Kubernetes-1.28-blue.svg)](https://kubernetes.io/)
[![Hetzner Cloud](https://img.shields.io/badge/Hetzner-Cloud-red.svg)](https://www.hetzner.com/cloud)

Automatisches Setup für ein produktionsreifes Kubernetes-Cluster in der Hetzner Cloud mit nur einem Befehl.

## ✨ Features

- 🔄 **Vollautomatisches Setup** - Ein Script für alles
- 🌐 **Hetzner Load Balancer** - Native Integration
- 🔒 **Privates Netzwerk** - Sicherer Cluster-Traffic
- 📊 **Monitoring Ready** - Prometheus & Grafana Support
- 💰 **Kosteneffizient** - ~23€/Monat für komplettes Setup
- 🛡️ **Sicherheit** - Best Practices integriert
- 🔧 **Idempotent** - Mehrfach ausführbar ohne Probleme

## 🏗️ Architektur

```
                    Internet
                       |
                 Load Balancer
                 (LB11 - 5.39€)
                       |
        ┌──────────────┼──────────────┐
        |              |              |
   k8s-master    k8s-worker-1   k8s-worker-2
   (CX21-5.83€)  (CX21-5.83€)   (CX21-5.83€)
        |              |              |
        └──────────────┼──────────────┘
                Private Network
                (10.0.0.0/16)
```

## 🚀 Schnellstart

### Voraussetzungen

- [Hetzner Cloud Account](https://www.hetzner.com/cloud) mit API-Token
- [hcloud CLI](https://github.com/hetznercloud/cli) installiert
- [kubectl](https://kubernetes.io/docs/tasks/tools/) installiert
- SSH-Client

### Installation

1. **Repository klonen:**
   ```bash
   git clone https://github.com/username/hetzner-k8s-setup.git
   cd hetzner-k8s-setup
   chmod +x hetzner-k8s-setup.sh
   ```

2. **API-Token setzen:**
   ```bash
   export HCLOUD_TOKEN="your-hetzner-api-token"
   ```

3. **Cluster erstellen:**
   ```bash
   ./hetzner-k8s-setup.sh setup
   ```

4. **kubectl konfigurieren:**
   ```bash
   export KUBECONFIG=~/.kube/hetzner-k8s-config
   kubectl get nodes
   ```

Das war's! 🎉

## 📋 Was wird installiert?

### Infrastruktur
- ✅ 1x Master Node (CX21: 2 vCPU, 4GB RAM)
- ✅ 2x Worker Nodes (CX21: 2 vCPU, 4GB RAM)
- ✅ Load Balancer (LB11: 20GB Traffic)
- ✅ Privates Netzwerk (10.0.0.0/16)
- ✅ SSH-Key Management

### Kubernetes-Stack
- ✅ Kubernetes 1.28
- ✅ containerd Runtime
- ✅ Flannel CNI
- ✅ Hetzner Cloud Controller Manager
- ✅ Hetzner CSI Driver
- ✅ Nginx Beispiel-App

## 🎛️ Konfiguration

Anpassbare Parameter am Anfang des Scripts:

```bash
# Server-Konfiguration
SERVER_TYPE="cx21"        # cx11, cx21, cx31, cx41, cx51
WORKER_COUNT=2            # Anzahl Worker Nodes
LOCATION="nbg1"           # nbg1, fsn1, hel1
LB_TYPE="lb11"           # lb11, lb21, lb31

# Netzwerk
NETWORK_IP_RANGE="10.0.0.0/16"
SUBNET_IP_RANGE="10.0.1.0/24"
```

## 🖥️ Dashboard & GUI

### Option 1: Rancher (Empfohlen)
```bash
./setup-rancher.sh
# Zugriff: https://rancher.yourdomain.com
```

### Option 2: Kubernetes Dashboard
```bash
./setup-dashboard.sh
# Zugriff: http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/
```

### Option 3: Grafana + Prometheus
```bash
./setup-monitoring.sh
# Grafana: http://your-lb-ip:30080
# Prometheus: http://your-lb-ip:30301
```

## 📊 Monitoring Setup

Installiere das komplette Monitoring-Stack:

```bash
# Prometheus + Grafana + AlertManager
./grafana-prometheus-setup.sh

# Zugriff:
# Grafana: http://LOAD_BALANCER_IP:30080 (admin/admin123)
# Prometheus: http://LOAD_BALANCER_IP:30301
# AlertManager: http://LOAD_BALANCER_IP:30302
```

Enthält vorkonfigurierte Dashboards:
- 📈 Kubernetes Cluster Overview
- 🖥️ Node Resource Usage
- 📦 Pod Resource Usage
- 🔔 Alerting Rules

## 🔧 Verwaltung

### Cluster-Status prüfen
```bash
./hetzner-k8s-setup.sh status
```

### Logs anzeigen
```bash
kubectl logs -n kube-system -l app=hcloud-cloud-controller-manager
```

### Worker Node hinzufügen
```bash
# Neuen Server erstellen
hcloud server create --name k8s-worker-3 --type cx21 --image ubuntu-22.04 --ssh-key k8s-key --location nbg1 --network k8s-network

# Kubernetes installieren (siehe manual-k8s-install.sh)
# Join-Command vom Master ausführen
```

### Cluster skalieren
```bash
# Horizontal Pod Autoscaler
kubectl autoscale deployment nginx-example --cpu-percent=80 --min=1 --max=10

# Cluster Autoscaler (optional)
# Siehe docs/cluster-autoscaler.md
```

## 🛡️ Sicherheit

### Implementierte Sicherheitsmaßnahmen:
- ✅ Privates Netzwerk für Cluster-Kommunikation
- ✅ SSH-Key basierte Authentifizierung
- ✅ Automatische Security-Updates
- ✅ Pod Security Standards
- ✅ Network Policies (via Calico - optional)

### Empfohlene Zusatz-Sicherheit:
```bash
# Falco für Runtime Security
kubectl apply -f https://raw.githubusercontent.com/falcosecurity/falco/main/deploy/kubernetes/falco-with-ebpf-modern.yaml

# OPA Gatekeeper für Policy Enforcement
kubectl apply -f https://raw.githubusercontent.com/open-policy-agent/gatekeeper/release-3.14/deploy/gatekeeper.yaml
```

## 💰 Kosten-Übersicht

| Komponente | Typ | Monatliche Kosten |
|------------|-----|-------------------|
| Master Node | CX21 | 5,83€ |
| Worker Node 1 | CX21 | 5,83€ |
| Worker Node 2 | CX21 | 5,83€ |
| Load Balancer | LB11 | 5,39€ |
| **Gesamt** | | **~22,88€** |

*Preise sind ca.-Angaben und können sich ändern.*

## 🔄 Backup & Disaster Recovery

### Velero für Cluster-Backups:
```bash
# Velero installieren
./setup-velero.sh

# Backup erstellen
velero backup create my-backup

# Restore
velero restore create --from-backup my-backup
```

## 🚨 Troubleshooting

### Häufige Probleme:

**Problem: API Server nicht erreichbar**
```bash
# Lösung: Sicherheitsgruppen prüfen
hcloud server describe k8s-master
```

**Problem: Pods bleiben in Pending**
```bash
# Lösung: Node-Status prüfen
kubectl describe nodes
kubectl get events --sort-by=.metadata.creationTimestamp
```

**Problem: Load Balancer funktioniert nicht**
```bash
# Lösung: Health Checks prüfen
hcloud load-balancer describe k8s-lb
```

### Logs sammeln:
```bash
# System-Logs
journalctl -u kubelet

# Cluster-Events
kubectl get events -A --sort-by=.metadata.creationTimestamp

# Pod-Logs
kubectl logs -n kube-system -l component=kube-apiserver
```

## 🔄 Updates

### Kubernetes-Updates:
```bash
# Auf Master Node
kubeadm upgrade plan
kubeadm upgrade apply v1.29.0

# Auf Worker Nodes
kubeadm upgrade node
```

### System-Updates:
```bash
# Automatisch via unattended-upgrades (bereits konfiguriert)
# Oder manuell:
apt update && apt upgrade -y
```

## 🧹 Cleanup

```bash
# Komplettes Cluster löschen
./hetzner-k8s-setup.sh cleanup

# Nur bestimmte Komponenten löschen
hcloud server delete k8s-worker-2
hcloud load-balancer delete k8s-lb
```

## 📚 Weiterführende Dokumentation

- [Kubernetes Dokumentation](https://kubernetes.io/docs/)
- [Hetzner Cloud API](https://docs.hetzner.cloud/)
- [Helm Charts](https://helm.sh/docs/)
- [Prometheus Operator](https://prometheus-operator.dev/)

## 🤝 Contributing

Contributions sind willkommen! Bitte:

1. Fork das Repository
2. Erstelle einen Feature-Branch (`git checkout -b feature/amazing-feature`)
3. Committe deine Änderungen (`git commit -m 'Add amazing feature'`)
4. Push zum Branch (`git push origin feature/amazing-feature`)
5. Öffne einen Pull Request

### Development Setup:
```bash
# Pre-commit hooks installieren
pip install pre-commit
pre-commit install

# Tests ausführen
./tests/run-tests.sh
```

## 📄 Lizenz

Dieses Projekt ist unter der MIT-Lizenz lizenziert - siehe [LICENSE](LICENSE) für Details.

## 🙏 Danksagungen

- [Hetzner Cloud](https://www.hetzner.com/cloud) für die großartige Infrastruktur
- [Kubernetes Community](https://kubernetes.io/community/) für das fantastische Ökosystem
- Alle Contributors die dieses Projekt verbessern

## 📞 Support

- 🐛 **Bug Reports:** [GitHub Issues](https://github.com/username/hetzner-k8s-setup/issues)
- 💬 **Diskussionen:** [GitHub Discussions](https://github.com/username/hetzner-k8s-setup/discussions)
- 📧 **Email:** support@example.com

---

**⭐ Wenn dir dieses Projekt hilft, gib ihm gerne einen Stern!**

![Kubernetes](https://img.shields.io/badge/kubernetes-%23326ce5.svg?style=for-the-badge&logo=kubernetes&logoColor=white)
![Hetzner](https://img.shields.io/badge/Hetzner-D50C2D?style=for-the-badge&logo=hetzner&logoColor=white)
![Docker](https://img.shields.io/badge/docker-%230db7ed.svg?style=for-the-badge&logo=docker&logoColor=white)
