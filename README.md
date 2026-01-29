# Cloudflare Tunnel - Expose Local VM to Internet

Panduan lengkap untuk expose VM lokal ke internet secara **gratis** menggunakan Cloudflare Tunnel.

---

## Apa itu Cloudflare Tunnel?

Cloudflare Tunnel memungkinkan kamu mengekspos aplikasi yang berjalan di jaringan lokal ke internet tanpa perlu:
- Public IP
- Port forwarding
- Firewall configuration

```
┌─────────────────┐      ┌─────────────────┐      ┌─────────────────┐
│   VM LOKAL      │      │   CLOUDFLARE    │      │    INTERNET     │
│                 │ ───► │   TUNNEL        │ ◄─── │                 │
│   Nginx :80     │      │   (gratis)      │      │   User akses    │
└─────────────────┘      └─────────────────┘      └─────────────────┘
                                │
                                ▼
                    https://app.yourdomain.com
                              atau
                    https://random.trycloudflare.com
```

---

## Kenapa Cloudflare Tunnel?

| Feature | Cloudflare Tunnel |
|---------|-------------------|
| Harga | **Gratis** |
| HTTPS | Otomatis (SSL/TLS) |
| Custom Domain | ✅ Support |
| Bandwidth | Unlimited |
| Koneksi | Stabil & Cepat |
| DDoS Protection | ✅ Included |
| Setup | Mudah |

---

## Prerequisites

- VM dengan Ubuntu/Debian
- Aplikasi yang berjalan (misal Nginx di port 80)
- Akun Cloudflare (gratis)
- (Optional) Domain yang sudah ditambahkan ke Cloudflare

---

## Quick Start (5 Menit)

Untuk testing cepat **tanpa perlu domain**:

```bash
# 1. Install cloudflared
curl -L https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64.deb -o cloudflared.deb
sudo dpkg -i cloudflared.deb

# 2. Jalankan quick tunnel
cloudflared tunnel --url http://localhost:80
```

Output:
```
Your quick tunnel has been created! Visit it at:
https://random-words-here.trycloudflare.com
```

🎉 **Done!** URL bisa diakses dari mana saja di dunia.

---

## Setup Lengkap (Dengan Custom Domain)

### Step 1: Daftar Cloudflare

1. Buka https://dash.cloudflare.com/sign-up
2. Buat akun dengan email
3. (Optional) Tambahkan domain kamu ke Cloudflare

---

### Step 2: Install cloudflared di VM

#### Ubuntu/Debian

```bash
# Tambahkan repository
curl -fsSL https://pkg.cloudflare.com/cloudflare-main.gpg | sudo tee /usr/share/keyrings/cloudflare-main.gpg >/dev/null

echo 'deb [signed-by=/usr/share/keyrings/cloudflare-main.gpg] https://pkg.cloudflare.com/cloudflared focal main' | sudo tee /etc/apt/sources.list.d/cloudflared.list

# Install
sudo apt update
sudo apt install cloudflared -y

# Verifikasi
cloudflared --version
```

#### Alternatif: Download Binary Langsung

```bash
# Download
curl -L https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64.deb -o cloudflared.deb

# Install
sudo dpkg -i cloudflared.deb

# Cleanup
rm cloudflared.deb
```

---

### Step 3: Login ke Cloudflare

```bash
cloudflared tunnel login
```

Akan muncul URL di terminal. Buka URL tersebut di browser dan pilih domain yang ingin digunakan.

Setelah authorize, credential akan disimpan di:
```
~/.cloudflared/cert.pem
```

---

### Step 4: Buat Tunnel

```bash
# Buat tunnel dengan nama
cloudflared tunnel create devops-tunnel
```

Output:
```
Tunnel credentials written to /home/user/.cloudflared/<TUNNEL_ID>.json
Created tunnel devops-tunnel with id xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
```

**Catat TUNNEL_ID** — akan digunakan di config.

List tunnel yang ada:
```bash
cloudflared tunnel list
```

---

### Step 5: Buat Configuration File

```bash
nano ~/.cloudflared/config.yml
```

#### Config untuk Single Service

```yaml
tunnel: devops-tunnel
credentials-file: /home/gitlabadmin/.cloudflared/<TUNNEL_ID>.json

ingress:
  - hostname: app.yourdomain.com
    service: http://localhost:80
  - service: http_status:404
```

#### Config untuk Multiple Services

```yaml
tunnel: devops-tunnel
credentials-file: /home/gitlabadmin/.cloudflared/<TUNNEL_ID>.json

ingress:
  # Website utama
  - hostname: app.yourdomain.com
    service: http://localhost:80
  
  # API backend
  - hostname: api.yourdomain.com
    service: http://localhost:3000
  
  # Grafana monitoring
  - hostname: grafana.yourdomain.com
    service: http://localhost:3001
  
  # Jenkins CI/CD
  - hostname: jenkins.yourdomain.com
    service: http://localhost:8080
  
  # Catch-all (wajib di akhir)
  - service: http_status:404
```

**Ganti:**
- `<TUNNEL_ID>` dengan ID tunnel kamu
- `yourdomain.com` dengan domain kamu
- Path credentials sesuai lokasi file `.json`

---

### Step 6: Route DNS

Hubungkan tunnel ke subdomain:

```bash
# Untuk setiap hostname di config
cloudflared tunnel route dns devops-tunnel app.yourdomain.com
cloudflared tunnel route dns devops-tunnel api.yourdomain.com
cloudflared tunnel route dns devops-tunnel grafana.yourdomain.com
```

Cloudflare akan otomatis membuat CNAME record di DNS.

Verifikasi di Cloudflare Dashboard → DNS → Records.

---

### Step 7: Jalankan Tunnel

#### Manual (Foreground)

```bash
cloudflared tunnel run devops-tunnel
```

#### Dengan Config File

```bash
cloudflared tunnel --config ~/.cloudflared/config.yml run devops-tunnel
```

---

### Step 8: Setup sebagai System Service (Recommended)

Supaya tunnel berjalan otomatis saat boot:

```bash
# Install service
sudo cloudflared service install

# Start service
sudo systemctl start cloudflared

# Enable auto-start on boot
sudo systemctl enable cloudflared

# Check status
sudo systemctl status cloudflared
```

#### Jika config tidak di default location:

```bash
sudo cloudflared --config /home/gitlabadmin/.cloudflared/config.yml service install
```

---

## Useful Commands

### Tunnel Management

```bash
# List semua tunnel
cloudflared tunnel list

# Info tunnel specific
cloudflared tunnel info devops-tunnel

# Delete tunnel
cloudflared tunnel delete devops-tunnel

# Cleanup unused tunnels
cloudflared tunnel cleanup devops-tunnel
```

### Service Management

```bash
# Start/Stop/Restart
sudo systemctl start cloudflared
sudo systemctl stop cloudflared
sudo systemctl restart cloudflared

# Check status
sudo systemctl status cloudflared

# View logs
sudo journalctl -u cloudflared -f
```

### Testing

```bash
# Test config syntax
cloudflared tunnel --config ~/.cloudflared/config.yml ingress validate

# Test routing rules
cloudflared tunnel --config ~/.cloudflared/config.yml ingress rule https://app.yourdomain.com
```

---

## Contoh Use Cases

### 1. Expose Web Server

```yaml
ingress:
  - hostname: web.example.com
    service: http://localhost:80
  - service: http_status:404
```

### 2. Expose dengan Path Routing

```yaml
ingress:
  - hostname: app.example.com
    path: /api/*
    service: http://localhost:3000
  - hostname: app.example.com
    service: http://localhost:80
  - service: http_status:404
```

### 3. SSH Access via Browser

```yaml
ingress:
  - hostname: ssh.example.com
    service: ssh://localhost:22
  - service: http_status:404
```

Akses via: `https://ssh.example.com` (Cloudflare Access required)

### 4. TCP Service (Database, etc)

```yaml
ingress:
  - hostname: db.example.com
    service: tcp://localhost:5432
  - service: http_status:404
```

---

## Troubleshooting

### Tunnel Tidak Bisa Connect

```bash
# Check cloudflared service
sudo systemctl status cloudflared

# Check logs
sudo journalctl -u cloudflared -f --no-pager | tail -50

# Verify credentials file exists
ls -la ~/.cloudflared/
```

### Error: "failed to connect to origin"

Pastikan service yang ingin di-expose sudah berjalan:
```bash
# Check apakah nginx/app running
sudo systemctl status nginx
curl localhost:80
```

### Error: "tunnel credentials file not found"

```bash
# Check path di config.yml
# Pastikan TUNNEL_ID benar
cloudflared tunnel list

# Regenerate credentials jika perlu
cloudflared tunnel delete devops-tunnel
cloudflared tunnel create devops-tunnel
```

### DNS Not Resolving

```bash
# Check DNS records di Cloudflare Dashboard
# Atau pakai dig
dig app.yourdomain.com

# Re-route DNS
cloudflared tunnel route dns devops-tunnel app.yourdomain.com
```

---

## Security Best Practices

1. **Gunakan Cloudflare Access** untuk endpoint sensitif (SSH, admin panels)
2. **Jangan expose** database langsung ke internet
3. **Restrict IP** di aplikasi jika memungkinkan
4. **Enable firewall** di VM (UFW)
5. **Monitor logs** secara berkala

---

## Pricing

| Tier | Harga | Tunnels | Features |
|------|-------|---------|----------|
| **Free** | $0 | Unlimited | Basic tunneling, HTTPS |
| Teams | $0 (50 users) | Unlimited | + Access policies |
| Enterprise | Custom | Unlimited | + SLA, Support |

**Untuk personal/belajar, Free tier sudah sangat cukup!**

---

## Quick Reference

```bash
# Install
sudo apt install cloudflared -y

# Quick tunnel (no domain)
cloudflared tunnel --url http://localhost:80

# Login
cloudflared tunnel login

# Create tunnel
cloudflared tunnel create <name>

# Route DNS
cloudflared tunnel route dns <name> <hostname>

# Run tunnel
cloudflared tunnel run <name>

# Install as service
sudo cloudflared service install
sudo systemctl enable cloudflared
sudo systemctl start cloudflared
```

---

## File Locations

| File | Location |
|------|----------|
| Credentials (cert.pem) | `~/.cloudflared/cert.pem` |
| Tunnel credentials | `~/.cloudflared/<TUNNEL_ID>.json` |
| Config file | `~/.cloudflared/config.yml` |
| Service config | `/etc/cloudflared/config.yml` |
| Logs | `journalctl -u cloudflared` |

---

## References

- [Cloudflare Tunnel Documentation](https://developers.cloudflare.com/cloudflare-one/connections/connect-apps/)
- [cloudflared GitHub](https://github.com/cloudflare/cloudflared)
- [Cloudflare Zero Trust](https://www.cloudflare.com/products/zero-trust/)

---

*Expose your local services to the world, securely and for free! 🚀*
