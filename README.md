# Automating SSL with Traefik, Let’s Encrypt, and Cloudflare

Securing all of my services with HTTPS is a must. Instead of managing certificates manually, I automated the process using **Traefik** (reverse proxy), **Let’s Encrypt** (free SSL), and **Cloudflare** (DNS). The result: wildcard and subdomain certificates that renew automatically without intervention.

---

## Setup Overview

I wanted wildcard SSL for subdomains like:

* `hoppscotch.tmtechnical.com`
* `admin-hoppscotch.tmtechnical.com`
* `api-hoppscotch.tmtechnical.com`
* `*.tmtechnical.com`

Traefik handles reverse proxying and certificate requests. Let’s Encrypt provides the certs. Cloudflare validates ownership via the DNS-01 challenge.

---

## Prerequisites

* A domain managed by Cloudflare DNS (mine: **tmtechnical.com**)
* Docker + Portainer (to manage stacks)
* A Cloudflare API token with:

  * Zone → DNS → Edit
  * Zone → Zone → Read

---

## Step 1: Traefik Service (Portainer Stack)

Here’s my Traefik definition (environment variables stored securely in Portainer):

```yaml
services:
  traefik:
    image: traefik:latest
    container_name: traefik
    restart: unless-stopped
    networks:
      - traefik_proxy
    ports:
      - "80:80"
      - "443:443"
    environment:
      CF_DNS_API_TOKEN: ${CF_DNS_API_TOKEN}
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - /opt/traefik/letsencrypt:/letsencrypt
    command:
      - "--entrypoints.web.address=:80"
      - "--entrypoints.websecure.address=:443"
      - "--providers.docker=true"
      - "--certificatesresolvers.le.acme.email=${ACME_EMAIL}"
      - "--certificatesresolvers.le.acme.storage=/letsencrypt/acme.json"
      - "--certificatesresolvers.le.acme.dnschallenge=true"
      - "--certificatesresolvers.le.acme.dnschallenge.provider=cloudflare"
      - "--log.level=DEBUG"

networks:
  traefik_proxy:
    external: true
```

---

## Step 2: ACME Storage

Create the file where Traefik will persist certs:

```bash
mkdir -p /opt/traefik/letsencrypt
touch /opt/traefik/letsencrypt/acme.json
chmod 600 /opt/traefik/letsencrypt/acme.json
```

This ensures Let’s Encrypt can write securely.

---

## Step 3: DNS Records

In Cloudflare DNS:

* Point `hoppscotch.tmtechnical.com`, `admin-hoppscotch.tmtechnical.com`, `api-hoppscotch.tmtechnical.com` → server’s public IP
* Optionally add a wildcard record `*.tmtechnical.com`

---

## Step 4: Certificates

Traefik auto-requests certificates the first time a router with `tls.certresolver=le` is hit. For example, in my Hoppscotch stack:

```yaml
labels:
  - "traefik.enable=true"
  - "traefik.http.routers.hoppscotch-secure.rule=Host(`hoppscotch.tmtechnical.com`)"
  - "traefik.http.routers.hoppscotch-secure.entrypoints=websecure"
  - "traefik.http.routers.hoppscotch-secure.tls.certresolver=le"
  - "traefik.http.services.hoppscotch-secure.loadbalancer.server.port=3000"
```

---

## Step 5: Verify

Check logs:

```bash
docker logs -f traefik | grep acme
```

You should see Traefik issuing certificates for your domains. After success, browsing to `https://hoppscotch.tmtechnical.com` shows a valid lock icon.

---

## Benefits

* 🔄 Fully automated renewals (no cron jobs or certbot)
* 🛡 Free, trusted SSL certs from Let’s Encrypt
* 🌍 Wildcard coverage for entire subdomains
* 🧩 Works seamlessly with Portainer + Docker labels

---

✅ With this setup, any new service added with proper Traefik labels instantly gets a valid certificate.

---

