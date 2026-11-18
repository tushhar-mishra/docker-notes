# 39a · TLS and Reverse Proxies in Production

> **Level:** ★★ intermediate.
>
> **You will learn:** how TLS (the "S" in HTTPS) actually works (certificates, chains, the handshake, SNI), why production containers sit behind a **reverse proxy** that terminates TLS, how to get and renew free certificates automatically with **ACME/Let's Encrypt** (HTTP-01 vs DNS-01), how to run this with Traefik or Caddy in Compose, the headers apps need behind a proxy (`X-Forwarded-*`), timeouts, WebSockets and HTTP/2, trusting a **private CA** inside containers, mTLS between services, and how to debug TLS with `openssl s_client` and `curl -v`.
>
> **You need:** topic 28 (publishing ports), topic 29 (DNS), topics 35–39 (Compose in production).

---

## What it is

- **TLS** (Transport Layer Security) = the protocol that **encrypts** a connection and **proves the server's identity** with a certificate. HTTPS = HTTP inside TLS. (SSL is TLS's old, deprecated predecessor. People still say "SSL certificate".)
- A **certificate** = a file containing a **public key** plus the names it's valid for (e.g. `shop.example.com`), **signed** by a **Certificate Authority (CA)**. A CA is an organisation that browsers and operating systems trust.
- A **reverse proxy** = a server in front of your apps. It receives all client connections (usually on ports 80/443), terminates TLS, and forwards requests to the right backend container by hostname or path. Examples: **Traefik**, **Caddy**, **nginx**, **HAProxy**, **Envoy**.

```
client ──HTTPS:443──► reverse proxy container ──plain HTTP (internal Docker network)──► app containers
          (TLS terminated here: certificate + private key live only in the proxy)
```

---

## Why it exists

- **Every production service must use HTTPS**: browsers mark plain HTTP as "Not secure", passwords and cookies would travel in clear text, and many APIs and standards (HTTP/2 in browsers, secure cookies, OAuth) require TLS.
- **One place for TLS**: instead of every app container handling certificates, the proxy does it once. Apps stay simple and listen on plain HTTP on an internal network.
- **One public entry point**: only the proxy publishes ports (topic 28). Databases and apps aren't exposed at all.
- **Certificates expire** (Let's Encrypt: 90 days, and the industry is moving to shorter lifetimes still). Manual renewal causes outages, so automation (ACME) is mandatory.

---

## Analogy

The **reception desk of an office building**. Visitors (clients) show ID at reception; reception shows its official building permit (certificate) to prove it's the real building, and all conversations through the reception window are private (encryption). Reception then walks the visitor to the right office (backend) along internal corridors (the Docker network). Offices don't each need their own permit and ID checker. The permit is renewed automatically before it expires (ACME), so the building never has to close for paperwork.

---

## How it works inside

### 1. The certificate chain

```
Root CA certificate (pre-installed in OS/browser trust stores, e.g. /etc/ssl/certs/)
   └─ signs ─► Intermediate CA certificate (e.g. Let's Encrypt "R11")
                  └─ signs ─► Your leaf certificate: CN/SAN = shop.example.com, valid 90 days, public key
```

- **SAN** (Subject Alternative Name) = the list of hostnames a certificate is valid for. Clients check the requested hostname against it.
- The server must send its **leaf certificate plus the intermediate(s)** (the "full chain"). A missing intermediate is a classic error: browsers sometimes cope (they cache intermediates), but `curl`, Java and other clients fail with "unable to get local issuer certificate".
- The **private key** never leaves the server. Whoever has it can impersonate you, so protect it like a secret (topic 44).

### 2. The TLS 1.3 handshake (simplified)

1. **ClientHello**: the client says which TLS versions and ciphers it supports, and **which hostname it wants** (**SNI**, Server Name Indication).
2. **ServerHello + certificate**: the server picks the certificate matching that SNI name and sends it with the chain.
3. The client **verifies** the chain up to a trusted root, checks the hostname against the SAN and the validity dates.
4. Both sides derive shared encryption keys (key exchange), and encrypted HTTP starts.

**SNI is what lets one proxy on one IP serve hundreds of domains**, each with its own certificate. The proxy routes on SNI/Host to the right backend.

### 3. TLS termination vs passthrough vs re-encryption

| Mode | Where TLS ends | Use when |
|---|---|---|
| **Termination** (most common) | At the proxy; plain HTTP to backends on a private network | Standard web apps |
| **Re-encryption** | Proxy terminates, then opens a new TLS connection to the backend | Compliance requires encryption on internal networks |
| **Passthrough** | Proxy forwards raw TLS (routes by SNI only); the backend terminates | The backend must see the client certificate, or end-to-end TLS is required |

### 4. ACME and Let's Encrypt

**ACME** (Automatic Certificate Management Environment) is the protocol a client (Traefik, Caddy, certbot) uses to get certificates automatically from a CA like **Let's Encrypt** (free). The CA must verify that you control the domain, through a **challenge**:

| Challenge | How you prove control | Needs | Wildcards (`*.example.com`) |
|---|---|---|---|
| **HTTP-01** | CA fetches `http://<domain>/.well-known/acme-challenge/<token>` from your server on **port 80** | Port 80 reachable from the internet; DNS pointing at the server | No |
| **TLS-ALPN-01** | CA connects on **port 443** with a special TLS extension | Port 443 reachable | No |
| **DNS-01** | You create a TXT record `_acme-challenge.<domain>` (the proxy does it through your DNS provider's API) | API token for your DNS provider | **Yes**. Also works for internal servers not reachable from the internet |

Important operational facts:
- **Persist the ACME storage** (`acme.json` for Traefik, `/data` for Caddy) on a **volume**. Lose it, and the proxy requests new certificates on every restart, quickly hitting **Let's Encrypt rate limits** (for example a weekly limit of duplicate certificates per exact set of names). Then you can't get a certificate at all for days.
- Use Let's Encrypt's **staging** environment while testing configs.
- Renewal happens automatically about 30 days before expiry. **Monitor expiry anyway** (topic 47).

### 5. What apps need to know behind a proxy

The app sees connections **from the proxy**, over plain HTTP. So the proxy adds headers:

| Header | Meaning | App uses it for |
|---|---|---|
| `X-Forwarded-For` | Original client IP (a list if there are several proxies) | Logs, rate limiting, geo |
| `X-Forwarded-Proto: https` | Original scheme | Building correct redirect URLs and secure cookies. Otherwise the app thinks it's on HTTP and redirects in loops |
| `X-Forwarded-Host` | Original Host header | Absolute URLs |
| `Forwarded` | The standardised combined form (RFC 7239; an RFC, "Request for Comments", is an official internet standard document) | Same |

- Apps must trust these headers **only from the proxy** (configure "trusted proxies"), or clients can spoof their IP.
- **HSTS** (HTTP Strict Transport Security, the header `Strict-Transport-Security: max-age=31536000`) tells browsers to always use HTTPS for your domain. Add it once HTTPS works everywhere: it's sticky.
- Redirect port 80 → 443 at the proxy (but keep `/.well-known/acme-challenge/` reachable on 80 if you use HTTP-01).

### 6. Timeouts, WebSockets, HTTP/2, body size

Frequent production surprises at the proxy layer:
- **Timeouts**: proxies have idle/read timeouts (often 60 s). Long requests, report generation, or server-sent events get cut with **502/504**. Set them deliberately per route.
- **WebSockets** need the `Upgrade`/`Connection` headers passed through (Traefik and Caddy do this automatically; nginx needs explicit config).
- **Upload size**: nginx's default `client_max_body_size` is **1 MB**, so uploads fail with **413 Request Entity Too Large**.
- **HTTP/2 and HTTP/3** are usually enabled at the proxy toward clients. Backends can stay on HTTP/1.1.
- **502 Bad Gateway** = the proxy couldn't get a valid response from the backend (backend down, wrong port, connection refused). **504 Gateway Timeout** = the backend was too slow.

### 7. Trusting a private CA inside containers

Companies often use an **internal CA** for internal services (and TLS-inspecting proxies re-sign all traffic with a corporate CA). Containers only trust the CA bundle **in their image**, so calls to internal HTTPS services fail with `x509: certificate signed by unknown authority`.

Fix in the image (Debian/Ubuntu-based):

```dockerfile
COPY corp-root-ca.crt /usr/local/share/ca-certificates/corp-root-ca.crt
RUN update-ca-certificates
```

- Alpine: the same paths with the `ca-certificates` package. RHEL/UBI: `/etc/pki/ca-trust/source/anchors/` + `update-ca-trust`.
- Some runtimes keep **their own** trust store: Java uses its `cacerts` keystore (`keytool -importcert`), Python `requests` uses `certifi` (set `REQUESTS_CA_BUNDLE`), and Node uses its built-in list (`NODE_EXTRA_CA_CERTS`).
- For **dockerd** talking to a private registry: `/etc/docker/certs.d/<registry:port>/ca.crt` (topic 22).

### 8. mTLS between services

**mTLS** (mutual TLS) = the **client also presents a certificate**, so the server knows exactly which service is calling. It's used for zero-trust internal networks, the Docker API over TCP (`--tlsverify`, topic 34), databases and service meshes. Certificates are issued by an internal CA and rotated automatically (service mesh, SPIFFE/SPIRE, cert-manager in Kubernetes; topic 44).

---

## Commands / files

### Traefik with automatic Let's Encrypt certificates (Compose)

```yaml
# compose.yaml (on a server whose DNS name shop.example.com points to it, ports 80/443 open)
services:
  proxy:
    image: traefik:v3.5
    command:
      - --providers.docker=true
      - --providers.docker.exposedbydefault=false
      - --entrypoints.web.address=:80
      - --entrypoints.web.http.redirections.entrypoint.to=websecure
      - --entrypoints.websecure.address=:443
      - --certificatesresolvers.le.acme.email=ops@example.com
      - --certificatesresolvers.le.acme.storage=/acme/acme.json
      - --certificatesresolvers.le.acme.httpchallenge.entrypoint=web
      # while testing, use the staging CA:
      # - --certificatesresolvers.le.acme.caserver=https://acme-staging-v02.api.letsencrypt.org/directory
    ports: ["80:80", "443:443"]
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro    # see topic 40: prefer a socket proxy in production
      - acme:/acme
    restart: unless-stopped

  web:
    image: ghcr.io/acme/shop-web:1.4.2
    labels:
      - traefik.enable=true
      - traefik.http.routers.web.rule=Host(`shop.example.com`)
      - traefik.http.routers.web.entrypoints=websecure
      - traefik.http.routers.web.tls.certresolver=le
      - traefik.http.services.web.loadbalancer.server.port=8000
      - traefik.http.middlewares.hsts.headers.stsSeconds=31536000
      - traefik.http.routers.web.middlewares=hsts
    restart: unless-stopped

volumes:
  acme:
```

- Only the proxy publishes ports. `web` listens on 8000 on the internal network only.
- `acme` volume = certificate storage, persisted across restarts (rate limits!).
- Traefik forwards `X-Forwarded-For/Proto/Host` automatically.

### Caddy: the shortest possible HTTPS setup

```
# Caddyfile
shop.example.com {
    reverse_proxy web:8000
}
```

```yaml
services:
  caddy:
    image: caddy:2
    ports: ["80:80", "443:443", "443:443/udp"]     # 443/udp = HTTP/3
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile:ro
      - caddy_data:/data                           # certificates: persist!
    restart: unless-stopped
volumes:
  caddy_data:
```

- Caddy gets and renews certificates automatically for every site name in the Caddyfile.

### nginx (static certificates, e.g. from your company CA)

```nginx
server {
    listen 443 ssl;
    http2 on;
    server_name shop.example.com;
    ssl_certificate     /etc/nginx/tls/fullchain.pem;   # leaf + intermediates
    ssl_certificate_key /etc/nginx/tls/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    client_max_body_size 20m;

    location / {
        proxy_pass http://web:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_read_timeout 120s;
        proxy_http_version 1.1;                        # needed for WebSockets
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```

### Debug TLS from the command line

```bash
openssl s_client -connect shop.example.com:443 -servername shop.example.com </dev/null 2>/dev/null \
  | openssl x509 -noout -subject -issuer -dates -ext subjectAltName
```

```
subject=CN = shop.example.com
issuer=C = US, O = Let's Encrypt, CN = R11
notBefore=Oct 20 08:12:01 2026 GMT
notAfter=Jan 18 08:12:00 2027 GMT
X509v3 Subject Alternative Name:
    DNS:shop.example.com
```

- `-servername` sends SNI. Without it, you may get the proxy's **default** certificate and draw the wrong conclusion.
- `issuer` = who signed it. `notAfter` = expiry date. SAN = the valid names.

Check the chain the server sends:

```bash
openssl s_client -connect shop.example.com:443 -servername shop.example.com -showcerts </dev/null 2>/dev/null | grep -E '^ *[0-9] s:|i:'
```

```
 0 s:CN = shop.example.com
   i:C = US, O = Let's Encrypt, CN = R11
 1 s:C = US, O = Let's Encrypt, CN = R11
   i:C = US, O = Internet Security Research Group, CN = ISRG Root X1
```

- Entry `0` = leaf, `1` = intermediate. Only one entry = **missing intermediate**.

Verbose curl:

```bash
curl -v https://shop.example.com/ -o /dev/null 2>&1 | grep -E 'SSL connection|subject:|issuer:|expire date|HTTP/'
curl -v --resolve shop.example.com:443:10.0.0.21 https://shop.example.com/   # test one server, bypassing DNS
```

Test from **inside a container** (to catch missing CA bundles):

```bash
docker run --rm curlimages/curl -sS https://internal.corp.example.com/health
```

```
curl: (60) SSL certificate problem: unable to get local issuer certificate
```

→ the image doesn't trust your internal CA (section 7).

Expiry monitoring (Prometheus blackbox exporter metric idea): `probe_ssl_earliest_cert_expiry - time() < 14*24*3600` → alert 14 days before expiry.

---

## Hands-on lab

(No public domain needed. We use a local CA.)

1. **Make a local CA and a server certificate:**
   ```bash
   mkdir -p ~/tlslab && cd ~/tlslab
   openssl req -x509 -newkey rsa:2048 -nodes -days 30 -keyout ca.key -out ca.crt -subj "/CN=Lab Root CA"
   openssl req -newkey rsa:2048 -nodes -keyout site.key -out site.csr -subj "/CN=app.lab.local"
   printf "subjectAltName=DNS:app.lab.local" > san.ext
   openssl x509 -req -in site.csr -CA ca.crt -CAkey ca.key -CAcreateserial -days 30 -out site.crt -extfile san.ext
   ```
   - **CSR** (Certificate Signing Request) = your public key + requested names, sent to the CA to be signed.
2. **Run nginx with it** in front of a backend (`hello-flask` from topic 14, or `nginx` itself as the backend), using the config above with `server_name app.lab.local`, mounting `site.crt`/`site.key`.
3. **Test:**
   ```bash
   curl -v --resolve app.lab.local:443:127.0.0.1 https://app.lab.local/           # fails: unknown CA
   curl --cacert ca.crt --resolve app.lab.local:443:127.0.0.1 https://app.lab.local/   # works
   ```
4. **Trust the CA inside a container:** build an image that copies `ca.crt` into `/usr/local/share/ca-certificates/` and runs `update-ca-certificates`. Use it to curl the proxy over a shared Docker network.
5. **Break it on purpose:** a wrong `server_name`/SAN (hostname mismatch), an expired certificate (`-days 0`), a missing `X-Forwarded-Proto` (watch an app that builds redirect URLs loop between HTTP and HTTPS), and `client_max_body_size` (upload 2 MB → 413). Note each error message.
6. Clean up: remove the containers and `~/tlslab`.

---

## Where you'll use it

- **Every internet-facing service.** The proxy + ACME pattern is standard on single hosts.
- **Kubernetes:** the same ideas as Ingress/Gateway API controllers + **cert-manager** (ACME in Kubernetes).
- **Incidents:** "certificate expired", "502 after deploy", "redirect loop", "413 on upload", "x509 unknown authority in containers" are all very common tickets.
- **Interviews:** "What happens in a TLS handshake?", "How does one IP serve many HTTPS sites?", "502 vs 504?"

---

## When it breaks

| Symptom / error | Meaning | Fix |
|---|---|---|
| Browser: `NET::ERR_CERT_DATE_INVALID` | Certificate expired | Fix renewal (ACME logs, storage volume, port 80 reachability); alert on expiry |
| `curl: (60) unable to get local issuer certificate` | Missing intermediate on the server, or the client doesn't trust the CA | Serve the full chain; add the CA to the image's trust store |
| `x509: certificate is valid for a.example.com, not b.example.com` | Hostname not in SAN, or wrong cert served (missing SNI) | Correct SAN; send SNI; check router rules |
| Let's Encrypt `too many certificates already issued` / `rateLimited` | Lost ACME storage, repeated requests | Persist storage; use staging for tests; wait out the limit |
| ACME HTTP-01 fails `Timeout during connect` | Port 80 blocked (security group/firewall), or DNS points elsewhere | Open 80; check DNS; or use DNS-01 |
| 502 Bad Gateway | Backend down, wrong port/hostname in the proxy, backend bound to 127.0.0.1 | `docker logs proxy`; check the backend from the proxy's network (topic 30) |
| 504 Gateway Timeout | Backend slower than the proxy timeout | Fix slowness or raise the timeout for that route |
| Infinite redirect loop | App doesn't see `X-Forwarded-Proto: https`, or doesn't trust the proxy | Configure trusted proxies; pass the header |
| 413 Request Entity Too Large | Proxy body-size limit | Raise it (nginx `client_max_body_size`) |
| WebSocket connections drop or never upgrade | Upgrade headers not passed, or idle timeout | Pass the headers; raise idle timeouts |
| All client IPs in app logs = proxy IP | App ignores `X-Forwarded-For` | Configure the app's trusted-proxy setting |

---

## Quick recap

- TLS encrypts and authenticates. A certificate (public key + SAN names, signed by a CA) must be served with its **intermediate chain**, and the **private key** stays secret.
- The handshake's **SNI** lets one proxy serve many domains. The proxy **terminates TLS** and forwards plain HTTP on an internal network. Only the proxy publishes ports.
- **ACME** (Let's Encrypt) automates certificates: HTTP-01 (port 80), TLS-ALPN-01 (443), DNS-01 (API; wildcards, internal hosts). **Persist the storage** and monitor expiry.
- Apps behind a proxy need `X-Forwarded-For/Proto/Host` and a trusted-proxy setting. Watch timeouts (504), backend errors (502), body size (413) and WebSockets.
- Containers trust only their image's CA bundle: add private CAs in the Dockerfile (plus Java/Python/Node specifics). mTLS = both sides present certificates.

**Interview questions**

1. **Q: What happens during a TLS handshake?**
   A: The client sends ClientHello with supported versions/ciphers and the hostname (SNI). The server responds with its certificate chain. The client verifies the chain to a trusted root and checks the hostname and dates. Both sides perform a key exchange to derive session keys, then encrypted application data flows.

2. **Q: How do you serve HTTPS for several containerised apps on one server?**
   A: Run a reverse proxy (Traefik, Caddy or nginx) as the only container publishing 80/443. It terminates TLS with per-host certificates chosen by SNI, gets and renews them via ACME with persisted storage, and routes by hostname to backend containers on an internal Docker network.

3. **Q: An app container fails calling an internal HTTPS API with "x509: certificate signed by unknown authority". Fix?**
   A: The image doesn't trust the internal CA. Add the CA certificate to the image's trust store (`update-ca-certificates`, or update-ca-trust on RHEL), plus any runtime-specific store (Java cacerts, `REQUESTS_CA_BUNDLE`, `NODE_EXTRA_CA_CERTS`), and rebuild. Don't disable verification.
