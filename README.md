# 🧭HAProxy Load Balancer Deployment Guide  
## For Red Hat-Based Distributions

📌 **IMPORTANT NOTE:** `<HAProxy-ip>` is the IP address of your HAProxy VM/server. Replace `<HAProxy-ip>` with your actual IP (e.g., `10.0.0.170`) in all `bind` directives, firewall rules, and test commands throughout this guide.

> ✅ This guide works on **any modern Red Hat-based distribution** using `dnf` and `firewalld`.

---

## 🟢 STEP 1: Update System & Install HAProxy
```bash
# 1. Update all packages to latest stable
sudo dnf update -y

# 2. Install HAProxy from official distribution repositories
sudo dnf install haproxy -y

# 3. Verify installed version
haproxy -v
```
> 📌 **Version Note:**    
> The configs below are compatible with **HAProxy 2.4+**.

---

## 🟢 STEP 2: Apply Minimal Configuration (Ready-to-Run)
This configuration uses your exact preferred structure with placeholders. Works immediately after updating IPs.

```bash
# Backup original config
sudo cp /etc/haproxy/haproxy.cfg /etc/haproxy/haproxy.cfg.bak

# Open config for editing
sudo nano /etc/haproxy/haproxy.cfg
```
## 🔧 Basic minimal Configuration Template
```ini
global
    log         127.0.0.1 local2
    chroot      /var/lib/haproxy
    pidfile     /var/run/haproxy.pid
    maxconn     4000
    user        haproxy
    group       haproxy
    daemon
    stats socket /var/lib/haproxy/stats

defaults
    mode                    http
    log                     global
    option                  httplog
    option                  dontlognull
    option                  redispatch
    retries                 3
    timeout http-request    10s
    timeout connect         10s
    timeout client          1m
    timeout server          1m
    timeout check           10s

frontend http_front
    # Change IP/Port to your HAProxy Server's one
    bind <HAProxy-ip>:<listen_port>
    default_backend web_servers

    # Optional stats dashboard (uncomment to enable)
    # stats enable
    # stats uri /stats
    # stats auth admin:ChangeMe123!

backend web_servers
    balance roundrobin
    option httpchk
    http-check send meth GET uri /
    http-check expect status 200

    # Change IP/Port to your backend servers
    server web0 <backend_ip_1>:<backend_port> check
    server web1 <backend_ip_2>:<backend_port> check
```
**Save & exit:** `Ctrl+O` → `Enter` → `Ctrl+X`

### 🔑 Minimal Config Placeholder Reference
| Placeholder | Default Value | What to Replace |
|-------------|---------------|-----------------|
| `<HAProxy-ip>` | *(required)* | Your HAProxy server IP (e.g., `10.0.0.170`) |
| `<listen_port>` | `80` | Port clients connect to (`80` or `443`) |
| `<backend_ip_1>` | *(required)* | First backend server IP (e.g., `192.168.1.10`) |
| `<backend_ip_2>` | *(required)* | Second backend server IP (e.g., `192.168.1.11`) |
| `<backend_port>` | `80` | Port your backend app listens on |

> 💡 Add more `server` lines if you have more backends. Delete lines you don't need.

---

## 🔧 ADVANCED: Fully Customizable Configuration Template
Use this template for production-grade tuning, TLS termination, custom timeouts, or complex routing. Replace all `<placeholders>` with your exact values.

```ini
#---------------------------------------------------------------------
# Global Settings
#---------------------------------------------------------------------
global
    log /dev/log    local0
    log /dev/log    local1 notice
    chroot      /var/lib/haproxy
    stats socket /run/haproxy/admin.sock mode 660 level admin
    stats timeout 30s
    user        haproxy
    group       haproxy
    daemon
    maxconn     <max_connections>
    tune.ssl.default-dh-param <dh_param_size>

#---------------------------------------------------------------------
# Default Settings
#---------------------------------------------------------------------
defaults
    log                     global
    mode                    <proxy_mode>
    option                  httplog
    option                  dontlognull
    option                  redispatch
    retries                 3
    timeout http-request    10s
    timeout connect         <connect_timeout>
    timeout client          <client_timeout>
    timeout server          <server_timeout>
    timeout check           10s
    errorfile 400 /etc/haproxy/errors/400.http
    errorfile 403 /etc/haproxy/errors/403.http
    errorfile 408 /etc/haproxy/errors/408.http
    errorfile 500 /etc/haproxy/errors/500.http
    errorfile 502 /etc/haproxy/errors/502.http
    errorfile 503 /etc/haproxy/errors/503.http
    errorfile 504 /etc/haproxy/errors/504.http

#---------------------------------------------------------------------
# Frontend: Client Listener
#---------------------------------------------------------------------
frontend http_front
    bind <HAProxy-ip>:<listen_port>
    # bind <HAProxy-ip>:443 ssl crt <ssl_cert_path> alpn h2,http/1.1  # Uncomment for HTTPS

    default_backend <backend_name>

    http-request set-header X-Forwarded-For %[src]
    http-request set-header X-Forwarded-Proto %[ssl_fc,iif(https,http)]
    # http-request redirect scheme https unless { ssl_fc }  # Uncomment to force HTTPS

    # Statistics Dashboard
    stats enable
    stats uri  <stats_uri>
    stats refresh <stats_refresh_interval>s
    stats auth <stats_username>:<stats_password>

#---------------------------------------------------------------------
# Backend: Server Pool
#---------------------------------------------------------------------
backend <backend_name>
    balance <balance_algorithm>
    option httpchk
    http-check send meth GET uri <health_check_path>
    http-check expect status <health_check_status>

    server <server1_name> <server1_ip>:<server1_port> check inter 5s fall 3 rise 2
    server <server2_name> <server2_ip>:<server2_port> check inter 5s fall 3 rise 2
    server <server3_name> <server3_ip>:<server3_port> check inter 5s fall 3 rise 2
```

### 🔑 Advanced Config Placeholder Reference
| Placeholder | Default Value | Notes / Recommendations |
|-------------|---------------|--------------------------|
| `<max_connections>` | `4096` | Increase to `16384`+ for high-traffic |
| `<dh_param_size>` | `2048` | Use `4096` for strict compliance |
| `<proxy_mode>` | `http` | Use `tcp` for raw traffic (DBs, SSH) |
| `<connect/client/server_timeout>` | `10s`/`1m`/`1m` | Adjust for slow APIs or WebSockets |
| `<listen_port>` | `80` | Use `443` if enabling SSL/TLS |
| `<ssl_cert_path>` | *(commented)* | Create: `cat cert.pem key.pem > bundle.pem` |
| `<backend_name>` | `web_servers` | Must match in `frontend` & `backend` |
| `<stats_uri/username/password>` | `/stats`/`admin`/`ChangeMe123!` | Change credentials before production |
| `<balance_algorithm>` | `roundrobin` | `leastconn`, `source`, `uri` |
| `<health_check_path>` | `/` | Use `/health` or `/ping` if available |
| `<health_check_status>` | `200` | Use `200-399` to accept redirects |
| `<server#_ip/port>` | *(required)* | Must be reachable from `<HAProxy-ip>` |

---

## 🟢 STEP 3: Validate Syntax & Start Service
```bash
# 1. Validate configuration (MUST return "Configuration file is valid")
sudo haproxy -c -f /etc/haproxy/haproxy.cfg

# 2. Enable & start HAProxy
sudo systemctl enable --now haproxy

# 3. Verify service status
sudo systemctl status haproxy --no-pager
```
⚠️ **If validation fails**, check for typos, missing values, or mismatched backend names.

---

## 🟢 STEP 4: Configure Firewall & SELinux (Universal for RHEL-Based)
```bash
# Open HTTP & HTTPS ports (firewalld is default on all modern RHEL-based distros)
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --reload

# Verify applied rules
sudo firewall-cmd --list-all
```

🔒 **SELinux Note:**  
Ports `80` and `443` are permitted by default on all RHEL-based systems.  
If `<listen_port>` is a non-standard value (e.g., `8080`), run:
```bash
# Install SELinux management tools (if not already installed)
sudo dnf install policycoreutils-python-utils -y

# Allow the new port in SELinux
sudo semanage port -a -t http_port_t -p tcp <listen_port>

# Also open in firewalld
sudo firewall-cmd --permanent --add-port=<listen_port>/tcp
sudo firewall-cmd --reload
```

---

## 🟢 STEP 5: Test Load Balancing & Monitor
```bash
# Test connectivity (run 3+ times to verify traffic rotation)
curl -I http://<HAProxy-ip>

# View live HAProxy logs (works on all systemd-based RHEL distros)
journalctl -u haproxy -f --no-pager

# Access stats dashboard in browser (if enabled)
# http://<HAProxy-ip>/stats | User: admin | Pass: ChangeMe123!
```
✅ **Expected:** HTTP `200` responses cycling between your backend servers.

---

## 🔁 Zero-Downtime Configuration Reload (All RHEL-Based Distros)
Never use `restart` in production. Always validate then gracefully reload:
```bash
sudo haproxy -c -f /etc/haproxy/haproxy.cfg && sudo systemctl reload haproxy
```

---

## 📋 Final Deployment Checklist
- [ ] Replace `<HAProxy-ip>` with your actual HAProxy server IP everywhere
- [ ] Update `<backend_ip_1>`, `<backend_ip_2>`, and `<backend_port>` in minimal config
- [ ] Verify all backend IPs are reachable from `<HAProxy-ip>` (`ping` / `curl`)
- [ ] Ensure health endpoints return `200` on all backends
- [ ] Change `stats auth` credentials if enabling the dashboard
- [ ] Confirm `firewall-cmd --list-all` shows correct open ports
- [ ] Run `curl -I http://<HAProxy-ip>` multiple times to verify distribution

---

## 🔍 Troubleshooting Quick Reference
| Symptom | Solution |
|---------|----------|
| `Configuration file is invalid` | Run `sudo haproxy -c -f /etc/haproxy/haproxy.cfg` to see exact error line |
| `Connection refused` on `<HAProxy-ip>` | Check `sudo systemctl status haproxy` and `sudo firewall-cmd --list-all` |
| Backends marked `DOWN` in stats | Verify backend servers are running and reachable (`curl http://<backend_ip>:<port>`) |
| SELinux blocking connections | Check audit log: `sudo ausearch -m avc -ts recent \| grep haproxy` |
| Stats dashboard not loading | Ensure `stats auth` credentials are correct and firewall allows port 80 |

---

## 📚 Official Documentation References
| Resource | Link |
|----------|------|
| HAProxy Configuration Manual (Latest Stable) | https://docs.haproxy.org/ |
| RHEL Configuring HAProxy | https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/ |

---

🎉 **Done!** Your HAProxy load balancer is now ready to deploy on `<HAProxy-ip>:<listen_port>` across **all modern Red Hat-based distributions**.

> 💡 **Pro Tip:** Save your final config with a timestamp for easy rollback:  
> `sudo cp /etc/haproxy/haproxy.cfg /root/haproxy_$(date +%F).cfg`

Reply with your actual `<HAProxy-ip>`, backend IPs, ports, and health path if you need a pre-filled, production-ready config block.
