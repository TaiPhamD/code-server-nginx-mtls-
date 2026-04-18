# code-server + Nginx + mTLS Setup

## Files in this folder

- `code-server-nginx.conf` — Nginx reverse proxy config with mTLS

## Prerequisites

- code-server installed and running on `127.0.0.1:9000`
- Nginx installed (`sudo apt install nginx`)

## Step 1: Generate Certificates

```bash
# Create cert directory
sudo mkdir -p /etc/code-server-certs

# CA (signs both server and client certs)
openssl genrsa -out /etc/code-server-certs/ca.key 4096
openssl req -new -x509 -days 3650 -key /etc/code-server-certs/ca.key \
  -out /etc/code-server-certs/ca.crt -subj "/CN=CodeServer CA"

# Server cert (nginx presents this)
openssl genrsa -out /etc/code-server-certs/server.key 2048
openssl req -new -key /etc/code-server-certs/server.key \
  -out /etc/code-server-certs/server.csr -subj "/CN=localhost"
openssl x509 -req -days 3650 -in /etc/code-server-certs/server.csr \
  -CA /etc/code-server-certs/ca.crt -CAkey /etc/code-server-certs/ca.key \
  -CAcreateserial -out /etc/code-server-certs/server.crt \
  -extfile <(echo -e "subjectAltName=DNS:localhost,IP:127.0.0.1")

# Client cert (browsers present this)
openssl genrsa -out client.key 2048
openssl req -new -key client.key -out client.csr -subj "/CN=code-server-client"
openssl x509 -req -days 3650 -in client.csr \
  -CA /etc/code-server-certs/ca.crt -CAkey /etc/code-server-certs/ca.key \
  -CAcreateserial -out client.crt
openssl pkcs12 -export -out client.p12 \
  -inkey client.key -in client.crt -certfile /etc/code-server-certs/ca.crt \
  -passout pass:YOUR_PASSWORD_HERE

# Set permissions
sudo chmod 600 /etc/code-server-certs/ca.key /etc/code-server-certs/server.key
sudo chmod 644 /etc/code-server-certs/ca.crt /etc/code-server-certs/server.crt
```

## Step 2: Install Nginx Config

```bash
sudo cp code-server-nginx.conf /etc/nginx/sites-available/code-server
sudo rm -f /etc/nginx/sites-enabled/default
sudo ln -sf /etc/nginx/sites-available/code-server /etc/nginx/sites-enabled/code-server
sudo nginx -t && sudo systemctl restart nginx
```

## Step 3: Install Client Cert on Your Device

### Desktop (Chrome / Firefox)

1. Import `client.p12` into your browser's certificate store (password: `YOUR_PASSWORD_HERE`)
2. Import `ca.crt` as a trusted CA authority
3. Visit `https://<server-ip>`

### iOS (Safari)

1. Email/download `client.p12` to your iPhone
2. Open it → Settings → General → **Profile Downloaded** → Install (enter `YOUR_PASSWORD_HERE`)
3. Settings → General → About → **Certificate Trust Settings** → Enable full trust for "CodeServer CA"
4. Open Safari → `https://<server-ip>`

### macOS (Chrome)

1. Double-click `client.p12` → Keychain Access → Login
2. Double-click `ca.crt` → Trust → "When using this certificate: Always Trust"
3. Visit `https://<server-ip>`

## What This Config Does

- **mTLS** — clients must present a valid client certificate signed by our CA
- **HTTP → HTTPS redirect** — port 80 redirects to HTTPS
- **gzip compression** — compresses JS/CSS/JSON for faster loading
- **Static asset caching** — versioned assets cached for 1 year
- **WebSocket support** — proper proxy headers for VS Code websockets
- **Security headers** — HSTS, X-Frame-Options, X-Content-Type-Options
- **Client cert logging** — nginx logs the DN of connecting clients

## Troubleshooting

- **400 "No required SSL certificate"** — Client cert not installed or not trusted
- **WebSocket 101 errors** — Check nginx error log: `sudo tail -f /var/log/nginx/code-server-error.log`
- **Reload config after changes:** `sudo nginx -t && sudo systemctl reload nginx`
