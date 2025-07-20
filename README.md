#!/bin/bash

# =============================================================================
# Automatisches Kubernetes Setup für Hetzner Cloud
# =============================================================================

set -e  # Script bei Fehlern beenden

# Farben für Output
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
BLUE='\033[0;34m'
NC='\033[0m' # No Color

# Logging-Funktionen
log_info() {
    echo -e "${BLUE}[INFO]${NC} $1"
}

log_success() {
    echo -e "${GREEN}[SUCCESS]${NC} $1"
}

log_warning() {
    echo -e "${YELLOW}[WARNING]${NC} $1"
}

log_error() {
    echo -e "${RED}[ERROR]${NC} $1"
}

# =============================================================================
# KONFIGURATION - Hier anpassen!
# =============================================================================

# Hetzner Cloud Konfiguration
HCLOUD_TOKEN="${HCLOUD_TOKEN:-}"  # Setze als Umgebungsvariable oder hier direkt
PROJECT_NAME="k8s-cluster"
LOCATION="nbg1"  # nbg1, fsn1, hel1
SERVER_TYPE="cx21"  # cx11, cx21, cx31, cx41, cx51
LB_TYPE="lb11"    # lb11, lb21, lb31

# Cluster Konfiguration
CLUSTER_NAME="hetzner-k8s"
WORKER_COUNT=2
SSH_KEY_NAME="k8s-key"
NETWORK_NAME="k8s-network"
NETWORK_ZONE="eu-central"
NETWORK_IP_RANGE="10.0.0.0/16"
SUBNET_IP_RANGE="10.0.1.0/24"

# Kubernetes Version
K8S_VERSION="1.28"

# SSH Konfiguration
SSH_KEY_PATH="$HOME/.ssh/hetzner_k8s"
SSH_PUB_KEY_PATH="${SSH_KEY_PATH}.pub"

# =============================================================================
# VORAUSSETZUNGEN PRÜFEN
# =============================================================================

check_requirements() {
    log_info "Prüfe Voraussetzungen..."
    
    # hcloud CLI prüfen
    if ! command -v hcloud &> /dev/null; then
        log_error "hcloud CLI ist nicht installiert!"
        log_info "Installation: brew install hcloud (macOS) oder https://github.com/hetznercloud/cli"
        exit 1
    fi
    
    # kubectl prüfen
    if ! command -v kubectl &> /dev/null; then
        log_error "kubectl ist nicht installiert!"
        log_info "Installation: https://kubernetes.io/docs/tasks/tools/"
        exit 1
    fi
    
    # SSH-Client prüfen
    if ! command -v ssh &> /dev/null; then
        log_error "SSH-Client ist nicht installiert!"
        exit 1
    fi
    
    # API Token prüfen
    if [[ -z "$HCLOUD_TOKEN" ]]; then
        log_error "HCLOUD_TOKEN ist nicht gesetzt!"
        log_info "Setze die Umgebungsvariable: export HCLOUD_TOKEN='dein-token'"
        exit 1
    fi
    
    log_success "Alle Voraussetzungen erfüllt"
}

# =============================================================================
# HCLOUD SETUP
# =============================================================================

setup_hcloud_context() {
    log_info "Richte hcloud Kontext ein..."
    
    # Kontext erstellen (falls nicht vorhanden)
    if ! hcloud context list | grep -q "$PROJECT_NAME"; then
        echo "$HCLOUD_TOKEN" | hcloud context create "$PROJECT_NAME"
    fi
    
    hcloud context use "$PROJECT_NAME"
    log_success "hcloud Kontext eingerichtet"
}

# =============================================================================
# SSH-KEY MANAGEMENT
# =============================================================================

setup_ssh_key() {
    log_info "Richte SSH-Key ein..."
    
    # SSH-Key generieren falls nicht vorhanden
    if [[ ! -f "$SSH_KEY_PATH" ]]; then
        log_info "Generiere neuen SSH-Key..."
        ssh-keygen -t rsa -b 4096 -f "$SSH_KEY_PATH" -N "" -C "hetzner-k8s-$(date +%Y%m%d)"
    fi
    
    # SSH-Key zu Hetzner hochladen falls nicht vorhanden
    if ! hcloud ssh-key list | grep -q "$SSH_KEY_NAME"; then
        log_info "Lade SSH-Key zu Hetzner hoch..."
        hcloud ssh-key create --name "$SSH_KEY_NAME" --public-key-from-file "$SSH_PUB_KEY_PATH"
    fi
    
    log_success "SSH-Key eingerichtet"
}

# =============================================================================
# NETZWERK SETUP
# =============================================================================

setup_network() {
    log_info "Richte privates Netzwerk ein..."
    
    # Netzwerk erstellen falls nicht vorhanden
    if ! hcloud network list | grep -q "$NETWORK_NAME"; then
        log_info "Erstelle privates Netzwerk..."
        hcloud network create --name "$NETWORK_NAME" --ip-range "$NETWORK_IP_RANGE"
        
        # Subnetz hinzufügen
        hcloud network add-subnet "$NETWORK_NAME" \
            --network-zone "$NETWORK_ZONE" \
            --type cloud \
            --ip-range "$SUBNET_IP_RANGE"
    fi
    
    log_success "Netzwerk eingerichtet"
}

# =============================================================================
# SERVER ERSTELLEN
# =============================================================================

create_servers() {
    log_info "Erstelle Kubernetes-Server..."
    
    # Master Node erstellen
    if ! hcloud server list | grep -q "k8s-master"; then
        log_info "Erstelle Master Node..."
        hcloud server create \
            --name "k8s-master" \
            --type "$SERVER_TYPE" \
            --image "ubuntu-22.04" \
            --ssh-key "$SSH_KEY_NAME" \
            --location "$LOCATION" \
            --network "$NETWORK_NAME"
    fi
    
    # Worker Nodes erstellen
    for i in $(seq 1 $WORKER_COUNT); do
        if ! hcloud server list | grep -q "k8s-worker-$i"; then
            log_info "Erstelle Worker Node $i..."
            hcloud server create \
                --name "k8s-worker-$i" \
                --type "$SERVER_TYPE" \
                --image "ubuntu-22.04" \
                --ssh-key "$SSH_KEY_NAME" \
                --location "$LOCATION" \
                --network "$NETWORK_NAME"
        fi
    done
    
    # Warten bis alle Server bereit sind
    log_info "Warte auf Server-Bereitschaft..."
    sleep 30
    
    # Server IPs abrufen
    MASTER_IP=$(hcloud server ip k8s-master)
    log_info "Master IP: $MASTER_IP"
    
    for i in $(seq 1 $WORKER_COUNT); do
        WORKER_IP=$(hcloud server ip "k8s-worker-$i")
        log_info "Worker $i IP: $WORKER_IP"
    done
    
    log_success "Alle Server erstellt"
}

# =============================================================================
# KUBERNETES INSTALLATION
# =============================================================================

install_kubernetes_dependencies() {
    local server_ip=$1
    local server_name=$2
    
    log_info "Installiere Kubernetes-Abhängigkeiten auf $server_name..."
    
    # Ersten Teil: System-Updates und Reboot wenn nötig
    ssh -i "$SSH_KEY_PATH" -o StrictHostKeyChecking=no root@"$server_ip" << 'EOF'
        set -e
        
        # System aktualisieren
        apt update && DEBIAN_FRONTEND=noninteractive apt upgrade -y
        apt install -y apt-transport-https ca-certificates curl gpg
        
        # Prüfen ob Neustart erforderlich ist
        if [[ -f /var/run/reboot-required ]]; then
            echo "REBOOT_REQUIRED=1" > /tmp/k8s-setup-status
            echo "System restart required - rebooting..."
            nohup bash -c 'sleep 2 && reboot' >/dev/null 2>&1 &
            exit 0
        else
            echo "REBOOT_REQUIRED=0" > /tmp/k8s-setup-status
        fi
EOF
    
    # Prüfen ob Neustart erforderlich war
    local reboot_required=0
    if ssh -i "$SSH_KEY_PATH" -o StrictHostKeyChecking=no root@"$server_ip" 'cat /tmp/k8s-setup-status 2>/dev/null | grep REBOOT_REQUIRED=1' >/dev/null 2>&1; then
        reboot_required=1
    fi
    
    if [[ $reboot_required -eq 1 ]]; then
        log_warning "$server_name wird neu gestartet... Warte 60 Sekunden"
        sleep 60
        
        # Warten bis Server wieder erreichbar ist
        log_info "Warte auf $server_name nach Neustart..."
        for i in {1..30}; do
            if ssh -i "$SSH_KEY_PATH" -o StrictHostKeyChecking=no -o ConnectTimeout=5 root@"$server_ip" 'echo "Server ready"' >/dev/null 2>&1; then
                log_success "$server_name ist wieder erreichbar"
                break
            fi
            sleep 10
            if [[ $i -eq 30 ]]; then
                log_error "$server_name ist nach Neustart nicht erreichbar!"
                exit 1
            fi
        done
    fi
    
    # Zweiter Teil: Kubernetes Installation nach eventuellem Reboot
    ssh -i "$SSH_KEY_PATH" -o StrictHostKeyChecking=no root@"$server_ip" << 'EOF'
        set -e
        
        # Docker installieren
        curl -fsSL https://download.docker.com/linux/ubuntu/gpg | gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
        echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | tee /etc/apt/sources.list.d/docker.list > /dev/null
        apt update
        apt install -y docker-ce docker-ce-cli containerd.io
        
        # Docker starten und aktivieren
        systemctl start docker
        systemctl enable docker
        
        # Kubernetes Repository hinzufügen
        curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
        echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /' | tee /etc/apt/sources.list.d/kubernetes.list
        
        # Kubernetes Komponenten installieren
        apt update
        apt install -y kubelet kubeadm kubectl
        apt-mark hold kubelet kubeadm kubectl
        
        # Swap deaktivieren
        swapoff -a
        sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
        
        # Container Runtime konfigurieren
        cat > /etc/containerd/config.toml << 'EOCONFIG'
disabled_plugins = []
imports = []
oom_score = 0
plugin_dir = ""
required_plugins = []
root = "/var/lib/containerd"
state = "/run/containerd"
version = 2

[plugins]
  [plugins."io.containerd.grpc.v1.cri"]
    [plugins."io.containerd.grpc.v1.cri".containerd]
      [plugins."io.containerd.grpc.v1.cri".containerd.runtimes]
        [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
          runtime_type = "io.containerd.runc.v2"
          [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
            SystemdCgroup = true
EOCONFIG
        
        systemctl restart containerd
        systemctl enable containerd
        
        # Kernel Module laden
        cat > /etc/modules-load.d/k8s.conf << 'EOMODULES'
overlay
br_netfilter
EOMODULES
        
        modprobe overlay
        modprobe br_netfilter
        
        # Sysctl Parameter setzen
        cat > /etc/sysctl.d/k8s.conf << 'EOSYSCTL'
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOSYSCTL
        
        sysctl --system
EOF
    
    log_success "Kubernetes-Abhängigkeiten auf $server_name installiert"
}

initialize_master_node() {
    log_info "Initialisiere Master Node..."
    
    MASTER_IP=$(hcloud server ip k8s-master)
    
    ssh -i "$SSH_KEY_PATH" -o StrictHostKeyChecking=no root@"$MASTER_IP" << 'EOF'
        set -e
        
        # Public IP für API Server Zertifikat abrufen (ohne CIDR)
        PUBLIC_IP=$(curl -s http://169.254.169.254/hetzner/v1/metadata/public-ipv4 | cut -d'/' -f1)
        
        # Kubernetes Cluster initialisieren
        kubeadm init --pod-network-cidr=10.244.0.0/16 --apiserver-cert-extra-sans=$PUBLIC_IP
        
        # kubectl für root konfigurieren
        mkdir -p $HOME/.kube
        cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
        chown $(id -u):$(id -g) $HOME/.kube/config
        
        # Pod Network (Flannel) installieren
        kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
        
        # Join-Token generieren und speichern
        kubeadm token create --print-join-command > /tmp/join-command.sh
        chmod +x /tmp/join-command.sh
EOF
    
    log_success "Master Node initialisiert"
}

join_worker_nodes() {
    log_info "Verbinde Worker Nodes mit Cluster..."
    
    MASTER_IP=$(hcloud server ip k8s-master)
    
    # Join-Command vom Master abrufen
    scp -i "$SSH_KEY_PATH" -o StrictHostKeyChecking=no root@"$MASTER_IP":/tmp/join-command.sh /tmp/join-command.sh
    
    for i in $(seq 1 $WORKER_COUNT); do
        WORKER_IP=$(hcloud server ip "k8s-worker-$i")
        log_info "Verbinde k8s-worker-$i mit Cluster..."
        
        # Join-Command auf Worker ausführen
        scp -i "$SSH_KEY_PATH" -o StrictHostKeyChecking=no /tmp/join-command.sh root@"$WORKER_IP":/tmp/join-command.sh
        ssh -i "$SSH_KEY_PATH" -o StrictHostKeyChecking=no root@"$WORKER_IP" 'bash /tmp/join-command.sh'
    done
    
    rm -f /tmp/join-command.sh
    log_success "Alle Worker Nodes verbunden"
}

# =============================================================================
# HETZNER CLOUD CONTROLLER MANAGER
# =============================================================================

install_hcloud_ccm() {
    log_info "Installiere Hetzner Cloud Controller Manager..."
    
    MASTER_IP=$(hcloud server ip k8s-master)
    
    # Hetzner API Token als Secret erstellen
    ssh -i "$SSH_KEY_PATH" -o StrictHostKeyChecking=no root@"$MASTER_IP" << EOF
        set -e
        
        # Cloud Controller Manager installieren
        kubectl apply -f https://github.com/hetznercloud/hcloud-cloud-controller-manager/releases/latest/download/ccm-networks.yaml
        
        # API Token Secret erstellen
        kubectl -n kube-system create secret generic hcloud \\
            --from-literal=token=$HCLOUD_TOKEN \\
            --from-literal=network=$NETWORK_NAME || true
        
        # CSI Driver installieren
        kubectl apply -f https://raw.githubusercontent.com/hetznercloud/csi-driver/main/deploy/kubernetes/hcloud-csi.yml
        
        # CSI Secret erstellen
        kubectl -n kube-system create secret generic hcloud-csi \\
            --from-literal=token=$HCLOUD_TOKEN || true
EOF
    
    log_success "Hetzner Cloud Controller Manager installiert"
}

# =============================================================================
# LOAD BALANCER SETUP
# =============================================================================

create_load_balancer() {
    log_info "Erstelle Load Balancer..."
    
    # Load Balancer erstellen falls nicht vorhanden
    if ! hcloud load-balancer list | grep -q "k8s-lb"; then
        log_info "Erstelle Load Balancer..."
        hcloud load-balancer create \
            --name "k8s-lb" \
            --type "$LB_TYPE" \
            --location "$LOCATION" \
            --network "$NETWORK_NAME"
        
        # Kurz warten bis Load Balancer bereit ist
        sleep 15
        
        # HTTP Service hinzufügen
        hcloud load-balancer add-service k8s-lb \
            --protocol http \
            --listen-port 80 \
            --destination-port 30080 \
            --proxyprotocol
        
        # HTTPS Service hinzufügen
        hcloud load-balancer add-service k8s-lb \
            --protocol https \
            --listen-port 443 \
            --destination-port 30443 \
            --proxyprotocol
        
        # Worker Nodes als Targets hinzufügen
        for i in $(seq 1 $WORKER_COUNT); do
            hcloud load-balancer add-target k8s-lb \
                --type server \
                --server "k8s-worker-$i"
        done
    fi
    
    LB_IP=$(hcloud load-balancer describe k8s-lb -o json | grep -o '"public_net":{"enabled":true,"ipv4":{"ip":"[^"]*"' | grep -o '[0-9.]*')
    log_success "Load Balancer erstellt - IP: $LB_IP"
}

# =============================================================================
# KUBECONFIG SETUP
# =============================================================================

setup_local_kubeconfig() {
    log_info "Richte lokale kubeconfig ein..."
    
    MASTER_IP=$(hcloud server ip k8s-master)
    
    # kubeconfig herunterladen
    scp -i "$SSH_KEY_PATH" -o StrictHostKeyChecking=no root@"$MASTER_IP":/etc/kubernetes/admin.conf "$HOME/.kube/hetzner-k8s-config"
    
    # Server IP in kubeconfig aktualisieren
    sed -i.bak "s/6443/6443/g" "$HOME/.kube/hetzner-k8s-config"
    sed -i.bak "s/10\.0\.1\.10/$MASTER_IP/g" "$HOME/.kube/hetzner-k8s-config"
    
    log_success "kubeconfig heruntergeladen: ~/.kube/hetzner-k8s-config"
    log_info "Verwende: export KUBECONFIG=~/.kube/hetzner-k8s-config"
}

# =============================================================================
# BEISPIEL-DEPLOYMENT
# =============================================================================

deploy_example_app() {
    log_info "Deploye Beispiel-Anwendung..."
    
    MASTER_IP=$(hcloud server ip k8s-master)
    
    ssh -i "$SSH_KEY_PATH" -o StrictHostKeyChecking=no root@"$MASTER_IP" << 'EOF'
        set -e
        
        # Nginx Deployment erstellen
        cat > /tmp/nginx-example.yaml << 'EOYAML'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-example
  namespace: default
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx-example
  template:
    metadata:
      labels:
        app: nginx-example
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "64Mi"
            cpu: "50m"
          limits:
            memory: "128Mi"
            cpu: "100m"
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-example-service
  namespace: default
spec:
  type: NodePort
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080
    protocol: TCP
  selector:
    app: nginx-example
EOYAML
        
        kubectl apply -f /tmp/nginx-example.yaml
        
        # Warten bis Pods bereit sind
        kubectl wait --for=condition=ready pod -l app=nginx-example --timeout=300s
EOF
    
    log_success "Beispiel-Anwendung deployed"
}

# =============================================================================
# CLUSTER STATUS
# =============================================================================

show_cluster_status() {
    log_info "Zeige Cluster-Status..."
    
    MASTER_IP=$(hcloud server ip k8s-master)
    LB_IP=$(hcloud load-balancer describe k8s-lb -o json | grep -o '"public_net":{"enabled":true,"ipv4":{"ip":"[^"]*"' | grep -o '[0-9.]*')
    
    echo ""
    log_success "=== KUBERNETES CLUSTER ERFOLGREICH EINGERICHTET ==="
    echo ""
    echo -e "${GREEN}Cluster Informationen:${NC}"
    echo -e "  Master Node IP:    $MASTER_IP"
    echo -e "  Load Balancer IP:  $LB_IP"
    echo -e "  kubeconfig:        ~/.kube/hetzner-k8s-config"
    echo ""
    echo -e "${GREEN}Nächste Schritte:${NC}"
    echo -e "  1. export KUBECONFIG=~/.kube/hetzner-k8s-config"
    echo -e "  2. kubectl get nodes"
    echo -e "  3. kubectl get pods -A"
    echo -e "  4. Besuche http://$LB_IP für die Beispiel-App"
    echo ""
    echo -e "${GREEN}Cluster-Verwaltung:${NC}"
    echo -e "  SSH Master:   ssh -i $SSH_KEY_PATH root@$MASTER_IP"
    echo -e "  Cluster Info: kubectl cluster-info"
    echo -e "  Node Status:  kubectl get nodes -o wide"
    echo ""
    echo -e "${YELLOW}Geschätzte monatliche Kosten: ~23€${NC}"
    echo ""
    
    # Cluster Status anzeigen
    ssh -i "$SSH_KEY_PATH" -o StrictHostKeyChecking=no root@"$MASTER_IP" << 'EOF'
        echo "=== CLUSTER STATUS ==="
        kubectl get nodes -o wide
        echo ""
        echo "=== RUNNING PODS ==="
        kubectl get pods -A
EOF
}

# =============================================================================
# CLEANUP FUNKTION
# =============================================================================

cleanup_cluster() {
    log_warning "Lösche Kubernetes-Cluster..."
    read -p "Bist du sicher? Alle Ressourcen werden gelöscht! (y/N): " -n 1 -r
    echo
    if [[ $REPLY =~ ^[Yy]$ ]]; then
        # Load Balancer löschen
        if hcloud load-balancer list | grep -q "k8s-lb"; then
            hcloud load-balancer delete k8s-lb
        fi
        
        # Server löschen
        hcloud server delete k8s-master || true
        for i in $(seq 1 $WORKER_COUNT); do
            hcloud server delete "k8s-worker-$i" || true
        done
        
        # Netzwerk löschen
        if hcloud network list | grep -q "$NETWORK_NAME"; then
            hcloud network delete "$NETWORK_NAME"
        fi
        
        # SSH-Key löschen
        if hcloud ssh-key list | grep -q "$SSH_KEY_NAME"; then
            hcloud ssh-key delete "$SSH_KEY_NAME"
        fi
        
        log_success "Cluster gelöscht"
    fi
}

# =============================================================================
# HAUPTFUNKTION
# =============================================================================

main() {
    echo -e "${BLUE}"
    echo "==============================================="
    echo "  Hetzner Cloud Kubernetes Setup Script"
    echo "==============================================="
    echo -e "${NC}"
    
    case "${1:-setup}" in
        "setup")
            check_requirements
            setup_hcloud_context
            setup_ssh_key
            setup_network
            create_servers
            
            # Kubernetes auf allen Servern installieren
            MASTER_IP=$(hcloud server ip k8s-master)
            install_kubernetes_dependencies "$MASTER_IP" "k8s-master"
            
            for i in $(seq 1 $WORKER_COUNT); do
                WORKER_IP=$(hcloud server ip "k8s-worker-$i")
                install_kubernetes_dependencies "$WORKER_IP" "k8s-worker-$i"
            done
            
            initialize_master_node
            join_worker_nodes
            install_hcloud_ccm
            create_load_balancer
            setup_local_kubeconfig
            deploy_example_app
            show_cluster_status
            ;;
        "status")
            show_cluster_status
            ;;
        "cleanup")
            cleanup_cluster
            ;;
        *)
            echo "Usage: $0 {setup|status|cleanup}"
            echo "  setup   - Erstelle komplettes Kubernetes-Cluster"
            echo "  status  - Zeige Cluster-Status"
            echo "  cleanup - Lösche alle Ressourcen"
            exit 1
            ;;
    esac
}

# Script ausführen
main "$@"
