# Telco Build — Open CLEC Infrastructure Stack

A production-grade **CLEC (Competitive Local Exchange Carrier)** infrastructure built from the ground up on a local virtualization platform. This repository is an educational build log covering architecture decisions, network design, VM cloning methodology, security hardening, and telecom service stack configuration.

Everything here is vendor-neutral where possible. All IPs, hostnames, and environment-specific values are replaced with `<placeholders>`.

---

## Stack Overview

10 virtual machines across 5 functional roles, running Ubuntu 22.04 LTS:

| VM | Role | Networks |
|----|------|----------|
| kamailio-01 | SIP Proxy — primary | External + Voice + Sync |
| kamailio-02 | SIP Proxy — secondary | External + Voice + Sync |
| asterisk-01 | PBX — primary | External + Voice + Data |
| asterisk-02 | PBX — secondary | External + Voice + Data |
| asterisk-03 | PBX — tertiary | External + Voice + Data |
| a2billing-01 | Billing — primary | External + Data |
| a2billing-02 | Billing — secondary | External + Data |
| mariadb-01 | Database — primary | Data + Sync only |
| mariadb-02 | Database — secondary | Data + Sync only |
| monitor-01 | Security + Monitoring | External + Voice + Data |

Each VM: 8 GB RAM · 4 vCPU · 30 GB thin-provisioned disk

---

## Network Design

Five isolated virtual networks serve distinct traffic types:

| Network | Physical Uplink | MTU | Purpose |
|---------|-----------------|-----|---------|
| Management | yes | 1500 | Hypervisor management — not visible to VMs |
| External | yes | 1500 | SIP trunks, internet, web portals |
| Voice | **none** | **9000** | Kamailio ↔ Asterisk internal SIP/RTP |
| Data | **none** | **9000** | MariaDB ↔ A2Billing ↔ Asterisk |
| Sync | **none** | **9000** | DB replication + cluster heartbeat |

### Why No Uplink on Internal Networks?

Voice, Data, and Sync have no physical NIC attached. This is a hypervisor-level air gap — not a firewall rule, not an ACL. There is literally no wire connecting these networks to the outside world. MariaDB nodes are physically unreachable from the internet regardless of firewall configuration.

### Why MTU 9000 (Jumbo Frames)?

Standard MTU 1500 was designed for shared Ethernet in the 1980s. On an internal virtual switch with no physical limitations, jumbo frames deliver:

- 6× more payload per packet
- Fewer interrupts and context switches per MB transferred
- Measurable improvement on MariaDB replication streams, A2Billing CDR writes, and SIP/RTP inter-service flows

No routing or firewall changes needed — jumbo frames apply only within the isolated virtual switch.

---

## Call Flow

```
Internet / SIP Carriers
        │
        ▼
┌─────────────────────────────────┐
│  kamailio-01  │  kamailio-02    │  ← LCR: cheapest carrier per prefix
│  SIP Proxy + Load Balancer      │
└──────────────┬──────────────────┘
               │ Voice network (MTU 9000, isolated)
        ┌──────┴───────────────────────────┐
        │  asterisk-01 · asterisk-02 · 03  │
        │  PBX — load balanced via Kamailio│
        └──────┬───────────────────────────┘
               │ Data network (MTU 9000, isolated)
        ┌──────┴──────────────────┐
        │  A2Billing (billing)    │ ← real-time call auth
        │  MariaDB (CDR + subs)   │ ← call detail records
        └─────────────────────────┘
```

**Least Cost Routing (LCR):** On every outbound call, Kamailio evaluates the registered carrier rate tables in MariaDB and routes to the cheapest carrier for that destination prefix. If the carrier fails, the next cheapest is tried automatically. Rate tables are refreshed daily via automated CSV import.

**Real-time billing:** Every call triggers an AGI script in Asterisk that queries A2Billing before connecting. Insufficient balance = immediate rejection. Sub-second decision via the isolated Data network.

---

## Build Process

### 1. Golden Image

Install Ubuntu 22.04 Server on the first VM and configure it as a master template before cloning.

```bash
# Generate ED25519 key for all VM access
ssh-keygen -t ed25519 -C "clec-admin" -f ~/.ssh/clec-rsa -N ""

# Passwordless sudo for admin account
echo 'clec ALL=(ALL) NOPASSWD:ALL' | sudo tee /etc/sudoers.d/clec
chmod 440 /etc/sudoers.d/clec

# Hostname
sudo hostnamectl set-hostname kamailio-01
echo '127.0.1.1 kamailio-01' | sudo tee -a /etc/hosts

# Static IP via netplan
sudo tee /etc/netplan/00-installer-config.yaml << 'EOF'
network:
  version: 2
  ethernets:
    ens192:
      dhcp4: no
      addresses:
        - <IP>/24
      routes:
        - to: default
          via: <GATEWAY>
      nameservers:
        addresses: [8.8.8.8, 1.1.1.1]
EOF
sudo chmod 600 /etc/netplan/00-installer-config.yaml
sudo netplan apply

# Telecom service packages
sudo apt-get update && sudo apt-get install -y \
  kamailio kamailio-mysql-modules kamailio-tls-modules kamailio-utils-modules \
  asterisk asterisk-config asterisk-modules \
  mariadb-server mariadb-client \
  apache2 php php-mysql libapache2-mod-php \
  open-vm-tools openssh-server curl wget git \
  build-essential make net-tools iproute2 \
  fail2ban ufw logrotate htop iotop nload

# Timezone
sudo timedatectl set-timezone America/New_York

# 4 GB swap
sudo fallocate -l 4G /swapfile
sudo chmod 600 /swapfile && sudo mkswap /swapfile && sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab

sudo reboot
```

### 2. Virtual Network Setup

```bash
ssh root@<HYPERVISOR_IP>

# Voice — isolated, jumbo frames
vswitch-create VOICE && vswitch-portgroup VOICE VOICE && vswitch-mtu VOICE 9000

# Data — isolated, jumbo frames
vswitch-create DATA && vswitch-portgroup DATA DATA && vswitch-mtu DATA 9000

# Sync — isolated, jumbo frames (cluster heartbeat + DB replication)
vswitch-create SYNC && vswitch-portgroup SYNC SYNC && vswitch-mtu SYNC 9000
```

*Replace `vswitch-create` / `vswitch-portgroup` / `vswitch-mtu` with the appropriate commands for your hypervisor.*  
Example for VMware (esxcli):

```bash
esxcli network vswitch standard add --vswitch-name=SYNC
esxcli network vswitch standard portgroup add --vswitch-name=SYNC --portgroup-name=SYNC
esxcli network vswitch standard set --vswitch-name=SYNC --mtu=9000
```

### 3. Clone 9 VMs from Golden Image

On VMware hypervisors, `vmkfstools` performs a storage-layer thin clone. On fast NVMe storage, 9 × 30 GB VMs complete in under 2 minutes.

```bash
SRC="<DATASTORE>/kamailio-01/kamailio-01.vmdk"

for VM in kamailio-02 asterisk-01 asterisk-02 asterisk-03 \
           a2billing-01 a2billing-02 mariadb-01 mariadb-02 monitor-01; do
  vmkfstools -i "$SRC" "<DATASTORE>/$VM/$VM.vmdk" -d thin
  echo "Cloned: $VM"
done
```

**How thin cloning works:**  
Unlike a file copy, a metadata-level thin clone rewrites only the VMDK descriptor. Unchanged disk blocks are referenced from the source (copy-on-write). A 30 GB VM clones in seconds because no actual 30 GB of data moves at clone time — pages are materialized on first write.

### 4. NIC Assignment per Role

Assign each VM the correct virtual networks by appending to its VM configuration file:

```bash
# Kamailio: External + Voice + Sync
append_nics kamailio-01 VOICE SYNC
append_nics kamailio-02 VOICE SYNC

# Asterisk: External + Voice + Data
append_nics asterisk-01 VOICE DATA
append_nics asterisk-02 VOICE DATA
append_nics asterisk-03 VOICE DATA

# A2Billing: External + Data
append_nics a2billing-01 DATA
append_nics a2billing-02 DATA

# MariaDB: Data + Sync only (no external NIC)
change_primary_network mariadb-01 DATA
append_nics mariadb-01 SYNC
change_primary_network mariadb-02 DATA
append_nics mariadb-02 SYNC

# Monitor: External + Voice + Data
append_nics monitor-01 VOICE DATA
```

VMware VMX syntax example:

```
ethernet1.present = "TRUE"
ethernet1.virtualDev = "vmxnet3"
ethernet1.networkName = "VOICE"
ethernet1.addressType = "generated"
```

### 5. Individual VM Configuration

Each clone boots with the golden image IP. Configure one at a time:

```bash
configure_vm() {
  local VMID=$1 NAME=$2 NEWIP=$3

  hypervisor-power-on $VMID

  until ssh -i ~/.ssh/clec-rsa -o ConnectTimeout=5 \
    -o StrictHostKeyChecking=no clec@<GOLD_IP> "echo ok" 2>/dev/null; do
    sleep 2
  done

  ssh -i ~/.ssh/clec-rsa clec@<GOLD_IP> << CMD
    # Reset machine-id (clones inherit golden image ID)
    echo '' | sudo tee /etc/machine-id > /dev/null
    sudo systemd-machine-id-setup

    # Hostname
    sudo hostnamectl set-hostname $NAME
    sudo sed -i '/^127.0.1.1/d' /etc/hosts
    echo '127.0.1.1 $NAME' | sudo tee -a /etc/hosts

    # New IP in netplan
    sudo sed -i "s|<GOLD_IP>/24|${NEWIP}/24|g" /etc/netplan/00-installer-config.yaml
    sudo chmod 600 /etc/netplan/00-installer-config.yaml
    sudo rm -f /var/lib/dhcp/dhclient* 2>/dev/null
    echo "configured -> $NEWIP"
    sudo poweroff
CMD
}

configure_vm <ID> kamailio-02  <IP>
configure_vm <ID> asterisk-01  <IP>
configure_vm <ID> asterisk-02  <IP>
configure_vm <ID> asterisk-03  <IP>
configure_vm <ID> a2billing-01 <IP>
configure_vm <ID> a2billing-02 <IP>
configure_vm <ID> mariadb-01   <IP>
configure_vm <ID> mariadb-02   <IP>
configure_vm <ID> monitor-01   <IP>
```

**Why reset machine-id?**  
Systemd uses `/etc/machine-id` as the DHCP client identifier and for D-Bus addressing. All clones inherit the golden image ID, causing ID collisions that affect network identity and log rotation. `systemd-machine-id-setup` generates a new unique ID from hardware entropy.

---

## Phase 2 — Security & Monitoring

### OSSEC HIDS

OSSEC runs on a server/agent model:

- **monitor-01** is the OSSEC server — receives events from all 9 agents, correlates alerts, sends notifications
- All 9 other VMs run OSSEC agents — report file integrity events, log anomalies, brute-force detections

Monitored directories: `/etc`, `/bin`, `/sbin`, `/usr/bin`, `/usr/sbin`  
Active response: auto-block attacking source IPs via local firewall rules within seconds of threshold breach  
Alerts: email on every Level 7+ security event

### Zabbix Monitoring

- **ISP connectivity** — continuous ping to external resolver; alert on sustained packet loss
- **SIP health** — OPTIONS ping to each Kamailio node; alert if no 200 OK within threshold
- **Database health** — MariaDB connection check on each node
- **Resource thresholds** — CPU, RAM, disk alerts before service impact
- **Agent2** on all VMs — active checks with encrypted transport

---

## Codec Reference

| Codec | Standard | Where Used |
|-------|----------|------------|
| G.711 μ-law (ulaw) | North American PSTN | All carrier termination in NANP |
| G.711 A-law (alaw) | European PSTN | International termination |
| G.722 | HD Voice (16 kHz) | Internal LAN phones |

Carrier negotiation in North America defaults to G.711 μ-law. The internal Voice network carries SIP/RTP at MTU 9000, so no transcoding occurs on internal hops — codec passes end-to-end.

---

## Phases Roadmap

| Phase | Status | Scope |
|-------|--------|-------|
| 1 — Hypervisor & VMs | ✅ Complete | vSwitches, golden image, 10 VMs cloned and configured |
| 2 — Security & Monitoring | 🔄 In progress | OSSEC server + agents, Zabbix, fail2ban |
| 3 — Database | Pending | MariaDB replication via Sync network |
| 4 — SIP Proxy | Pending | Kamailio config, LCR, dispatcher, TLS, clustering |
| 5 — PBX | Pending | Asterisk PJSIP, dialplan, CDR, call recording |
| 6 — Billing | Pending | A2Billing engine, rate tables, customer portal |
| 7 — Cloud Integration | Pending | WireGuard tunnel, SIP federation, DNS failover |
| 8 — Phones | Pending | VoIP phone VLANs, QoS (DSCP/802.1p), SIP registration |

---

*Repository updated as each phase completes. All values are sanitized — placeholders replace all environment-specific data.*
