# MongoDB MCP Server on EC2 + MCP Inspector (stdio, streamable_http)

This repo gets you a working MongoDB MCP Server on EC2 with NGINX + Let’s Encrypt, served over HTTPS at your custom domain.

Notes:
- Validated with Node 20.x (NodeSource) and modelcontextprotocol/inspector@latest
- Renewal is automatic; if you’re paranoid, add a root cron to reload NGINX after certbot renew.

<br>

## Goals
- Spin up an EC2 instance and run the MongoDB MCP Server.
- Verify stdio mode locally via MCP Inspector.
- Expose the server via HTTPS using your own domain + NGINX.
- Verify streamable_http(s) via MCP Inspector from your laptop.

<br>

## Prerequisites
- An AWS account with permissions to create EC2 instances and manage Security Groups.
- A registered domain you control (e.g., WordPress.com, Route 53, etc.) to create a subdomain (e.g., `mcp.yourdomain.com`) pointing to EC2.
- Basic familiarity with SSH into EC2.
- A MongoDB database available for queries.
- MCP Inspector installed locally.

<br>

## High-Level Workflow
- Provision EC2 → secure inbound rules (22/80/443).
- Install Node.js runtime and MongoDB MCP Server package.
- Validate the server locally (bind to `127.0.0.1`).
- Install NGINX + Certbot → obtain TLS cert for `mcp.yourdomain.com`.
- Reverse-proxy `/mcp` → local MCP server with SSE-friendly settings.
- Use MCP Inspector to test:
  - stdio (local launch)
  - streamable_http(s) (remote HTTPS URL)

<br>
<br>

# 1) Launch EC2 & Prepare the Host

## 1.1 Create the EC2 instance where you will run the MCP server
- AMI: Amazon Linux 2023
- Instance type: `t3.small`
- Security Group (inbound) as you need:
  - `22/tcp` (SSH) — restrict to your IP(s)
  - `80/tcp` (HTTP) - 80 must be open in the EC2 Security Group for Let’s Encrypt’s HTTP-01 challenge
  - `443/tcp` (HTTPS)
  - `8080/tcp` (Custom TCP)
- Key pair: create/download (e.g., `ssappenfield-keypair.pem`)

<table><tr><td><img src="./img_ec2_micro_instance.jpg" width="800"/></td></tr></table>

## 1.2 SSH in (first login) and basic host prep
```bash
# macOS/Linux
chmod 600 ssappenfield-keypair.pem
ssh -i ssappenfield-keypair.pem ec2-user@<EC2_PUBLIC_IP>
sudo dnf update -y
```

<br>
<br>


# 2) Install, Run & Tets the MongoDB MCP Server (EC2)

## 2.1 Install Node.js (v20) and basic tools
```bash
curl -fsSL https://rpm.nodesource.com/setup_20.x | sudo bash -
sudo dnf install -y nodejs
node -v && npm -v
```

## 2.2 Run the MongoDB MCP Server
```bash
npx -y mongodb-mcp-server@latest \
  --transport http \
  --httpHost=0.0.0.0 \
  --httpPort=8080 \
  --connectionString "mongodb+srv://<USER>:<PASS>@<HOST>/?retryWrites=true&w=majority&appName=<AppName>" \
  --readOnly
```

## 2.3 Do a quick test of HTTP MCP using the command line (laptop terminal)
```bash
curl -iS -X POST http://<EC2_PUBLIC_IP>:8080/mcp \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -d '{
    "jsonrpc":"2.0",
    "id":1,
    "method":"initialize",
    "params":{
      "protocolVersion":"2025-06-18",
      "capabilities":{},
      "clientInfo":{"name":"curl","version":"0.1"}
    }
  }'
```
> Grab the session ID from the output 

> Use the Session ID (e.g. cacef634-8581-4522-b074-ef6dc3b96fc3) to view a list of tools
```bash
curl -iS -X POST http://<EC2_PUBLIC_IP>:8080/mcp \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -H "Mcp-Session-Id: cacef634-8581-4522-b074-ef6dc3b96fc3" \
  -d '{
    "jsonrpc":"2.0",
    "id":2,
    "method":"tools/list",
    "params":{}
  }'
```

> Or use the Session ID (e.g. cacef634-8581-4522-b074-ef6dc3b96fc3) to view a list of databases
```bash
curl -iS -X POST http://<EC2_PUBLIC_IP>:8080/mcp \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -H "Mcp-Session-Id: cacef634-8581-4522-b074-ef6dc3b96fc3" \
  -d '{
    "jsonrpc":"2.0",
    "id":3,
    "method":"tools/call",
    "params":{
      "name":"list-databases",
      "arguments":{}
    }
  }'
```

## 2.4 Optional local MCP Inspector tool test — stdio (Optional)
Running Section 2.4 (stdio) is still worth it because it isolates variables and proves the core pieces work before you involve EC2, HTTPS, DNS, or NGINX. In other words, this is running a local test from your laptop only and does not include EC2.

Run MCP Inspector from your laptop's command terminal
```bash
npx @modelcontextprotocol/inspector@latest
```

Your browser should open automatically with a URL that looks something like: `http://localhost:6274/?MCP_PROXY_AUTH_TOKEN=`

In the inspector UI, use this input and click Connect
- Transport / Type: `stdio`
- Command: `npx`
- Arguments: `-y mongodb-mcp-server@latest --transport stdio --connectionString "mongodb+srv://<USER>:<PASS>@<HOST>/?retryWrites=true&w=majority&appName=<AppName>" --readOnly`

Validate basic actions
- initialize — should return `serverInfo` and `capabilities`.
- tools/list — confirm tools appear (e.g., `list-databases`, etc.).
- Run a simple tool — e.g., `list-databases` and observe output.

<table><tr><td><img src="./img_mcpinsp_stdio.jpg"/></td></tr></table>

<br>
<br>

# 3) Add HTTPS with NGINX + Let’s Encrypt (reverse proxy)

## 3.1 Create DNS for your HTTPS endpoint
You’ll expose MCP at a friendly name like `mcp.<your-domain>`.
1. Go to your DNS records
2. Add A record
   - Name/Host: `mcp`
   - Value/Points to: `<EC2_PUBLIC_IP>`
   - TTL: default is fine

> I'd suggest a short wait (a few minutes) for propagation.

<table><tr><td><img src="./img_dns_record.jpg" width="600"/></td></tr></table>

### Verify DNS and quick connectivity check (laptop terminal)
```bash
dig +short mcp.<your-domain>
ssh -i ssappenfield-keypair.pem ec2-user@mcp.<your-domain>
```

## 3.2 Install and start NGINX (EC2)
```bash
sudo dnf install -y nginx
sudo systemctl enable --now nginx
```

## 3.3 Obtain a certificate with Let’s Encrypt, Certbot (EC2)
Install Certbot:
```bash
sudo dnf install -y certbot python3-certbot-nginx
```

Request the cert (replace with your domain/email):
```bash
sudo certbot --nginx -d mcp.<your-domain> --redirect -m you@example.com --agree-tos -n
```

> If Certbot says it cannot find a matching server block, create the NGINX file in section 4.3, reload NGINX

## 3.4 Create the NGINX vhost with SSE-friendly proxy (EC2)
> replace `<yourdomain>`
```bash
sudo tee /etc/nginx/conf.d/mcp.conf >/dev/null <<'NGINX'
server {
    listen 80;
    server_name mcp.<yourdomain>.com;

    # Allow Let's Encrypt HTTP-01 challenges
    location /.well-known/acme-challenge/ { root /var/www/html; }

    # Redirect everything else to HTTPS
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name mcp.<yourdomain>.com;

    ssl_certificate     /etc/letsencrypt/live/mcp.<yourdomain>.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/mcp.<yourdomain>.com/privkey.pem;

    # MCP control + streaming endpoint
    location /mcp {
        proxy_pass http://127.0.0.1:8080/mcp;

        proxy_set_header Host            $host;
        proxy_set_header X-Forwarded-For $remote_addr;

        # Streaming-/SSE-friendly bits
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_buffering off;
        proxy_read_timeout 1h;
        chunked_transfer_encoding off;
    }

    # Simple OK at /
    location = / {
        return 200 "OK\n";
        add_header Content-Type text/plain;
    }
}
NGINX
```

Validate and reload:
```bash
sudo nginx -t && sudo systemctl reload nginx
```

<br>
<br>

# 4) HTTPS Remote Test with MCP Inspector — streamable_http(s)

## 4.1 Quick curl sanity tests (laptop terminal)
Initialize (must accept both JSON + SSE):
```bash
curl -iS -X POST https://mcp.<yourdomain>.com/mcp \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -d '{
    "jsonrpc":"2.0",
    "id":1,
    "method":"initialize",
    "params":{
      "protocolVersion":"2025-06-18",
      "capabilities":{},
      "clientInfo":{"name":"curl","version":"0.1"}
    }
  }'
```
> You can re-run similar tests as you did previously like listing tools

## 4.2 Use MCP Inspector with Transport: Streamable HTTP (laptop terminal)
```bash
npx @modelcontextprotocol/inspector@latest
```
- Transport / Type: `Streamable HTTP`
- URL: `https://mcp.<yourdomain>.com/mcp`
- Expect: Inspector shows a live session; `initialize` returns, `tools/list` works, and tool executions stream back as SSE events.

> Make sure you use HTTPS. NGINX sends a 301 redirect to the HTTPS URL for HTTP calls. But if the MCP Inspector uses HTTP it will fail because MCP Inspector’s “Streamable HTTP” connection doesn’t follow redirects for the long-lived SSE streams. Over https://…/mcp, there’s no redirect and the proxy to your local MCP server works—so it connects.

<table><tr><td><img src="./img_mcpinsp_https.jpg"/></td></tr></table>

---

## Real-World Considerations
- So yeah, let's state the obvious here. This is just for a sanity check. In a real world scenario, you would implement real security among other things like using systemd service.
