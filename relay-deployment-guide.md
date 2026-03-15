# Deploying strfry Nostr Relay on Azure

## What This Is and Why You're Doing It

**NostrWolfe** is an agent commerce protocol that lets AI agents discover, negotiate with, and pay each other over the Nostr network, settled via Bitcoin Lightning payments (L402).

The protocol uses 4 custom Nostr event kinds:
- **Kind 38400** — Agent Capability (an agent advertising a service or product it offers)
- **Kind 38401** — Service Request (an agent asking for a service)
- **Kind 38402** — Service Agreement (a bilateral contract between two agents)
- **Kind 38403** — Attestation (a review/rating of an agent after completing work)

**A Nostr relay is a WebSocket server that stores and routes these events.** Think of it as a message broker — agents publish events to it and query it to discover other agents. Without this relay running, agents have nowhere to publish or discover capabilities.

**strfry** is a high-performance open-source Nostr relay written in C++. It's the most widely deployed relay implementation in the Nostr ecosystem. It stores events in LMDB (an embedded database), accepts WebSocket connections, and supports filtering by event kind, author, tags, etc.

**Your task:** Deploy strfry as a Docker container on Azure so it's accessible at `wss://agents.lightningenable.com`. This is the dedicated relay for agent-to-agent commerce events. Once running, the Python SDK (`le-agent-sdk`), TypeScript SDK, .NET SDK, MCP tools, and the NostrWolfe iOS app will all connect to this relay automatically.

**The existing Lightning Enable API** is already deployed at `api.lightningenable.com` on Azure App Service in the `appsvc_windows_eastus_basic` resource group. This relay should be deployed in the same resource group for consistency.

**Relationship to LE API:** The LE API has its own agent endpoints (`/api/agents/*`) that provide a centralized REST convenience layer. The Nostr relay provides decentralized, permissionless discovery. Both work together — the MCP tools try the API first and fall back to the relay. Agents can use either or both. The relay is the primary backbone; the API is the convenience layer.

---

## Overview

Deploy the NostrWolfe agent relay at `wss://agents.lightningenable.com` using [strfry](https://github.com/hoytech/strfry), a high-performance C++ Nostr relay backed by LMDB.

**Target:** A relay purpose-built for Agent Service Agreement (ASA) events, accepting agent-specific event kinds and encrypted DMs for negotiation.

**Docker image:** Built from `ghcr.io/hoytech/strfry` or the repo Dockerfile (Alpine-based, exposes port 7777).

---

## Option 1: Azure Container Instance (Recommended -- ~$5/mo)

Cheapest option. Single container, persistent storage via Azure File Share.

### Prerequisites

- Azure CLI (`az`) installed and authenticated (`az login`)
- Existing resource group: `appsvc_windows_eastus_basic`
- DNS access for `agents.lightningenable.com`
- **DNS provider:** Check where `lightningenable.com` DNS is managed. If already on Cloudflare, use Option 1 directly. If DNS is on Azure DNS or another provider, either migrate to Cloudflare (free plan works) or use Option 3 (VM + Caddy) which handles TLS via Let's Encrypt without needing Cloudflare. To check: `dig NS lightningenable.com` — if nameservers are `*.ns.cloudflare.com`, it's already on Cloudflare.

### Step 1: Create a Storage Account and File Share

strfry uses LMDB, which needs persistent storage across container restarts.

```bash
# Create storage account (name must be globally unique, lowercase, no hyphens)
az storage account create \
  --name strfryrelaystorage \
  --resource-group appsvc_windows_eastus_basic \
  --location eastus \
  --sku Standard_LRS

# Get the storage key
STORAGE_KEY=$(az storage account keys list \
  --account-name strfryrelaystorage \
  --resource-group appsvc_windows_eastus_basic \
  --query "[0].value" -o tsv)

# Create file shares for DB and config
az storage share create \
  --name strfry-db \
  --account-name strfryrelaystorage \
  --account-key "$STORAGE_KEY"

az storage share create \
  --name strfry-config \
  --account-name strfryrelaystorage \
  --account-key "$STORAGE_KEY"
```

### Step 2: Upload strfry.conf

Create the config file locally, then upload it.

```bash
cat > /tmp/strfry.conf << 'CONF'
##
## strfry configuration for NostrWolfe Agent Relay
##

# Database settings
db = "./strfry-db/"

dbParams {
    # Maximum number of threads/open transactions
    maxreaders = 64

    # Size of mmap() -- 1GB is plenty for an agent relay
    mapsize = 1073741824
}

events {
    # Maximum size of normalised JSON, in bytes (128KB)
    maxEventSize = 131072

    # Events newer than this many seconds are rejected (0 = no limit)
    rejectEventsNewerThanSeconds = 900

    # Events older than this many seconds are rejected (~1 year)
    rejectEventsOlderThanSeconds = 31536000

    # Ephemeral events older than this are rejected (5 minutes)
    rejectEphemeralEventsOlderThanSeconds = 300

    # Ephemeral events are not stored in the DB
    ephemeralEventsLifetimeSeconds = 300

    # Maximum number of tags per event
    maxNumTags = 2000

    # Maximum size for tag values, in bytes
    maxTagValSize = 1024
}

relay {
    # Bind to all interfaces inside the container
    bind = "0.0.0.0"

    # Port 80 so Cloudflare can proxy (Cloudflare only connects to standard ports)
    port = 80

    # No TLS at container level -- terminated by Cloudflare
    nofiles = 1000
    maxWebsocketPayloadSize = 131072

    autoPingSeconds = 55

    enableTcpKeepalive = false

    queryTimesliceBudgetMicroseconds = 10000
    maxFilterLimit = 500
    maxSubsPerConnection = 20

    writePolicy {
        # Path to a write-policy plugin (see Step 3 for the plugin script)
        plugin = "/app/strfry-write-policy.sh"
    }

    compression {
        enabled = true
        slidingWindow = true
    }

    logging {
        dumpInAll = false
        dumpInEvents = false
        dumpInReqs = false
        dbScanPerf = false
    }

    numThreads {
        ingester = 1
        reqWorker = 1
        reqMonitor = 1
        negentropy = 1
    }

    negentropy {
        enabled = true
        maxSyncEvents = 1000000
    }

    info {
        name = "NostrWolfe Agent Relay"
        description = "Relay for Agent Service Agreements (ASA) on Nostr. Accepts agent profiles, ASA events, and encrypted DMs for negotiation."
        pubkey = ""
        contact = "support@lightningenable.com"
    }
}
CONF
```

Upload to Azure File Share:

```bash
az storage file upload \
  --share-name strfry-config \
  --source /tmp/strfry.conf \
  --path strfry.conf \
  --account-name strfryrelaystorage \
  --account-key "$STORAGE_KEY"
```

### Step 3: Create and Upload the Write-Policy Plugin

This plugin restricts the relay to only accept agent-relevant event kinds.

```bash
cat > /tmp/strfry-write-policy.sh << 'PLUGIN'
#!/bin/sh
# strfry write-policy plugin
# Accepts only agent-relevant event kinds:
#   0     = metadata (agent profiles)
#   4     = encrypted DM (legacy, NIP-04)
#   44    = encrypted DM (NIP-44 gift wrap)
#   1059  = gift wrap (NIP-59, used by NIP-44)
#   10002 = relay list metadata
#   30078 = application-specific data
#   38400 = ASA capability advertisement
#   38401 = ASA service request
#   38402 = ASA service agreement
#   38403 = ASA service result/receipt

while read -r line; do
    KIND=$(echo "$line" | sed -n 's/.*"kind":\([0-9]*\).*/\1/p')

    case "$KIND" in
        0|4|44|1059|10002|30078|38400|38401|38402|38403)
            # Accept the event
            ID=$(echo "$line" | sed -n 's/.*"id":"\([^"]*\)".*/\1/p')
            echo "{\"id\":\"${ID}\",\"action\":\"accept\"}"
            ;;
        *)
            # Reject with reason
            ID=$(echo "$line" | sed -n 's/.*"id":"\([^"]*\)".*/\1/p')
            echo "{\"id\":\"${ID}\",\"action\":\"reject\",\"msg\":\"kind ${KIND} not accepted on this relay\"}"
            ;;
    esac
done
PLUGIN
```

Upload it:

```bash
az storage file upload \
  --share-name strfry-config \
  --source /tmp/strfry-write-policy.sh \
  --path strfry-write-policy.sh \
  --account-name strfryrelaystorage \
  --account-key "$STORAGE_KEY"
```

### Step 4: Create the Container Instance

```bash
az container create \
  --resource-group appsvc_windows_eastus_basic \
  --name strfry-agent-relay \
  --image ghcr.io/hoytech/strfry \
  --os-type Linux \
  --cpu 1 \
  --memory 1 \
  --ports 80 \
  --ip-address Public \
  --dns-name-label nostrwolfe-relay \
  --command-line "/app/strfry relay" \
  --azure-file-volume-account-name strfryrelaystorage \
  --azure-file-volume-account-key "$STORAGE_KEY" \
  --azure-file-volume-share-name strfry-db \
  --azure-file-volume-mount-path /app/strfry-db \
  --restart-policy Always
```

> **Note:** ACI supports only one `--azure-file-volume-*` set via CLI. To mount both config and db, use an ARM template or YAML deployment file (see Step 4b).

### Step 4b: Deploy with YAML (Recommended -- Multiple Volume Mounts)

```bash
cat > /tmp/strfry-aci.yaml << YAML
apiVersion: '2021-10-01'
location: eastus
name: strfry-agent-relay
properties:
  containers:
  - name: strfry
    properties:
      image: ghcr.io/hoytech/strfry
      command:
      - /app/strfry
      - --config=/app/strfry.conf
      - relay
      resources:
        requests:
          cpu: 1.0
          memoryInGb: 1.0
      ports:
      - port: 80
        protocol: TCP
      volumeMounts:
      - name: strfry-db-volume
        mountPath: /app/strfry-db
      - name: strfry-config-volume
        mountPath: /app/strfry.conf
        readOnly: true
        subPath: strfry.conf
      - name: strfry-config-volume
        mountPath: /app/strfry-write-policy.sh
        readOnly: true
        subPath: strfry-write-policy.sh
  osType: Linux
  ipAddress:
    type: Public
    dnsNameLabel: nostrwolfe-relay
    ports:
    - port: 7777
      protocol: TCP
  restartPolicy: Always
  volumes:
  - name: strfry-db-volume
    azureFile:
      shareName: strfry-db
      storageAccountName: strfryrelaystorage
      storageAccountKey: ${STORAGE_KEY}
  - name: strfry-config-volume
    azureFile:
      shareName: strfry-config
      storageAccountName: strfryrelaystorage
      storageAccountKey: ${STORAGE_KEY}
type: Microsoft.ContainerInstance/containerGroups
YAML

# Substitute the storage key into the YAML
sed -i.bak "s|\${STORAGE_KEY}|${STORAGE_KEY}|g" /tmp/strfry-aci.yaml

az container create \
  --resource-group appsvc_windows_eastus_basic \
  --file /tmp/strfry-aci.yaml
```

### Step 5: Verify the Container is Running

```bash
# Check container status
az container show \
  --resource-group appsvc_windows_eastus_basic \
  --name strfry-agent-relay \
  --query "{Status:instanceView.state, IP:ipAddress.ip, FQDN:ipAddress.fqdn, Ports:ipAddress.ports}" \
  -o table

# View logs
az container logs \
  --resource-group appsvc_windows_eastus_basic \
  --name strfry-agent-relay
```

The container's public FQDN will be: `nostrwolfe-relay.eastus.azurecontainer.io`

### Step 6: Set Up TLS with Cloudflare (Recommended)

ACI does not provide built-in TLS. The simplest approach is Cloudflare:

1. **Add DNS record in Cloudflare:**
   - Type: `CNAME`
   - Name: `agents`
   - Target: `nostrwolfe-relay.eastus.azurecontainer.io`
   - Proxy: **ON** (orange cloud)

2. **Configure Cloudflare SSL/TLS:**
   - SSL mode: `Flexible` (Cloudflare terminates TLS, connects to origin on port 80 via plain HTTP/WS)
   - Minimum TLS version: 1.2

3. **Configure Cloudflare for WebSockets:**
   - Go to Network tab -> Enable WebSockets (on by default for free and paid plans)

> **How it works:** Clients connect to `wss://agents.lightningenable.com` (port 443). Cloudflare terminates TLS and forwards to the ACI container on port 80 via plain WebSocket. No origin rules needed since port 80 is a standard Cloudflare-supported port.

**Alternative: Cloudflare Tunnel (No Public IP Needed)**

If you want to avoid exposing a public IP entirely:

```bash
# Install cloudflared on the container or run as a sidecar
# Create a tunnel
cloudflared tunnel create nostrwolfe-relay

# Configure the tunnel to point to localhost:7777
cloudflared tunnel route dns nostrwolfe-relay agents.lightningenable.com

# Run the tunnel
cloudflared tunnel run nostrwolfe-relay
```

### Step 7: Test the Relay

```bash
# Install wscat if not already installed
npm install -g wscat

# Test direct (no TLS)
wscat -c ws://nostrwolfe-relay.eastus.azurecontainer.io

# Test via Cloudflare (with TLS)
wscat -c wss://agents.lightningenable.com

# Once connected, send a REQ to verify:
# > ["REQ", "test-1", {"kinds": [38400], "limit": 5}]
# Expected response: ["EOSE", "test-1"]  (empty since no events yet)

# Test publishing an event (NIP-01 format):
# > ["EVENT", {"id":"...","pubkey":"...","created_at":...,"kind":38400,"tags":[],"content":"test","sig":"..."}]
```

---

## Option 2: Azure App Service with Linux Container (~$13-55/mo)

More expensive but includes managed SSL, custom domains, auto-restart, and scaling.

### Step 1: Create an App Service Plan

```bash
az appservice plan create \
  --name strfry-relay-plan \
  --resource-group appsvc_windows_eastus_basic \
  --location eastus \
  --is-linux \
  --sku B1
```

### Step 2: Create the Web App with Docker Container

```bash
az webapp create \
  --resource-group appsvc_windows_eastus_basic \
  --plan strfry-relay-plan \
  --name nostrwolfe-relay \
  --deployment-container-image-name ghcr.io/hoytech/strfry
```

### Step 3: Configure WebSocket Support and Port

```bash
# Enable WebSockets
az webapp config set \
  --resource-group appsvc_windows_eastus_basic \
  --name nostrwolfe-relay \
  --web-sockets-enabled true

# Tell App Service the container listens on 7777
az webapp config appsettings set \
  --resource-group appsvc_windows_eastus_basic \
  --name nostrwolfe-relay \
  --settings WEBSITES_PORT=7777

# Set the strfry config environment variable
az webapp config appsettings set \
  --resource-group appsvc_windows_eastus_basic \
  --name nostrwolfe-relay \
  --settings STRFRY_CONFIG=/etc/strfry.conf

# Set startup command
az webapp config set \
  --resource-group appsvc_windows_eastus_basic \
  --name nostrwolfe-relay \
  --startup-file "/app/strfry relay"
```

### Step 4: Mount Persistent Storage

```bash
# Create an Azure Storage mount for the LMDB database
az webapp config storage-account add \
  --resource-group appsvc_windows_eastus_basic \
  --name nostrwolfe-relay \
  --custom-id strfry-db \
  --storage-type AzureFiles \
  --share-name strfry-db \
  --account-name strfryrelaystorage \
  --access-key "$STORAGE_KEY" \
  --mount-path /app/strfry-db

az webapp config storage-account add \
  --resource-group appsvc_windows_eastus_basic \
  --name nostrwolfe-relay \
  --custom-id strfry-config \
  --storage-type AzureFiles \
  --share-name strfry-config \
  --account-name strfryrelaystorage \
  --access-key "$STORAGE_KEY" \
  --mount-path /etc/strfry-config
```

> **Note:** App Service mounts Azure Files at the directory level. You may need to adjust the strfry.conf `STRFRY_CONFIG` env var to `/etc/strfry-config/strfry.conf`.

### Step 5: Set Custom Domain and SSL

```bash
# Add custom domain
az webapp config hostname add \
  --resource-group appsvc_windows_eastus_basic \
  --webapp-name nostrwolfe-relay \
  --hostname agents.lightningenable.com

# Create a managed SSL certificate (free with App Service)
az webapp config ssl create \
  --resource-group appsvc_windows_eastus_basic \
  --name nostrwolfe-relay \
  --hostname agents.lightningenable.com

# Get the certificate thumbprint
THUMBPRINT=$(az webapp config ssl list \
  --resource-group appsvc_windows_eastus_basic \
  --query "[?subjectName=='agents.lightningenable.com'].thumbprint" -o tsv)

# Bind the SSL certificate
az webapp config ssl bind \
  --resource-group appsvc_windows_eastus_basic \
  --name nostrwolfe-relay \
  --certificate-thumbprint "$THUMBPRINT" \
  --ssl-type SNI
```

**DNS:** Create a CNAME record for `agents.lightningenable.com` pointing to `nostrwolfe-relay.azurewebsites.net`.

### Step 6: Restart and Test

```bash
az webapp restart \
  --resource-group appsvc_windows_eastus_basic \
  --name nostrwolfe-relay

# Check logs
az webapp log tail \
  --resource-group appsvc_windows_eastus_basic \
  --name nostrwolfe-relay

# Test
wscat -c wss://agents.lightningenable.com
```

---

## Option 3: Azure VM with Docker (~$7/mo)

Full control. B1s VM is the cheapest compute option.

### Step 1: Create the VM

```bash
az vm create \
  --resource-group appsvc_windows_eastus_basic \
  --name strfry-relay-vm \
  --image Ubuntu2404 \
  --size Standard_B1s \
  --admin-username azureuser \
  --generate-ssh-keys \
  --public-ip-sku Standard \
  --nsg-rule SSH
```

### Step 2: Open Port 7777 (and 443 if terminating TLS locally)

```bash
az vm open-port \
  --resource-group appsvc_windows_eastus_basic \
  --name strfry-relay-vm \
  --port 7777 \
  --priority 1010

az vm open-port \
  --resource-group appsvc_windows_eastus_basic \
  --name strfry-relay-vm \
  --port 443 \
  --priority 1020
```

### Step 3: SSH In and Install Docker + Deploy

```bash
# Get the VM's public IP
VM_IP=$(az vm show \
  --resource-group appsvc_windows_eastus_basic \
  --name strfry-relay-vm \
  --show-details \
  --query publicIps -o tsv)

# SSH into the VM
ssh azureuser@$VM_IP
```

Once on the VM:

```bash
# Install Docker
sudo apt-get update
sudo apt-get install -y docker.io docker-compose-v2
sudo systemctl enable docker
sudo systemctl start docker
sudo usermod -aG docker azureuser

# Create directories
sudo mkdir -p /opt/strfry/strfry-db
sudo mkdir -p /opt/strfry/config

# Write strfry.conf (use the same config from Step 2 of Option 1)
sudo tee /opt/strfry/config/strfry.conf << 'CONF'
db = "./strfry-db/"
dbParams {
    maxreaders = 64
    mapsize = 1073741824
}
events {
    maxEventSize = 131072
    rejectEventsNewerThanSeconds = 900
    rejectEventsOlderThanSeconds = 31536000
    rejectEphemeralEventsOlderThanSeconds = 300
    ephemeralEventsLifetimeSeconds = 300
    maxNumTags = 2000
    maxTagValSize = 1024
}
relay {
    bind = "0.0.0.0"
    port = 7777
    nofiles = 1000
    maxWebsocketPayloadSize = 131072
    autoPingSeconds = 55
    enableTcpKeepalive = false
    queryTimesliceBudgetMicroseconds = 10000
    maxFilterLimit = 500
    maxSubsPerConnection = 20
    writePolicy {
        plugin = ""
    }
    compression {
        enabled = true
        slidingWindow = true
    }
    logging {
        dumpInAll = false
        dumpInEvents = false
        dumpInReqs = false
        dbScanPerf = false
    }
    numThreads {
        ingester = 1
        reqWorker = 1
        reqMonitor = 1
        negentropy = 1
    }
    negentropy {
        enabled = true
        maxSyncEvents = 1000000
    }
    info {
        name = "NostrWolfe Agent Relay"
        description = "Relay for Agent Service Agreements (ASA) on Nostr"
        pubkey = ""
        contact = "support@lightningenable.com"
    }
}
CONF

# Write docker-compose.yaml
sudo tee /opt/strfry/docker-compose.yaml << 'YAML'
services:
  strfry:
    image: ghcr.io/hoytech/strfry
    container_name: strfry-relay
    restart: always
    ports:
      - "7777:7777"
    volumes:
      - /opt/strfry/config/strfry.conf:/etc/strfry.conf:ro
      - /opt/strfry/strfry-db:/app/strfry-db
    environment:
      - STRFRY_CONFIG=/etc/strfry.conf
    command: ["relay"]
YAML

# Start the relay
cd /opt/strfry
sudo docker compose up -d

# Verify it's running
sudo docker compose logs -f
```

### Step 4: Set Up TLS with Caddy (Auto HTTPS)

```bash
# Install Caddy as a reverse proxy
sudo apt-get install -y debian-keyring debian-archive-keyring apt-transport-https curl
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | sudo tee /etc/apt/sources.list.d/caddy-stable.list
sudo apt-get update
sudo apt-get install caddy

# Configure Caddy
sudo tee /etc/caddy/Caddyfile << 'CADDY'
agents.lightningenable.com {
    reverse_proxy localhost:7777
}
CADDY

# Restart Caddy (it auto-provisions Let's Encrypt TLS)
sudo systemctl restart caddy
```

**DNS:** Create an A record for `agents.lightningenable.com` pointing to the VM's public IP.

---

## Post-Deployment Testing

### Quick Test with wscat

```bash
npm install -g wscat

# Connect to the relay
wscat -c wss://agents.lightningenable.com

# Send NIP-11 info request (HTTP, not WS)
curl -H "Accept: application/nostr+json" https://agents.lightningenable.com

# Expected NIP-11 response:
# {"name":"NostrWolfe Agent Relay","description":"Relay for Agent Service Agreements...","supported_nips":[1,2,...]}
```

### Test with Python SDK

```python
from le_agent_sdk import AgentManager

manager = AgentManager(relay_urls=["wss://agents.lightningenable.com"])
caps = await manager.discover()
print(f"Discovered {len(caps)} agent capabilities")
```

### Publish a Test Event

```bash
# Using nostril (go-based Nostr CLI) or nak
# Install nak: go install github.com/fiatjaf/nak@latest

# Publish a test kind-38400 event
nak event \
  --kind 38400 \
  --content '{"name":"test-agent","capabilities":["echo"]}' \
  -t d=test-agent-001 \
  wss://agents.lightningenable.com
```

---

## Monitoring

### Azure Container Instance

```bash
# View real-time logs
az container logs \
  --resource-group appsvc_windows_eastus_basic \
  --name strfry-agent-relay \
  --follow

# Check container metrics (CPU, memory, network)
az monitor metrics list \
  --resource /subscriptions/$(az account show --query id -o tsv)/resourceGroups/appsvc_windows_eastus_basic/providers/Microsoft.ContainerInstance/containerGroups/strfry-agent-relay \
  --metric CPUUsage MemoryUsage NetworkBytesReceivedPerSecond \
  --interval PT1H
```

### Set Up Alerts

```bash
# Alert on container restart
az monitor metrics alert create \
  --name strfry-restart-alert \
  --resource-group appsvc_windows_eastus_basic \
  --scopes /subscriptions/$(az account show --query id -o tsv)/resourceGroups/appsvc_windows_eastus_basic/providers/Microsoft.ContainerInstance/containerGroups/strfry-agent-relay \
  --condition "avg RestartCount > 2" \
  --window-size PT5M \
  --evaluation-frequency PT1M \
  --description "strfry container restarting frequently"
```

### Simple Uptime Check (External)

```bash
# Add a cron job or use Azure Logic App to ping the relay every 5 minutes
curl -s -o /dev/null -w "%{http_code}" \
  -H "Accept: application/nostr+json" \
  https://agents.lightningenable.com
# Should return 200
```

---

## DNS Configuration Summary

| Option | Record Type | Name | Value |
|--------|-------------|------|-------|
| ACI + Cloudflare | CNAME | agents | nostrwolfe-relay.eastus.azurecontainer.io |
| App Service | CNAME | agents | nostrwolfe-relay.azurewebsites.net |
| VM + Caddy | A | agents | (VM public IP) |

All DNS records are under the `lightningenable.com` zone.

---

## Cost Comparison

| Option | Monthly Cost | TLS | Auto-Restart | Scaling |
|--------|-------------|-----|--------------|---------|
| ACI (1 CPU, 1GB) | ~$5 | Via Cloudflare | Yes | Manual |
| App Service (B1) | ~$13 | Managed (free) | Yes | Manual/Auto |
| VM (B1s) + Caddy | ~$7 | Let's Encrypt | Via systemd | Manual |

---

## Recommendation

**Start with Option 1 (ACI + Cloudflare)** for the lowest cost and simplest deployment. If WebSocket reliability through Cloudflare becomes an issue, switch to **Option 3 (VM + Caddy)** for direct TLS termination at ~$7/mo.
