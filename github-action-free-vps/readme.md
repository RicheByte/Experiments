
## Final GitHub Actions Code

This is the final version of the workflow script. You will save this as a file in your repository in the steps below.

YAML

```
name: Infinite SSH and VPN via Cloudflare Tunnel
on:
  workflow_dispatch:
  schedule:
    - cron: "0 */5 * * *"   # restart every 5 hours
jobs:
  ssh:
    runs-on: ubuntu-latest
    timeout-minutes: 360
    steps:
      - name: Install dependencies
        run: |
          sudo apt-get update -y
          sudo apt-get install -y openssh-server autossh
          curl -fsSL https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64 \
            -o /usr/local/bin/cloudflared
          chmod +x /usr/local/bin/cloudflared

      - name: Start SSH server
        run: |
          sudo mkdir -p /var/run/sshd
          echo 'runner:runner' | sudo chpasswd
          sudo sed -i 's/#PasswordAuthentication yes/PasswordAuthentication yes/' /etc/ssh/sshd_config
          sudo sed -i 's/PasswordAuthentication no/PasswordAuthentication yes/' /etc/ssh/sshd_config
          sudo service ssh restart

      - name: Start Cloudflare Tunnel
        run: |
          cloudflared tunnel --url ssh://localhost:22 > cf.log 2>&1 &
          sleep 5
          echo "====================================="
          echo "SSH connection details will appear below:"
          grep -o 'trycloudflare.com[^ ]*' cf.log | head -n 1
          echo "====================================="

      - name: Install and Configure 3x-ui VPN
        env:
          VPN_CONFIG: ${{ secrets.VPN_CONFIG }}
        run: |
          bash <(curl -Ls https://raw.githubusercontent.com/mhsanaei/3x-ui/master/install.sh)
          echo "$VPN_CONFIG" | sudo tee /etc/x-ui/config.json > /dev/null
          sudo x-ui restart
          echo "VPN panel configured successfully."

      - name: Keep alive
        run: sleep 17400  # ~4h 50m
```

---

## Step-by-Step Installation Guide

Follow these four steps exactly to get your VPN running.

### Step 1: Prepare Your VPN Configuration

Your original configuration will fail because it requires SSL certificates that don't exist in the GitHub environment.

**Copy the entire corrected code block below.** This version has TLS security disabled so that it will run correctly.

JSON

```
{
  "api": {
    "services": [
      "HandlerService",
      "LoggerService",
      "StatsService"
    ],
    "tag": "api"
  },
  "burstObservatory": null,
  "dns": null,
  "fakedns": null,
  "inbounds": [
    {
      "allocate": null,
      "listen": "127.0.0.1",
      "port": 62789,
      "protocol": "dokodemo-door",
      "settings": {
        "address": "127.0.0.1"
      },
      "sniffing": null,
      "streamSettings": null,
      "tag": "api"
    },
    {
      "allocate": {
        "concurrency": 3,
        "refresh": 5,
        "strategy": "always"
      },
      "listen": null,
      "port": 443,
      "protocol": "vless",
      "settings": {
        "clients": [
          {
            "email": "9re4083n",
            "flow": "",
            "id": "709d217a-663c-4073-8b9b-7ef1f47eedb4"
          },
          {
            "email": "pc",
            "flow": "",
            "id": "49bf8a41-90eb-4c9d-85d2-7cdd0658cb4f"
          },
          {
            "email": "iphone",
            "flow": "",
            "id": "0b1469c1-f9ba-493c-b628-95a27e4531c2"
          },
          {
            "email": "Hp Elitebook",
            "flow": "",
            "id": "22c834de-3406-4f3f-8f84-99a288050628"
          },
          {
            "email": "Kanchuka",
            "flow": "",
            "id": "510b3e02-8c82-4a8e-81a3-d2b49ffb18d2"
          }
        ],
        "decryption": "none",
        "fallbacks": []
      },
      "sniffing": {
        "destOverride": [
          "http",
          "tls",
          "quic",
          "fakedns"
        ],
        "enabled": false,
        "metadataOnly": false,
        "routeOnly": false
      },
      "streamSettings": {
        "network": "ws",
        "security": "none",
        "wsSettings": {
          "acceptProxyProtocol": false,
          "headers": {},
          "heartbeatPeriod": 0,
          "host": "vpn.aitoolsfree.tech",
          "path": "/zoom"
        }
      },
      "tag": "inbound-443"
    }
  ],
  "log": {
    "access": "none",
    "dnsLog": false,
    "error": "",
    "loglevel": "warning",
    "maskAddress": ""
  },
  "observatory": null,
  "outbounds": [
    {
      "protocol": "freedom",
      "settings": {
        "domainStrategy": "AsIs",
        "noises": [],
        "redirect": ""
      },
      "tag": "direct"
    },
    {
      "protocol": "blackhole",
      "settings": {},
      "tag": "blocked"
    }
  ],
  "policy": {
    "levels": {
      "0": {
        "statsUserDownlink": true,
        "statsUserUplink": true
      }
    },
    "system": {
      "statsInboundDownlink": true,
      "statsInboundUplink": true,
      "statsOutboundDownlink": false,
      "statsOutboundUplink": false
    }
  },
  "reverse": null,
  "routing": {
    "domainStrategy": "AsIs",
    "rules": [
      {
        "inboundTag": [
          "api"
        ],
        "outboundTag": "api",
        "type": "field"
      },
      {
        "ip": [
          "geoip:private"
        ],
        "outboundTag": "blocked",
        "type": "field"
      },
      {
        "outboundTag": "blocked",
        "protocol": [
          "bittorrent"
        ],
        "type": "field"
      }
    ]
  },
  "stats": {},
  "transport": null
}
```

### Step 2: Create the GitHub Secret

1. Navigate to your GitHub repository.
    
2. Go to **Settings** > **Secrets and variables** > **Actions**.
    
3. Click the **New repository secret** button.
    
4. In the **Name** field, type exactly `VPN_CONFIG`.
    
5. In the **Secret** field, paste the corrected JSON code you copied from **Step 1**.
    
6. Click **Add secret**.
    

### Step 3: Create the Workflow File

1. In your GitHub repository, go to the **Code** tab.
    
2. Click **Add file** > **Create new file**.
    
3. In the file name box, type `.github/workflows/vpn.yml`.
    
4. Paste the **Final GitHub Actions Code** from the top of this guide into the editor.
    
5. Click **Commit changes...** to save the file.
    

### Step 4: Run the VPN

You're all set! To start the VPN:

1. Go to the **Actions** tab in your repository.
    
2. In the left sidebar, click on **Infinite SSH and VPN via Cloudflare Tunnel**.
    
3. Click the **Run workflow** dropdown, and then click the green **Run workflow** button.
    

The process will start. You can click on the running job to see the progress. In the "Start Cloudflare Tunnel" step, you will see your SSH connection details. The VPN will be configured automatically and will run for about 5 hours before the job ends. It will then automatically restart on the 5-hour schedule you defined.
