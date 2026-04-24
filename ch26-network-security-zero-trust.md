# Chapter 26 — Network Security & Zero-Trust for AI Cluster Fabrics

**Part IV: Overlay & Kubernetes Networking** | ~20 pages

---

## Installation

### System packages

Zero-trust tooling spans WireGuard kernel modules, IPsec daemons, and PKI utilities, all available in the Ubuntu 24.04 main archive.

```bash
sudo apt install -y \
    wireguard \
    wireguard-tools \
    strongswan \
    strongswan-swanctl \
    strongswan-pki \
    libcharon-extra-plugins \
    openssl \
    iproute2 \
    tcpdump \
    iperf3 \
    curl \
    jq
```

Verify key packages:

```bash
wg --version
# Expected: wireguard-tools v1.0.20210914 or newer

swanctl --version
# Expected: Linux swanctl 5.9.x or newer

openssl version
# Expected: OpenSSL 3.0.x (Ubuntu 24.04 ships 3.0.13+)

modinfo wireguard | grep version
# Expected: version: 1  (built into the 6.8 kernel — no external module needed)
```

### Kubernetes tooling (Kind + kubectl + Helm + Cilium CLI)

```bash
# kind — local Kubernetes clusters in Docker
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.22.0/kind-linux-amd64
chmod +x ./kind && sudo mv ./kind /usr/local/bin/kind
kind --version
# Expected: kind version 0.22.0

# kubectl
curl -LO "https://dl.k8s.io/release/$(curl -Ls https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl && sudo mv kubectl /usr/local/bin/
kubectl version --client
# Expected: Client Version: v1.30.x

# Helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version --short
# Expected: v3.15.x+gXXXXXXX

# Cilium CLI
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)
curl -L --remote-name-all \
    https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-amd64.tar.gz
tar xzvf cilium-linux-amd64.tar.gz
sudo mv cilium /usr/local/bin/
cilium version --client
# Expected: cilium-cli: v0.16.x ...

# SPIRE Helm chart (add repo)
helm repo add spiffe https://spiffe.github.io/helm-charts-hardened/
helm repo update

# cert-manager Helm chart
helm repo add jetstack https://charts.jetstack.io
helm repo update
```

### Python scripting environment (uv)

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
source $HOME/.local/bin/env

uv venv .venv
source .venv/bin/activate

uv pip install cryptography hvac requests
```

Verify:

```bash
python -c "from cryptography.hazmat.primitives.asymmetric import ec; print('cryptography OK')"
python -c "import hvac; print('hvac OK')"
# Expected: cryptography OK
#           hvac OK
```

---

## 26.1 Threat Model for AI Cluster Fabrics

AI GPU clusters present a threat surface that differs qualitatively from traditional enterprise environments. The combination of ultra-high-bandwidth RDMA fabrics, shared multi-tenant scheduling, and the enormous value of trained model weights creates specific attack vectors that conventional perimeter security cannot address.

### Lateral Movement via the RDMA Fabric

RDMA bypasses the host kernel — there is no TCP socket, no firewall hook, and no standard Linux netfilter chain between two RDMA endpoints. An attacker who gains InfiniBand or RoCEv2 access on one node can:

- Read or write GPU device memory directly via RDMA without any OS-level logging
- Pivot between nodes using RDMA verbs if queue pairs are not access-controlled
- Intercept gradient tensors mid-AllReduce if the fabric is unencrypted

RoCEv2 is UDP-encapsulated and thus susceptible to packet injection from anywhere on the same VLAN. The consequence: the fabric VLAN is effectively a trust boundary — and any node in that VLAN must be treated as potentially hostile.

### Credential Theft from NIC Firmware

Modern NICs (Mellanox ConnectX-7, Broadcom P5) have out-of-band management interfaces (BMC, IPMI) and embedded ARM cores running firmware. Vulnerabilities in this firmware have enabled:

- Extraction of TLS private keys cached in NIC firmware from previous workloads
- Persistent backdoors surviving host OS reinstallation
- Interception of PCIe DMA traffic destined for GPUs

Defense: NIC firmware must be patched and version-pinned; IPMI networks must be isolated on a dedicated OOB VLAN with strict ACLs.

### Management Plane Exposure

The management plane (SONiC ConfigDB API, gNMI streams, Kubernetes API server) controls the fabric. Compromise of any management-plane credential grants the ability to redirect traffic, change routing policies, or disable monitoring. Common exposure vectors:

- Kubernetes API server tokens embedded in training job manifests and leaked via model checkpoints
- gNMI credentials stored in plaintext in automation scripts
- Vault unsealing keys present in CI/CD pipeline environment variables

### Summary Threat Matrix

| Threat | Vector | Impact |
|---|---|---|
| RDMA eavesdropping | Unencrypted fabric VLAN | Model weight theft |
| RDMA injection | Spoofed RoCEv2 UDP | Training corruption |
| NIC firmware exploit | BMC/IPMI access | Persistent backdoor |
| API token leakage | Log files, checkpoints | Full cluster takeover |
| Lateral movement | Flat management VLAN | Multi-tenant data exfiltration |

---

## 26.2 Zero-Trust Principles Applied to the Fabric

Zero trust collapses the implicit "inside = trusted" assumption. Every network communication must be authenticated and authorized, regardless of source IP address. Applied to AI cluster fabrics:

### Identity, Not IP Address

In a Kubernetes cluster, pod IP addresses change with every restart. In a fabric with dynamic routing, the ingress point of a flow changes with every ECMP path change. IP address is not a stable identity anchor. Instead:

- **Workload identity**: SPIFFE URI encoded in an X.509 SVID (`spiffe://cluster.local/ns/training/sa/nccl-worker`)
- **Node identity**: Cilium assigns a numeric identity derived from the pod's labels and namespace, enforced by its eBPF data plane
- **Service account tokens**: Kubernetes-signed JWTs bound to a specific pod and service account

### Mutual Authentication

Every connection — node-to-node, pod-to-pod, management API call — must be authenticated by both parties:

- **WireGuard**: static key pairs exchanged out-of-band; each peer cryptographically authenticates to the other before any data flows
- **mTLS**: both client and server present X.509 certificates; the server verifies the client certificate against a trusted CA
- **IPsec IKEv2**: full certificate-based mutual authentication before any ESP-encrypted data plane traffic flows

### Least-Privilege Network Policy

A GPU worker pod needs to reach: its AllReduce peers, the object store endpoint, and the Kubernetes API. It does not need to reach the IPMI network, other tenants' namespaces, or the NOS management API. Cilium `CiliumNetworkPolicy` enforces this at the eBPF level with sub-microsecond overhead.

```
Zero-Trust Stack (bottom-up):
  ┌─────────────────────────────────────────┐
  │  Network Policy (Cilium identity-based) │  ← who can talk to whom
  │  mTLS / SPIFFE (service-to-service)     │  ← are you who you claim?
  │  WireGuard / IPsec (node-to-node)       │  ← is the wire encrypted?
  │  NIC firmware verified boot             │  ← is the hardware trustworthy?
  └─────────────────────────────────────────┘
```

---

## 26.3 Cilium Transparent Encryption

Cilium supports two transparent encryption modes: WireGuard and IPsec. "Transparent" means application pods require zero configuration changes — encryption is inserted at the node level by Cilium's eBPF data plane.

### WireGuard Mode

WireGuard mode uses the Linux `wireguard` kernel module. Cilium manages key generation and peer configuration automatically for each node in the cluster:

- Each node gets a WireGuard interface (`cilium_wg0`) managed by Cilium
- Cilium distributes each node's WireGuard public key via a Kubernetes CiliumNode CRD
- All pod-to-pod traffic crossing a node boundary is encrypted before leaving the host NIC

```bash
# Install Cilium with WireGuard transparent encryption
cilium install \
    --set encryption.enabled=true \
    --set encryption.type=wireguard \
    --set encryption.wireguard.userspaceFallback=false

# Wait for rollout
cilium status --wait
# Expected:
#     /¯¯\
#  /¯¯\__/¯¯\    Cilium:             OK
#  \__/¯¯\__/    Operator:           OK
#  /¯¯\__/¯¯\    Envoy DaemonSet:    disabled
#  \__/¯¯\__/    Hubble Relay:       disabled
#     \__/
# DaemonSet              cilium             Desired: 3, Ready: 3/3, Available: 3/3
# Deployment             cilium-operator    Desired: 1, Ready: 1/1, Available: 1/1
```

### Verifying Encryption Status

```bash
cilium encrypt status
# Expected output:
# Encryption: WireGuard
# Decryption interface(s): eth0, eth1
# Keys in use: 1
# Max Seq. Number: 0x0/0xffffffff
# Errors: 0
#
# Node       Public Key                                    Endpoint
# node1      +xNqo8TL9PGGxKO8EpKlFqOEJq/Z4F4qWfGfnpXXXXs=  10.0.0.1
# node2      R4mQwEXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX=  10.0.0.2
# node3      8kPlXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX=  10.0.0.3
```

### IPsec Mode

IPsec mode uses the Linux xfrm framework. Cilium pre-shares a symmetric key and programs xfrm states/policies for every node pair:

```bash
# Generate a pre-shared key (256-bit AES-CBC or AES-GCM)
PSK=$(dd if=/dev/urandom bs=20 count=1 2>/dev/null | xxd -p -c 40)
kubectl create secret generic cilium-ipsec-keys \
    --namespace kube-system \
    --from-literal=keys="3 rfc4106(gcm(aes)) ${PSK} 128"

# Install Cilium with IPsec
cilium install \
    --set encryption.enabled=true \
    --set encryption.type=ipsec \
    --set encryption.ipsec.keyFile=keys

cilium encrypt status
# Expected:
# Encryption: IPsec
# Keys in use: 1
# Key rotation phase: Normal
```

### Key Rotation

```bash
# WireGuard key rotation — Cilium regenerates keys on all nodes atomically
cilium encrypt rotate-key

# Expected: Key rotation initiated. Monitor with:
cilium encrypt status
# Keys in use: 2  (briefly, during rotation)
# ... then back to:
# Keys in use: 1

# IPsec key rotation — update the secret, Cilium picks it up
NEW_PSK=$(dd if=/dev/urandom bs=20 count=1 2>/dev/null | xxd -p -c 40)
kubectl patch secret cilium-ipsec-keys -n kube-system \
    --type=json \
    -p="[{\"op\":\"replace\",\"path\":\"/data/keys\",\"value\":\"$(echo -n "4 rfc4106(gcm(aes)) ${NEW_PSK} 128" | base64 -w0)\"}]"
```

---

## 26.4 WireGuard for Overlay Encryption

WireGuard is a modern VPN protocol integrated into the Linux kernel since 5.6. It uses Curve25519 for key exchange, ChaCha20-Poly1305 for authenticated encryption, and has a codebase of under 4,000 lines — an order of magnitude smaller than OpenVPN or IPsec stacks.

### Key Concepts

- **Static key pairs**: each peer has a 256-bit Curve25519 private key and a corresponding public key
- **Preshared key (optional)**: adds a 256-bit symmetric key for post-quantum resistance
- **AllowedIPs**: the set of IP prefixes a peer is authorised to route — acts as both routing table and ACL
- **Persistent keepalive**: essential for NAT traversal; sends a keepalive every N seconds to maintain the UDP hole punch

### Manual Configuration

```bash
# Load kernel module (already in-kernel on Ubuntu 24.04)
sudo modprobe wireguard
lsmod | grep wireguard
# Expected: wireguard              69632  0

# Generate key pair for node A
wg genkey | tee /tmp/node_a_private.key | wg pubkey > /tmp/node_a_public.key
cat /tmp/node_a_public.key
# Expected: base64-encoded 44-character string, e.g.:
# +xNqo8TL9PGGxKO8EpKlFqOEJq/Z4F4qWfGfnpXXXXs=

# Generate key pair for node B
wg genkey | tee /tmp/node_b_private.key | wg pubkey > /tmp/node_b_public.key

# Create the wg0 interface on node A
sudo ip link add wg0 type wireguard
sudo ip addr add 10.99.0.1/24 dev wg0
sudo wg set wg0 \
    listen-port 51820 \
    private-key /tmp/node_a_private.key \
    peer $(cat /tmp/node_b_public.key) \
        allowed-ips 10.99.0.2/32 \
        endpoint 192.168.1.2:51820 \
        persistent-keepalive 25
sudo ip link set wg0 up

# Verify
sudo wg show wg0
# Expected:
# interface: wg0
#   public key: +xNqo8TL9PGGxKO8EpKlFqOEJq/Z4F4qWfGfnpXXXXs=
#   private key: (hidden)
#   listening port: 51820
#
# peer: <node_b_pubkey>
#   endpoint: 192.168.1.2:51820
#   allowed ips: 10.99.0.2/32
#   latest handshake: (none yet)
#   persistent keepalive: every 25 seconds
```

### wg-quick Configuration File

For persistent configuration, `wg-quick` reads `/etc/wireguard/wg0.conf`:

```ini
[Interface]
PrivateKey = <node_a_private_key_base64>
Address    = 10.99.0.1/24
ListenPort = 51820
# Optional: PostUp/PreDown hooks for firewall rules
PostUp   = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PreDown  = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE

[Peer]
PublicKey           = <node_b_public_key_base64>
Endpoint            = 192.168.1.2:51820
AllowedIPs          = 10.99.0.2/32
PersistentKeepalive = 25
```

```bash
sudo wg-quick up wg0
# Expected:
# [#] ip link add wg0 type wireguard
# [#] wg setconf wg0 /dev/fd/63
# [#] ip -4 address add 10.99.0.1/24 dev wg0
# [#] ip link set mtu 1420 up dev wg0
# [#] iptables ...

sudo wg-quick down wg0
# Tears down the interface and reverses PostUp changes
```

---

## 26.5 IPsec with strongSwan

strongSwan is the dominant open-source IKEv2 implementation on Linux. It provides:

- **IKEv2**: the key exchange protocol — negotiates cipher suites and exchanges certificates
- **ESP** (Encapsulating Security Payload): the data-plane encryption protocol
- **Transport mode**: encrypts only the payload; original IP headers are preserved (suitable for host-to-host)
- **Tunnel mode**: encrypts the entire original packet and wraps it in a new IP header (suitable for gateway-to-gateway)

### swanctl Configuration

Modern strongSwan uses `swanctl` with `/etc/swanctl/swanctl.conf`:

```ini
connections {
    node-a-to-node-b {
        version    = 2          # IKEv2
        local_addrs  = 192.168.1.1
        remote_addrs = 192.168.1.2

        local {
            auth = pubkey
            certs = node_a_cert.pem
            id = "CN=node-a.cluster.local"
        }
        remote {
            auth = pubkey
            id = "CN=node-b.cluster.local"
        }

        children {
            node-tunnel {
                local_ts  = 10.0.0.0/24    # protect traffic from this prefix
                remote_ts = 10.0.1.0/24    # to this prefix
                esp_proposals = aes256gcm128-sha256-ecp256
                mode = tunnel              # or transport for host-to-host
                start_action = trap        # initiate on first matching packet
            }
        }
        proposals = aes256-sha256-ecp256
    }
}

secrets {
    # Private key for the local certificate
    private-node-a {
        file = node_a_key.pem
    }
}
```

```bash
# Load and verify configuration
sudo swanctl --load-all
# Expected:
# loaded connection 'node-a-to-node-b'
# loaded secret 'private-node-a'

# Initiate the IKE SA
sudo swanctl --initiate --child node-tunnel
# Expected:
# [IKE] initiating IKE_SA node-a-to-node-b[1] to 192.168.1.2
# [IKE] IKE_SA node-a-to-node-b[1] established between 192.168.1.1[node-a]...192.168.1.2[node-b]
# [IKE] CHILD_SA node-tunnel{1} established with SPIs ...

# List active SAs
sudo swanctl --list-sas
# Expected:
# node-a-to-node-b: #1, ESTABLISHED, IKEv2, ...
#   node-tunnel: #1, reqid 1, INSTALLED, TUNNEL, ...
#     local  10.0.0.0/24
#     remote 10.0.1.0/24
#     AES_GCM_16-128/PRF_HMAC_SHA2_256/ECP_256

# View xfrm states created by strongSwan
sudo ip xfrm state list
# Expected:
# src 192.168.1.1 dst 192.168.1.2
#     proto esp spi 0xXXXXXXXX reqid 1 mode tunnel
#     auth-trunc hmac(sha256) ...
#     enc cbc(aes) ...
```

### Certificate Generation for IKEv2

```bash
# Generate CA key and self-signed certificate
pki --gen --type ec --size 256 --outform pem > ca_key.pem
pki --self --ca --lifetime 3650 \
    --in ca_key.pem --type ec \
    --dn "CN=AI Cluster CA" --outform pem > ca_cert.pem

# Generate node certificate signed by the CA
pki --gen --type ec --size 256 --outform pem > node_a_key.pem
pki --req --in node_a_key.pem --type ec \
    --dn "CN=node-a.cluster.local" --outform pem > node_a_req.pem
pki --issue --lifetime 365 \
    --cacert ca_cert.pem --cakey ca_key.pem \
    --in node_a_req.pem --type pkcs10 \
    --san "node-a.cluster.local" --outform pem > node_a_cert.pem

# Install into strongSwan directories
sudo cp ca_cert.pem   /etc/swanctl/x509ca/
sudo cp node_a_cert.pem /etc/swanctl/x509/
sudo cp node_a_key.pem  /etc/swanctl/private/
sudo chmod 600 /etc/swanctl/private/node_a_key.pem
```

---

## 26.6 mTLS Between Services: SPIFFE/SPIRE

SPIFFE (Secure Production Identity Framework for Everyone) provides a portable workload identity standard for zero-trust environments. The key components:

- **SPIFFE ID**: a URI uniquely identifying a workload: `spiffe://trust-domain/path`
  - Example: `spiffe://cluster.local/ns/training/sa/nccl-worker`
- **SVID** (SPIFFE Verifiable Identity Document): an X.509 certificate or JWT carrying the SPIFFE ID in the Subject Alternative Name field
- **SPIRE**: the reference implementation — a server/agent pair that issues SVIDs to workloads

### SPIRE Architecture in Kubernetes

```
  ┌──────────────────────────────────┐
  │  SPIRE Server (Deployment)       │
  │  - issues SVIDs                  │
  │  - maintains trust bundles       │
  │  - attests agent nodes           │
  └──────────────┬───────────────────┘
                 │ gRPC (mTLS)
  ┌──────────────▼───────────────────┐
  │  SPIRE Agent (DaemonSet)         │
  │  - attests workloads via k8s API │
  │  - delivers SVIDs via UNIX socket│
  │  - /run/spire/sockets/agent.sock │
  └──────────────────────────────────┘
           │ (workloads fetch SVIDs
           │  via Workload API)
  ┌────────▼────────┐
  │  Training Pod   │
  │  (nccl-worker)  │
  └─────────────────┘
```

### SPIRE Deployment

```bash
# Deploy SPIRE via Helm
helm install spire spiffe/spire \
    --namespace spire-system \
    --create-namespace \
    --set global.spire.trustDomain=cluster.local \
    --set spire-server.replicaCount=1 \
    --set spire-agent.enabled=true

# Wait for rollout
kubectl rollout status daemonset/spire-agent -n spire-system
# Expected: daemon set "spire-agent" successfully rolled out

# List SPIRE server entries
kubectl exec -n spire-system deployment/spire-server -- \
    spire-server entry show
# Expected: (empty until entries are registered)

# Register a workload entry for nccl-worker pods
kubectl exec -n spire-system deployment/spire-server -- \
    spire-server entry create \
        -parentID "spiffe://cluster.local/ns/spire-system/sa/spire-agent" \
        -spiffeID "spiffe://cluster.local/ns/training/sa/nccl-worker" \
        -selector "k8s:ns:training" \
        -selector "k8s:sa:nccl-worker"
# Expected:
# Entry ID         : xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
# SPIFFE ID        : spiffe://cluster.local/ns/training/sa/nccl-worker
# Parent ID        : spiffe://cluster.local/ns/spire-system/sa/spire-agent
# TTL              : default
# Selector         : k8s:ns:training
# Selector         : k8s:sa:nccl-worker
```

### cert-manager Integration

cert-manager provides automated X.509 lifecycle management in Kubernetes, backed by various issuers (Let's Encrypt, Vault, self-signed):

```bash
# Install cert-manager
helm install cert-manager jetstack/cert-manager \
    --namespace cert-manager \
    --create-namespace \
    --set crds.enabled=true

kubectl get pods -n cert-manager
# Expected:
# NAME                                       READY   STATUS    RESTARTS   AGE
# cert-manager-xxxxxxxx-xxxxx               1/1     Running   0          60s
# cert-manager-cainjector-xxxxxxxx-xxxxx    1/1     Running   0          60s
# cert-manager-webhook-xxxxxxxx-xxxxx       1/1     Running   0          60s
```

```yaml
# cluster-issuer.yaml — self-signed CA for the lab
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned-cluster-issuer
spec:
  selfSigned: {}
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: cluster-ca
  namespace: cert-manager
spec:
  isCA: true
  commonName: ai-cluster-ca
  secretName: cluster-ca-secret
  privateKey:
    algorithm: ECDSA
    size: 256
  issuerRef:
    name: selfsigned-cluster-issuer
    kind: ClusterIssuer
    group: cert-manager.io
```

```bash
kubectl apply -f cluster-issuer.yaml
kubectl get certificate -n cert-manager
# Expected:
# NAME         READY   SECRET               AGE
# cluster-ca   True    cluster-ca-secret    10s
```

---

## 26.7 Secrets Management with HashiCorp Vault

Vault provides dynamic secrets — short-lived credentials generated on demand and automatically revoked. For AI clusters, the critical use cases are:

- **NIC credentials**: rotating API keys for NIC firmware management interfaces
- **PKI for management APIs**: issuing short-lived certificates for gNMI/RESTCONF endpoints
- **Database passwords**: Kubernetes operators and training jobs that need object store access

### Vault PKI Secrets Engine

```bash
# Install Vault CLI
curl -fsSL https://apt.releases.hashicorp.com/gpg | \
    sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] \
    https://apt.releases.hashicorp.com $(lsb_release -cs) main" | \
    sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install -y vault

# Start a dev Vault server (for lab purposes)
vault server -dev -dev-root-token-id="root" &
export VAULT_ADDR='http://127.0.0.1:8200'
export VAULT_TOKEN='root'

vault status
# Expected:
# Key             Value
# ---             -----
# Seal Type       shamir
# Initialized     true
# Sealed          false
# Version         1.16.x
# HA Enabled      false

# Enable the PKI secrets engine
vault secrets enable pki
vault secrets tune -max-lease-ttl=8760h pki

# Generate root CA inside Vault
vault write pki/root/generate/internal \
    common_name="AI Cluster Root CA" \
    issuer_name="root-2024" \
    ttl=87600h
# Expected:
# Key              Value
# ---              -----
# issuer_id        xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
# issuer_name      root-2024
# issuing_ca       -----BEGIN CERTIFICATE-----...

# Configure PKI role for NIC management certificates
vault write pki/roles/nic-management \
    allowed_domains="mgmt.cluster.local" \
    allow_subdomains=true \
    max_ttl=72h \
    key_type=ec \
    key_bits=256

# Issue a certificate for a NIC management API
vault write pki/issue/nic-management \
    common_name="switch01.mgmt.cluster.local" \
    ttl=24h
# Expected:
# Key                 Value
# ---                 -----
# ca_chain            [-----BEGIN CERTIFICATE-----...]
# certificate         -----BEGIN CERTIFICATE-----...
# expiration          1719360000
# issuing_ca          -----BEGIN CERTIFICATE-----...
# private_key         -----BEGIN EC PRIVATE KEY-----...
# private_key_type    ec
# serial_number       XX:XX:XX:XX:XX:XX:XX:XX:XX:XX:XX:XX:XX:XX:XX:XX
```

### Vault Python Client (hvac)

```python
# vault_nic_creds.py — issue and rotate NIC management certificates
import hvac
import json
from datetime import datetime

client = hvac.Client(url='http://127.0.0.1:8200', token='root')
assert client.is_authenticated(), "Vault authentication failed"

def issue_nic_cert(hostname: str, ttl: str = "24h") -> dict:
    """Issue a short-lived certificate for a NIC management endpoint."""
    response = client.secrets.pki.generate_certificate(
        name='nic-management',
        common_name=f'{hostname}.mgmt.cluster.local',
        extra_params={'ttl': ttl},
        mount_point='pki',
    )
    data = response['data']
    expiry = datetime.utcfromtimestamp(data['expiration'])
    print(f"Issued cert for {hostname}, expires: {expiry.isoformat()}Z")
    return data

def rotate_nic_certs(hostnames: list) -> None:
    """Rotate certificates for a list of NIC management interfaces."""
    for host in hostnames:
        cert_data = issue_nic_cert(host)
        # In production: push cert_data['certificate'] and
        # cert_data['private_key'] to the NIC management API
        print(f"  Serial: {cert_data['serial_number']}")

if __name__ == '__main__':
    switches = ['spine01', 'spine02', 'leaf01', 'leaf02']
    rotate_nic_certs(switches)
```

```bash
python vault_nic_creds.py
# Expected:
# Issued cert for spine01, expires: 2026-04-23T14:22:31Z
#   Serial: 2a:3b:4c:5d:6e:7f:8a:9b:0c:1d:2e:3f:4a:5b:6c:7d
# Issued cert for spine02, expires: 2026-04-23T14:22:31Z
#   Serial: 3b:4c:5d:6e:7f:8a:9b:0c:1d:2e:3f:4a:5b:6c:7d:8e
# Issued cert for leaf01, expires: 2026-04-23T14:22:31Z
#   Serial: 4c:5d:6e:7f:8a:9b:0c:1d:2e:3f:4a:5b:6c:7d:8e:9f
# Issued cert for leaf02, expires: 2026-04-23T14:22:31Z
#   Serial: 5d:6e:7f:8a:9b:0c:1d:2e:3f:4a:5b:6c:7d:8e:9f:0a
```

---

## 26.8 Network Policy as Code: Cilium CiliumNetworkPolicy

Cilium's `CiliumNetworkPolicy` (CNP) operates on workload identity — Cilium labels — rather than IP addresses. This makes policies stable across pod restarts and node reassignments.

### Identity-Based Policy

```yaml
# allow-nccl-allreduce.yaml
# Allow nccl-worker pods in the "training" namespace to communicate
# with each other on NCCL's default port range (10000-10099),
# and only allow egress to the object store.
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: nccl-worker-policy
  namespace: training
spec:
  endpointSelector:
    matchLabels:
      app: nccl-worker
  ingress:
    - fromEndpoints:
        - matchLabels:
            app: nccl-worker
            io.kubernetes.pod.namespace: training
      toPorts:
        - ports:
            - port: "10000"
              protocol: TCP
            - port: "10099"
              protocol: TCP
  egress:
    - toEndpoints:
        - matchLabels:
            app: nccl-worker
            io.kubernetes.pod.namespace: training
      toPorts:
        - ports:
            - port: "10000"
              protocol: TCP
    - toFQDNs:
        - matchPattern: "*.s3.amazonaws.com"
      toPorts:
        - ports:
            - port: "443"
              protocol: TCP
    # Allow DNS
    - toEndpoints:
        - matchLabels:
            k8s-app: kube-dns
            io.kubernetes.pod.namespace: kube-system
      toPorts:
        - ports:
            - port: "53"
              protocol: UDP
```

```bash
kubectl apply -f allow-nccl-allreduce.yaml
# Expected: ciliumnetworkpolicy.cilium.io/nccl-worker-policy created

# Check policy is enforced
kubectl get cnp -n training
# Expected:
# NAME                  AGE
# nccl-worker-policy    10s

# Verify via Cilium CLI
cilium policy get
# Expected: lists the policy with its identity selector and port rules
```

### Audit Mode

During migration, policies can be set to audit mode to log violations without dropping traffic:

```bash
# Enable audit mode globally (useful during policy authoring)
cilium config set PolicyAuditMode=true

# Check Hubble for policy audit events
cilium hubble observe --type policy-verdict --follow
# Expected (when a pod attempts a connection not permitted by policy):
# AUDIT  training/nccl-worker-0:52341 -> monitoring/prometheus-0:9090
#   policy-verdict:none AUDIT (denied)
#   Reason: policy-cidr / egress port-rule verdict: unknown
```

### Hubble Policy Visualization

```bash
# Deploy Hubble UI for visual policy map
cilium hubble enable --ui

# Port-forward to access the UI
kubectl port-forward -n kube-system svc/hubble-ui 12000:80 &

# Open http://localhost:12000 in a browser to see the live network map
# with policy verdicts color-coded (green=allowed, red=denied)

# CLI: observe flows between specific pods
cilium hubble observe \
    --namespace training \
    --from-pod nccl-worker-0 \
    --to-pod nccl-worker-1 \
    --last 20
# Expected:
# FORWARDED  training/nccl-worker-0:10001 -> training/nccl-worker-1:10001
#   TCP Flags: SYN
# FORWARDED  training/nccl-worker-0:10001 -> training/nccl-worker-1:10001
#   TCP Flags: ACK
```

---

## Lab Walkthrough 26 — WireGuard Encrypted Overlay + Cilium Transparent Encryption

This lab builds a complete zero-trust encrypted overlay in two phases: first a manual WireGuard tunnel between network namespaces on a single host (to understand the primitives), then Cilium transparent encryption on a Kind cluster (to see how it scales to a full cluster).

**Prerequisite check:**

```bash
which wg ip tcpdump iperf3 kind kubectl cilium
wg --version
# Expected: wireguard-tools v1.0.20210914

modinfo wireguard | grep -E "^(filename|version)"
# Expected:
# filename:       /lib/modules/6.8.0-xx-generic/kernel/drivers/net/wireguard/wireguard.ko.zst
# version:        1

# Confirm user has CAP_NET_ADMIN (or is root)
sudo -n ip link list &>/dev/null && echo "sudo OK" || echo "Need sudo access"
```

---

### Step 1 — Generate WireGuard Keypairs for Two Simulated Nodes

```bash
# Create a working directory
mkdir -p /tmp/wg-lab && cd /tmp/wg-lab

# Generate key pairs for "node-a" and "node-b"
# Each command: generate private key, write to file, derive public key from stdin
wg genkey > node_a.key && wg pubkey < node_a.key > node_a.pub
wg genkey > node_b.key && wg pubkey < node_b.key > node_b.pub

# Protect private keys
chmod 600 node_a.key node_b.key

# Display the public keys
echo "Node A public key: $(cat node_a.pub)"
echo "Node B public key: $(cat node_b.pub)"
# Expected (example — your keys will differ):
# Node A public key: +xNqo8TL9PGGxKO8EpKlFqOEJq/Z4F4qWfGfnpAbCdE=
# Node B public key: R4mQwEL9KHJiMNopQRsTUvWxYZabCdEfGhIjKlMnOpQ=
```

---

### Step 2 — Create Network Namespaces with veth Pair, Configure WireGuard Interfaces

```bash
# Create two network namespaces simulating separate nodes
sudo ip netns add node-a
sudo ip netns add node-b

# Create a veth pair connecting them (simulates the underlay network)
sudo ip link add veth-a type veth peer name veth-b
sudo ip link set veth-a netns node-a
sudo ip link set veth-b netns node-b

# Assign underlay addresses and bring interfaces up
sudo ip netns exec node-a ip addr add 192.168.100.1/24 dev veth-a
sudo ip netns exec node-a ip link set veth-a up
sudo ip netns exec node-a ip link set lo up

sudo ip netns exec node-b ip addr add 192.168.100.2/24 dev veth-b
sudo ip netns exec node-b ip link set veth-b up
sudo ip netns exec node-b ip link set lo up

# Verify underlay connectivity
sudo ip netns exec node-a ping -c 2 192.168.100.2
# Expected:
# PING 192.168.100.2 (192.168.100.2) 56(84) bytes of data.
# 64 bytes from 192.168.100.2: icmp_seq=1 ttl=64 time=0.087 ms
# 64 bytes from 192.168.100.2: icmp_seq=2 ttl=64 time=0.065 ms
# --- 192.168.100.2 ping statistics ---
# 2 packets transmitted, 2 received, 0% packet loss

# Create WireGuard interfaces inside each namespace
sudo ip netns exec node-a ip link add wg0 type wireguard
sudo ip netns exec node-a ip addr add 10.99.0.1/24 dev wg0

sudo ip netns exec node-b ip link add wg0 type wireguard
sudo ip netns exec node-b ip addr add 10.99.0.2/24 dev wg0

# Configure WireGuard in node-a: listen on 51820, use node-a's private key
sudo ip netns exec node-a \
    wg set wg0 listen-port 51820 private-key /tmp/wg-lab/node_a.key

# Configure WireGuard in node-b: listen on 51821
sudo ip netns exec node-b \
    wg set wg0 listen-port 51821 private-key /tmp/wg-lab/node_b.key

# Bring the WireGuard interfaces up
sudo ip netns exec node-a ip link set wg0 up
sudo ip netns exec node-b ip link set wg0 up

echo "WireGuard interfaces created in both namespaces"
```

---

### Step 3 — Exchange Public Keys and Configure Peers with AllowedIPs

```bash
NODE_A_PUB=$(cat /tmp/wg-lab/node_a.pub)
NODE_B_PUB=$(cat /tmp/wg-lab/node_b.pub)

# Add node-b as a peer of node-a
# endpoint: underlay address of node-b, port 51821
# allowed-ips: the WireGuard tunnel address of node-b
sudo ip netns exec node-a \
    wg set wg0 \
        peer "${NODE_B_PUB}" \
        endpoint 192.168.100.2:51821 \
        allowed-ips 10.99.0.2/32 \
        persistent-keepalive 5

# Add node-a as a peer of node-b
sudo ip netns exec node-b \
    wg set wg0 \
        peer "${NODE_A_PUB}" \
        endpoint 192.168.100.1:51820 \
        allowed-ips 10.99.0.1/32 \
        persistent-keepalive 5

echo "Peer configurations applied."
echo "node-a peers:"
sudo ip netns exec node-a wg show wg0 peers
echo "node-b peers:"
sudo ip netns exec node-b wg show wg0 peers
# Expected: each shows the other's public key on a single line
```

---

### Step 4 — wg show: Verify Handshake

```bash
# Trigger a handshake by sending a packet
sudo ip netns exec node-a ping -c 1 10.99.0.2

# Now check wg show on both sides
sudo ip netns exec node-a wg show wg0
# Expected:
# interface: wg0
#   public key: +xNqo8TL9PGGxKO8EpKlFqOEJq/Z4F4qWfGfnpAbCdE=
#   private key: (hidden)
#   listening port: 51820
#
# peer: R4mQwEL9KHJiMNopQRsTUvWxYZabCdEfGhIjKlMnOpQ=
#   endpoint: 192.168.100.2:51821
#   allowed ips: 10.99.0.2/32
#   latest handshake: 2 seconds ago          <-- this confirms the tunnel is live
#   transfer: 148 B received, 180 B sent
#   persistent keepalive: every 5 seconds

sudo ip netns exec node-b wg show wg0
# Expected: mirror output showing node-a as peer with recent handshake timestamp
```

---

### Step 5 — Ping Through the Encrypted Tunnel; tcpdump Confirms No Plaintext

```bash
# Start tcpdump on the underlay veth (captures the encrypted WireGuard UDP packets)
# Run in background, capture 20 packets
sudo ip netns exec node-a \
    tcpdump -i veth-a -c 20 -w /tmp/wg-lab/capture.pcap &
TCPDUMP_PID=$!

# Send test traffic through the WireGuard tunnel
sudo ip netns exec node-a ping -c 5 10.99.0.2

# Wait for tcpdump to finish
wait ${TCPDUMP_PID}

# Inspect the capture: verify it is UDP on port 51821 (WireGuard)
tcpdump -r /tmp/wg-lab/capture.pcap -nn
# Expected output (each packet is UDP, not ICMP):
# 14:22:31.001234 IP 192.168.100.1.51820 > 192.168.100.2.51821: UDP, length 128
# 14:22:31.001456 IP 192.168.100.2.51821 > 192.168.100.1.51820: UDP, length 128
# ...
# (NO "ICMP echo request" lines — the ICMP is encrypted inside the WireGuard UDP payload)

# Confirm no plaintext ICMP is visible
tcpdump -r /tmp/wg-lab/capture.pcap -nn 'icmp' | wc -l
# Expected: 0  (all ICMP is encrypted and invisible to the outer network)

# Optionally: hexdump one packet to confirm it is ciphertext
tcpdump -r /tmp/wg-lab/capture.pcap -nn -x | head -20
# Expected: random-looking hex bytes with no recognizable plaintext patterns
```

---

### Step 6 — Measure WireGuard Overhead with iperf3

```bash
# Start iperf3 server in node-b namespace
sudo ip netns exec node-b iperf3 -s -D --logfile /tmp/wg-lab/iperf_server.log
sleep 1

# Baseline: iperf3 over the unencrypted underlay veth
sudo ip netns exec node-a iperf3 -c 192.168.100.2 -t 10 -P 4
# Expected (example on a modern server):
# [SUM]   0.00-10.00  sec  19.8 GBytes  17.0 Gbits/sec    0   sender
# [SUM]   0.00-10.00  sec  19.7 GBytes  16.9 Gbits/sec         receiver

# Encrypted: iperf3 over the WireGuard tunnel
sudo ip netns exec node-a iperf3 -c 10.99.0.2 -t 10 -P 4
# Expected:
# [SUM]   0.00-10.00  sec  18.2 GBytes  15.6 Gbits/sec    0   sender
# [SUM]   0.00-10.00  sec  18.1 GBytes  15.5 Gbits/sec         receiver
#
# Overhead: ~(17.0 - 15.6) / 17.0 * 100 = ~8.2%
# This is within the expected 5-10% range for WireGuard on a CPU without
# AES-NI-accelerated ChaCha20 (with AES-NI the overhead drops to ~2-3%)

# Measure CPU usage during encryption (in a separate terminal)
# sudo ip netns exec node-a top -b -n 3 -p $(pgrep iperf3) | grep iperf3
```

---

### Step 7 — Enable Cilium Transparent Encryption on a Kind Cluster

```bash
# Create a 3-node Kind cluster (1 control plane + 2 workers)
# Kind does not install a CNI by default when using --config
cat > /tmp/wg-lab/kind-cluster.yaml << 'EOF'
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: wg-lab
nodes:
  - role: control-plane
  - role: worker
  - role: worker
networking:
  disableDefaultCNI: true
  podSubnet: "10.244.0.0/16"
  serviceSubnet: "10.96.0.0/16"
EOF

kind create cluster --config /tmp/wg-lab/kind-cluster.yaml
# Expected:
# Creating cluster "wg-lab" ...
#  ✓ Ensuring node image (kindest/node:v1.30.0) ...
#  ✓ Preparing nodes ...
#  ✓ Writing configuration ...
#  ✓ Starting control-plane ...
#  ✓ Installing StorageClass ...
#  ✓ Joining worker nodes ...
# Set kubectl context to "kind-wg-lab"
# ...

kubectl get nodes
# Expected (NotReady because no CNI yet):
# NAME                    STATUS     ROLES           AGE   VERSION
# wg-lab-control-plane    NotReady   control-plane   30s   v1.30.0
# wg-lab-worker           NotReady   <none>          15s   v1.30.0
# wg-lab-worker2          NotReady   <none>          15s   v1.30.0

# Install Cilium with WireGuard transparent encryption
cilium install \
    --context kind-wg-lab \
    --set encryption.enabled=true \
    --set encryption.type=wireguard \
    --set encryption.wireguard.userspaceFallback=false \
    --set k8sServiceHost=wg-lab-control-plane \
    --set k8sServicePort=6443

cilium status --context kind-wg-lab --wait
# Expected (after ~90 seconds):
#     /¯¯\
#  /¯¯\__/¯¯\    Cilium:             OK
#  \__/¯¯\__/    Operator:           OK
# DaemonSet              cilium   Desired: 3, Ready: 3/3, Available: 3/3
```

---

### Step 8 — cilium encrypt status: Verify Encryption is Active on All Nodes

```bash
cilium encrypt status --context kind-wg-lab
# Expected:
# Encryption: WireGuard
# Decryption interface(s): eth0
# Keys in use: 1
# Max Seq. Number: 0x0/0xffffffff
# Errors: 0
#
# Node                     Public Key                                    Endpoint
# wg-lab-control-plane     +xNqo8TL9PGGxKO8EpKlFqOEJq/Z4F4qWfGfnpABC=  172.18.0.2
# wg-lab-worker            R4mQwEL9KHJiMNopQRsTUvWxYZabCdEfGhIjKlMnO=  172.18.0.3
# wg-lab-worker2           8kPlXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX=  172.18.0.4

# Verify WireGuard interface is present on each node
for node in wg-lab-control-plane wg-lab-worker wg-lab-worker2; do
    echo "=== ${node} ==="
    docker exec "${node}" wg show
done
# Expected for each node:
# === wg-lab-control-plane ===
# interface: cilium_wg0
#   public key: +xNqo8TL9PGGxKO8EpKlFqOEJq/Z4F4qWfGfnpABC=
#   private key: (hidden)
#   listening port: 51871
#
# peer: R4mQwEL9KHJiMNopQRsTUvWxYZabCdEfGhIjKlMnO=
#   endpoint: 172.18.0.3:51871
#   allowed ips: 10.244.1.0/24, 10.244.1.xx/32
#   latest handshake: 45 seconds ago
#   transfer: 12.4 KiB received, 8.2 KiB sent
# peer: 8kPlXXXX...
#   ...

kubectl get nodes
# Expected: all nodes now Ready
# NAME                    STATUS   ROLES           AGE     VERSION
# wg-lab-control-plane    Ready    control-plane   4m30s   v1.30.0
# wg-lab-worker           Ready    <none>          4m15s   v1.30.0
# wg-lab-worker2          Ready    <none>          4m15s   v1.30.0
```

---

### Step 9 — Deploy Two Pods, Confirm Traffic is WireGuard-Encapsulated

```bash
# Deploy two pods on different worker nodes
kubectl run pod-a --image=nicolaka/netshoot \
    --overrides='{"spec":{"nodeName":"wg-lab-worker"}}' \
    -- sleep infinity

kubectl run pod-b --image=nicolaka/netshoot \
    --overrides='{"spec":{"nodeName":"wg-lab-worker2"}}' \
    -- sleep infinity

kubectl wait pod/pod-a pod/pod-b --for=condition=Ready --timeout=60s
# Expected:
# pod/pod-a condition met
# pod/pod-b condition met

POD_B_IP=$(kubectl get pod pod-b -o jsonpath='{.status.podIP}')
echo "pod-b IP: ${POD_B_IP}"
# Expected: 10.244.2.x

# Ping between pods — should succeed (traffic encrypted in transit)
kubectl exec pod-a -- ping -c 3 "${POD_B_IP}"
# Expected:
# PING 10.244.2.x (10.244.2.x) 56(84) bytes of data.
# 64 bytes from 10.244.2.x: icmp_seq=1 ttl=63 time=0.891 ms
# 64 bytes from 10.244.2.x: icmp_seq=2 ttl=63 time=0.743 ms
# 64 bytes from 10.244.2.x: icmp_seq=3 ttl=63 time=0.812 ms

# On the worker node, capture traffic on the physical interface
# to confirm it is WireGuard UDP, not plaintext ICMP
docker exec wg-lab-worker \
    tcpdump -i eth0 -c 10 -nn udp port 51871 2>/dev/null &

kubectl exec pod-a -- ping -c 3 "${POD_B_IP}"

# Expected tcpdump output on wg-lab-worker:
# 14:22:31.001 IP 172.18.0.3.51871 > 172.18.0.4.51871: UDP, length 148
# 14:22:31.002 IP 172.18.0.4.51871 > 172.18.0.3.51871: UDP, length 148
# ...
# (UDP on port 51871 = WireGuard; no plaintext pod ICMP visible on eth0)
```

---

### Step 10 — Key Rotation: cilium encrypt rotate-key

```bash
# Initiate key rotation
cilium encrypt rotate-key --context kind-wg-lab
# Expected:
# Key rotation initiated successfully.

# Monitor the rotation — briefly two keys are active
watch -n 1 'cilium encrypt status --context kind-wg-lab'
# Expected during rotation:
# Encryption: WireGuard
# Keys in use: 2   <-- old and new key both active during rotation
# ...
# After ~10 seconds:
# Keys in use: 1   <-- rotation complete, only new key active

# Verify new handshake was established
for node in wg-lab-worker wg-lab-worker2; do
    echo "=== ${node} ==="
    docker exec "${node}" wg show | grep "latest handshake"
done
# Expected: recent timestamps (within the last 30 seconds)
# === wg-lab-worker ===
#   latest handshake: 8 seconds ago
# === wg-lab-worker2 ===
#   latest handshake: 7 seconds ago

# Confirm no traffic interruption during rotation
kubectl exec pod-a -- ping -c 30 -i 0.2 "${POD_B_IP}" | tail -3
# Expected: 0% packet loss over the rotation window
# 30 packets transmitted, 30 received, 0% packet loss, time 5800ms
# rtt min/avg/max/mdev = 0.712/0.834/1.103/0.089 ms
```

---

### Step 11 — Cleanup

```bash
# Remove the Kind cluster
kind delete cluster --name wg-lab
# Expected:
# Deleting cluster "wg-lab" ...

# Remove the network namespaces and veth pair
sudo ip netns del node-a
sudo ip netns del node-b
# (The veth pair is automatically deleted when its namespace is deleted)

# Verify cleanup
sudo ip netns list
# Expected: (empty or only pre-existing namespaces)

ip netns list | grep node
# Expected: (empty)

# Remove temporary files
rm -rf /tmp/wg-lab

echo "Cleanup complete."
```

---

## Summary

- AI cluster fabrics face qualitatively different threats than enterprise networks: RDMA bypasses all kernel security hooks, NIC firmware is an attack surface, and management plane credentials carry enormous blast radius.
- Zero trust re-anchors security on cryptographic workload identity (SPIFFE SVIDs, WireGuard key pairs) rather than IP addresses, which are not stable in dynamic environments.
- Cilium transparent encryption in WireGuard mode provides automatic node-to-node encryption with ~5-10% throughput overhead and zero application changes; IPsec mode uses the Linux xfrm framework for hardware-offload-capable deployments.
- WireGuard's minimal code surface (under 4,000 lines), integration into the Linux kernel since 5.6, and resistance to timing side-channels make it the preferred primitive for overlay encryption.
- strongSwan IKEv2 with certificate-based authentication satisfies compliance requirements that mandate standards-track IPsec and provides ESP transport mode for host-to-host scenarios where the WireGuard kernel module is unavailable.
- SPIFFE/SPIRE provides portable workload identity that is independent of the network topology, enabling mTLS between services without embedding IPs or hostnames in certificates.
- HashiCorp Vault's PKI secrets engine issues short-lived certificates for NIC management APIs, drastically reducing the blast radius of a credential compromise.
- Cilium `CiliumNetworkPolicy` with audit mode enables incremental policy rollout: observe violations before enforcing drops, with Hubble providing a visual policy map.

---

## References

- WireGuard: wireguard.com — Donenfeld, J.A., "WireGuard: Next Generation Kernel Network Tunnel", NDSS 2017
- Cilium transparent encryption: docs.cilium.io/en/stable/security/network/encryption-wireguard/
- strongSwan documentation: docs.strongswan.org/docs/5.9/index.html
- SPIFFE specification: github.com/spiffe/spiffe/blob/main/standards/SPIFFE.md
- SPIRE GitHub: github.com/spiffe/spire
- HashiCorp Vault PKI secrets engine: developer.hashicorp.com/vault/docs/secrets/pki
- cert-manager documentation: cert-manager.io/docs/
- Cilium CiliumNetworkPolicy: docs.cilium.io/en/stable/network/kubernetes/policy/
- Hubble observability: docs.cilium.io/en/stable/observability/hubble/
- NIST SP 800-207: Zero Trust Architecture (nist.gov/publications/zero-trust-architecture)


---

© 2025 jreuben1. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).