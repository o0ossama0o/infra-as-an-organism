# Chapter 6: The Immune System - Security

## Introduction: Defense in Depth

Just as a biological organism has multiple layers of defense—skin, mucous membranes, white blood cells, antibodies—a healthy infrastructure organism requires defense in depth. This chapter explores the security layer of infrastructure as an organism, implementing protection mechanisms that prevent, detect, and respond to threats.

Modern infrastructure security has evolved from perimeter-based "castle and moat" approaches to comprehensive, layered strategies that assume breach and focus on limiting damage. We'll examine practical implementation of security controls, from network segmentation to secrets management, using production-ready tools and patterns.

**What you'll learn**:
- Network security and segmentation with iptables and UFW
- Secrets management with Vault and encrypted storage
- SSL/TLS automation with Let's Encrypt and cert-manager
- Authentication and authorization patterns (OAuth2, JWT, mTLS)
- Container security and image scanning
- Intrusion detection with Fail2ban and OSSEC
- Security monitoring and incident response
- Compliance and security hardening

---

## The Evolution of Infrastructure Security

### Phase 1: Perimeter Security (2000-2010)

**The "Castle and Moat" Era**

```
Internet → Firewall → Internal Network (trusted zone)
```

**Characteristics**:
- Single perimeter firewall
- Internal network fully trusted
- VPN for remote access
- Minimal internal segmentation

**Weaknesses**:
- Single point of failure
- No defense once perimeter breached
- East-west traffic unmonitored
- Cannot handle cloud/microservices

### Phase 2: Defense in Depth (2010-2016)

**Layered Security**

```
Internet → WAF → Load Balancer → Application Firewall → Services
         ↓          ↓                ↓                    ↓
      DMZ Zone   Public Zone     Private Zone      Database Zone
```

**Characteristics**:
- Multiple security layers
- Network segmentation by sensitivity
- Application-layer firewalls
- Intrusion detection systems

**Improvements**:
- Multiple barriers to breach
- Limited blast radius
- Better visibility
- Zone-based access control

### Phase 3: Zero Trust Architecture (2016-2020)

**"Never Trust, Always Verify"**

```
Every request → Authenticate → Authorize → Encrypt → Audit
```

**Characteristics**:
- No implicit trust (even internal)
- Identity-based access control
- Continuous verification
- Microsegmentation

**Key principles**:
- Verify explicitly (every request)
- Use least privilege access
- Assume breach (limit damage)
- End-to-end encryption

### Phase 4: Cloud-Native Security (2020+)

**Security as Code + Runtime Protection**

```
Build → Scan → Sign → Deploy → Monitor → Respond
  ↓       ↓      ↓       ↓         ↓         ↓
Policy  Vulns  Provenance Enforce   Detect   Auto-remediate
```

**Characteristics**:
- Security policies as code
- Automated vulnerability scanning
- Runtime security monitoring
- Supply chain security
- Service mesh security

**Modern requirements**:
- Ephemeral infrastructure
- Multi-cloud security
- Container and Kubernetes security
- API security
- DevSecOps integration

---

## Network Security: The Outer Layer

### Firewall Configuration with UFW

UFW (Uncomplicated Firewall) provides a user-friendly interface to iptables for host-based firewalling.

**Basic UFW Setup**:

```bash
#!/bin/bash
# Basic UFW configuration for a web server

# Reset to defaults
ufw --force reset

# Default policies: deny incoming, allow outgoing
ufw default deny incoming
ufw default allow outgoing

# Allow SSH (change port if non-standard)
ufw allow 22/tcp comment 'SSH'

# Allow HTTP/HTTPS
ufw allow 80/tcp comment 'HTTP'
ufw allow 443/tcp comment 'HTTPS'

# Allow specific service from specific IP
ufw allow from 10.0.1.0/24 to any port 5432 proto tcp comment 'PostgreSQL from app subnet'

# Rate limiting for SSH (prevent brute force)
ufw limit 22/tcp comment 'SSH rate limit'

# Enable firewall
ufw --force enable

# Show status
ufw status verbose
```

**Advanced UFW Rules**:

```bash
#!/bin/bash
# Advanced UFW configuration with service-specific rules

# Application profiles (create in /etc/ufw/applications.d/)
cat > /etc/ufw/applications.d/myapp <<EOF
[MyApp API]
title=MyApp API Server
description=API server for MyApp
ports=8080/tcp

[MyApp Metrics]
title=MyApp Metrics
description=Prometheus metrics endpoint
ports=9090/tcp
EOF

# Reload application profiles
ufw app update MyApp

# Allow specific applications
ufw allow 'MyApp API'

# Allow metrics only from monitoring subnet
ufw allow from 10.0.2.0/24 to any app 'MyApp Metrics'

# Block specific IP range
ufw deny from 192.168.100.0/24 comment 'Blocked subnet'

# Allow Docker containers (if using Docker)
ufw allow in on docker0
ufw allow out on docker0

# Log denied connections
ufw logging on

# Set log level
ufw logging medium
```

**UFW with Docker Integration**:

Docker bypasses UFW rules by default. Fix with `/etc/ufw/after.rules`:

```bash
# /etc/ufw/after.rules - Add these rules at the end

# BEGIN UFW AND DOCKER
*filter
:ufw-user-forward - [0:0]
:ufw-docker-logging-deny - [0:0]
:DOCKER-USER - [0:0]

-A DOCKER-USER -j ufw-user-forward
-A DOCKER-USER -j RETURN -s 10.0.0.0/8
-A DOCKER-USER -j RETURN -s 172.16.0.0/12
-A DOCKER-USER -j RETURN -s 192.168.0.0/16

-A DOCKER-USER -j ufw-docker-logging-deny -p tcp -m tcp --tcp-flags FIN,SYN,RST,ACK SYN -d 192.168.0.0/16
-A DOCKER-USER -j ufw-docker-logging-deny -p tcp -m tcp --tcp-flags FIN,SYN,RST,ACK SYN -d 10.0.0.0/8
-A DOCKER-USER -j ufw-docker-logging-deny -p tcp -m tcp --tcp-flags FIN,SYN,RST,ACK SYN -d 172.16.0.0/12
-A DOCKER-USER -j ufw-docker-logging-deny -p udp -m udp --dport 0:32767 -d 192.168.0.0/16
-A DOCKER-USER -j ufw-docker-logging-deny -p udp -m udp --dport 0:32767 -d 10.0.0.0/8
-A DOCKER-USER -j ufw-docker-logging-deny -p udp -m udp --dport 0:32767 -d 172.16.0.0/12

-A DOCKER-USER -j RETURN

-A ufw-docker-logging-deny -m limit --limit 3/min --limit-burst 10 -j LOG --log-prefix "[UFW DOCKER BLOCK] "
-A ufw-docker-logging-deny -j DROP

COMMIT
# END UFW AND DOCKER
```

### Network Segmentation with iptables

For more complex scenarios, use iptables directly:

```bash
#!/bin/bash
# Network segmentation with iptables

# Flush existing rules
iptables -F
iptables -X
iptables -t nat -F
iptables -t nat -X
iptables -t mangle -F
iptables -t mangle -X

# Default policies
iptables -P INPUT DROP
iptables -P FORWARD DROP
iptables -P OUTPUT ACCEPT

# Allow loopback
iptables -A INPUT -i lo -j ACCEPT
iptables -A OUTPUT -o lo -j ACCEPT

# Allow established connections
iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

# Allow SSH (with rate limiting)
iptables -A INPUT -p tcp --dport 22 -m conntrack --ctstate NEW -m recent --set
iptables -A INPUT -p tcp --dport 22 -m conntrack --ctstate NEW -m recent --update --seconds 60 --hitcount 4 -j DROP
iptables -A INPUT -p tcp --dport 22 -j ACCEPT

# Allow HTTP/HTTPS
iptables -A INPUT -p tcp --dport 80 -j ACCEPT
iptables -A INPUT -p tcp --dport 443 -j ACCEPT

# FORWARD rules for Docker networks
iptables -A FORWARD -i docker0 -o eth0 -j ACCEPT
iptables -A FORWARD -i eth0 -o docker0 -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT

# Allow specific subnets to access services
# Web tier can access API tier
iptables -A FORWARD -s 10.0.1.0/24 -d 10.0.2.0/24 -p tcp --dport 8080 -j ACCEPT

# API tier can access database tier
iptables -A FORWARD -s 10.0.2.0/24 -d 10.0.3.0/24 -p tcp --dport 5432 -j ACCEPT

# Block everything else between tiers
iptables -A FORWARD -s 10.0.1.0/24 -d 10.0.3.0/24 -j DROP

# Log dropped packets (limit to prevent log spam)
iptables -A INPUT -m limit --limit 5/min -j LOG --log-prefix "iptables INPUT denied: " --log-level 7
iptables -A FORWARD -m limit --limit 5/min -j LOG --log-prefix "iptables FORWARD denied: " --log-level 7

# Save rules (Ubuntu/Debian)
iptables-save > /etc/iptables/rules.v4

# Save rules (RHEL/CentOS)
service iptables save
```

**Network Zones**:

```
┌─────────────────────────────────────────────────────┐
│                   Internet                          │
└──────────────────┬──────────────────────────────────┘
                   │
            ┌──────▼──────┐
            │ DMZ Zone    │ 10.0.1.0/24
            │ - Traefik   │ (Public-facing services)
            │ - WAF       │
            └──────┬──────┘
                   │
            ┌──────▼──────┐
            │ App Zone    │ 10.0.2.0/24
            │ - API       │ (Application tier)
            │ - Services  │ Can access DB zone
            └──────┬──────┘
                   │
            ┌──────▼──────┐
            │ Data Zone   │ 10.0.3.0/24
            │ - Database  │ (Data tier)
            │ - Cache     │ No internet access
            └─────────────┘
```

---

## Secrets Management: Protecting Sensitive Data

### HashiCorp Vault

Vault provides centralized secrets management with strong access controls, audit logging, and dynamic secrets.

**Vault Setup with Docker Compose**:

```yaml
# docker-compose.vault.yml
version: '3.8'

services:
  vault:
    image: vault:1.15
    container_name: vault
    restart: unless-stopped
    ports:
      - "8200:8200"
    environment:
      VAULT_ADDR: 'http://0.0.0.0:8200'
      VAULT_API_ADDR: 'http://0.0.0.0:8200'
      VAULT_ADDRESS: 'http://0.0.0.0:8200'
    cap_add:
      - IPC_LOCK
    volumes:
      - ./vault/data:/vault/data
      - ./vault/config:/vault/config
      - ./vault/logs:/vault/logs
    command: server
    networks:
      - secure

  # Vault UI (optional)
  vault-ui:
    image: djenriquez/vault-ui:latest
    container_name: vault-ui
    restart: unless-stopped
    ports:
      - "8000:8000"
    environment:
      VAULT_URL_DEFAULT: http://vault:8200
      VAULT_AUTH_DEFAULT: TOKEN
    depends_on:
      - vault
    networks:
      - secure

networks:
  secure:
    driver: bridge
```

**Vault Configuration** (`vault/config/vault.hcl`):

```hcl
# vault/config/vault.hcl

ui = true

storage "file" {
  path = "/vault/data"
}

listener "tcp" {
  address     = "0.0.0.0:8200"
  tls_disable = 1
  # In production, enable TLS:
  # tls_cert_file = "/vault/config/cert.pem"
  # tls_key_file  = "/vault/config/key.pem"
}

api_addr = "http://0.0.0.0:8200"
cluster_addr = "https://0.0.0.0:8201"

# Enable audit logging
# audit device will be enabled after initialization
log_level = "info"

# Telemetry for monitoring
telemetry {
  prometheus_retention_time = "30s"
  disable_hostname = false
}
```

**Initialize and Unseal Vault**:

```bash
#!/bin/bash
# init-vault.sh - Initialize and unseal Vault

# Start Vault
docker-compose -f docker-compose.vault.yml up -d

# Wait for Vault to be ready
sleep 5

# Set Vault address
export VAULT_ADDR='http://localhost:8200'

# Initialize Vault (only run once!)
vault operator init -key-shares=5 -key-threshold=3 > vault-keys.txt

# IMPORTANT: Store vault-keys.txt securely!
# It contains the unseal keys and root token

# Extract unseal keys
UNSEAL_KEY_1=$(grep 'Unseal Key 1' vault-keys.txt | awk '{print $NF}')
UNSEAL_KEY_2=$(grep 'Unseal Key 2' vault-keys.txt | awk '{print $NF}')
UNSEAL_KEY_3=$(grep 'Unseal Key 3' vault-keys.txt | awk '{print $NF}')
ROOT_TOKEN=$(grep 'Initial Root Token' vault-keys.txt | awk '{print $NF}')

# Unseal Vault (need 3 of 5 keys)
vault operator unseal $UNSEAL_KEY_1
vault operator unseal $UNSEAL_KEY_2
vault operator unseal $UNSEAL_KEY_3

# Login with root token
vault login $ROOT_TOKEN

# Enable audit logging
vault audit enable file file_path=/vault/logs/audit.log

# Enable secrets engines
vault secrets enable -path=secret kv-v2
vault secrets enable -path=database database

echo "Vault initialized and unsealed!"
echo "Root token: $ROOT_TOKEN"
echo "Store vault-keys.txt in a secure location!"
```

**Using Vault in Applications**:

```go
// secrets.go - Vault integration in Go
package secrets

import (
    "fmt"
    "log"

    vault "github.com/hashicorp/vault/api"
)

type VaultClient struct {
    client *vault.Client
}

// NewVaultClient creates a new Vault client
func NewVaultClient(address, token string) (*VaultClient, error) {
    config := vault.DefaultConfig()
    config.Address = address

    client, err := vault.NewClient(config)
    if err != nil {
        return nil, fmt.Errorf("failed to create vault client: %w", err)
    }

    client.SetToken(token)

    return &VaultClient{client: client}, nil
}

// GetSecret retrieves a secret from Vault
func (vc *VaultClient) GetSecret(path string) (map[string]interface{}, error) {
    secret, err := vc.client.Logical().Read(path)
    if err != nil {
        return nil, fmt.Errorf("failed to read secret: %w", err)
    }

    if secret == nil {
        return nil, fmt.Errorf("secret not found at path: %s", path)
    }

    // KV v2 stores data under 'data' key
    data, ok := secret.Data["data"].(map[string]interface{})
    if !ok {
        return secret.Data, nil
    }

    return data, nil
}

// SetSecret stores a secret in Vault
func (vc *VaultClient) SetSecret(path string, data map[string]interface{}) error {
    // KV v2 requires data under 'data' key
    payload := map[string]interface{}{
        "data": data,
    }

    _, err := vc.client.Logical().Write(path, payload)
    if err != nil {
        return fmt.Errorf("failed to write secret: %w", err)
    }

    return nil
}

// GetDatabaseCredentials gets dynamic database credentials
func (vc *VaultClient) GetDatabaseCredentials(role string) (username, password string, err error) {
    path := fmt.Sprintf("database/creds/%s", role)

    secret, err := vc.client.Logical().Read(path)
    if err != nil {
        return "", "", fmt.Errorf("failed to get database credentials: %w", err)
    }

    username = secret.Data["username"].(string)
    password = secret.Data["password"].(string)

    // Set up renewal (credentials are temporary)
    go vc.renewCredentials(secret)

    return username, password, nil
}

// renewCredentials automatically renews credentials before expiry
func (vc *VaultClient) renewCredentials(secret *vault.Secret) {
    watcher, err := vc.client.NewLifetimeWatcher(&vault.LifetimeWatcherInput{
        Secret: secret,
    })
    if err != nil {
        log.Printf("Failed to create lifetime watcher: %v", err)
        return
    }

    go watcher.Start()
    defer watcher.Stop()

    for {
        select {
        case err := <-watcher.DoneCh():
            if err != nil {
                log.Printf("Credential renewal failed: %v", err)
            }
            return

        case renewal := <-watcher.RenewCh():
            log.Printf("Credentials renewed, new lease duration: %d", renewal.Secret.LeaseDuration)
        }
    }
}
```

**Application Integration Example**:

```go
// main.go - Using Vault for database credentials
package main

import (
    "database/sql"
    "log"
    "os"
    "time"

    _ "github.com/lib/pq"
    "myapp/secrets"
)

func main() {
    // Initialize Vault client
    vaultAddr := os.Getenv("VAULT_ADDR")
    vaultToken := os.Getenv("VAULT_TOKEN")

    vaultClient, err := secrets.NewVaultClient(vaultAddr, vaultToken)
    if err != nil {
        log.Fatalf("Failed to create Vault client: %v", err)
    }

    // Get database credentials (dynamic, temporary)
    username, password, err := vaultClient.GetDatabaseCredentials("readonly")
    if err != nil {
        log.Fatalf("Failed to get database credentials: %v", err)
    }

    // Connect to database
    dsn := fmt.Sprintf("postgres://%s:%s@localhost:5432/mydb?sslmode=require",
        username, password)

    db, err := sql.Open("postgres", dsn)
    if err != nil {
        log.Fatalf("Failed to connect to database: %v", err)
    }
    defer db.Close()

    // Use database...
    log.Println("Connected to database with dynamic credentials")

    // Keep application running
    select {}
}
```

### Encrypted Environment Variables

For simpler scenarios, use encrypted environment files:

```bash
#!/bin/bash
# encrypt-secrets.sh - Encrypt environment files with age

# Install age: https://github.com/FiloSottile/age
# apt-get install age

# Generate key pair (do this once)
age-keygen -o key.txt

# Public key will be shown, save it
PUBLIC_KEY=$(grep "public key:" key.txt | awk '{print $3}')

# Encrypt secrets
age -r $PUBLIC_KEY -o .env.encrypted .env

# Decrypt secrets (on production server)
age --decrypt -i key.txt .env.encrypted > .env

# Load secrets
source .env

# Start application
docker-compose up -d
```

**Automated Secret Rotation**:

```bash
#!/bin/bash
# rotate-secrets.sh - Automated secret rotation

VAULT_ADDR="http://localhost:8200"
VAULT_TOKEN="your-token"

# Rotate database password
rotate_db_password() {
    NEW_PASSWORD=$(openssl rand -base64 32)

    # Update in Vault
    vault kv put secret/database password="$NEW_PASSWORD"

    # Update in database
    psql -c "ALTER USER myapp_user WITH PASSWORD '$NEW_PASSWORD';"

    # Restart applications to pick up new password
    docker-compose restart api-service

    echo "Database password rotated successfully"
}

# Rotate API keys
rotate_api_keys() {
    NEW_API_KEY=$(uuidgen)

    # Update in Vault
    vault kv put secret/api-keys service_key="$NEW_API_KEY"

    # Update external service
    curl -X POST https://external-service.com/api/keys \
        -H "Authorization: Bearer $OLD_API_KEY" \
        -d "new_key=$NEW_API_KEY"

    echo "API key rotated successfully"
}

# Run rotation
rotate_db_password
rotate_api_keys

# Log rotation
echo "$(date): Secrets rotated" >> /var/log/secret-rotation.log
```

---

## SSL/TLS: Encrypting Traffic

### Automatic HTTPS with Traefik + Let's Encrypt

Traefik automatically obtains and renews SSL certificates:

```yaml
# docker-compose.traefik.yml
version: '3.8'

services:
  traefik:
    image: traefik:v2.10
    container_name: traefik
    restart: unless-stopped
    security_opt:
      - no-new-privileges:true
    ports:
      - "80:80"
      - "443:443"
    environment:
      - CF_API_EMAIL=${CF_API_EMAIL}
      - CF_API_KEY=${CF_API_KEY}
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ./traefik/traefik.yml:/traefik.yml:ro
      - ./traefik/acme.json:/acme.json
      - ./traefik/config:/config:ro
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.traefik.rule=Host(`traefik.yourdomain.com`)"
      - "traefik.http.routers.traefik.entrypoints=websecure"
      - "traefik.http.routers.traefik.tls.certresolver=letsencrypt"
      - "traefik.http.routers.traefik.service=api@internal"
      - "traefik.http.routers.traefik.middlewares=auth"
      - "traefik.http.middlewares.auth.basicauth.users=${TRAEFIK_DASHBOARD_USER}"
    networks:
      - web

networks:
  web:
    external: true
```

**Traefik Configuration** (`traefik/traefik.yml`):

```yaml
# traefik/traefik.yml

# API and Dashboard
api:
  dashboard: true
  insecure: false

# Entry points
entryPoints:
  web:
    address: ":80"
    # Redirect HTTP to HTTPS
    http:
      redirections:
        entryPoint:
          to: websecure
          scheme: https
          permanent: true

  websecure:
    address: ":443"
    http:
      tls:
        certResolver: letsencrypt
        domains:
          - main: yourdomain.com
            sans:
              - "*.yourdomain.com"

# Providers
providers:
  docker:
    endpoint: "unix:///var/run/docker.sock"
    exposedByDefault: false
    network: web

  file:
    directory: /config
    watch: true

# Let's Encrypt certificate resolver
certificatesResolvers:
  letsencrypt:
    acme:
      email: admin@yourdomain.com
      storage: /acme.json
      # Use DNS challenge for wildcard certificates
      dnsChallenge:
        provider: cloudflare
        delayBeforeCheck: 0
      # Or use HTTP challenge
      # httpChallenge:
      #   entryPoint: web

# Logging
log:
  level: INFO
  filePath: /var/log/traefik/traefik.log

accessLog:
  filePath: /var/log/traefik/access.log
  bufferingSize: 100

# Metrics
metrics:
  prometheus:
    entryPoint: metrics
    addEntryPointsLabels: true
    addRoutersLabels: true
    addServicesLabels: true
```

### Mutual TLS (mTLS) for Service-to-Service Communication

**Generate CA and Service Certificates**:

```bash
#!/bin/bash
# generate-mtls-certs.sh - Generate certificates for mTLS

# Create CA
openssl genrsa -out ca-key.pem 4096
openssl req -new -x509 -days 3650 -key ca-key.pem -sha256 -out ca.pem \
    -subj "/C=US/ST=State/L=City/O=Organization/CN=MyApp CA"

# Generate service certificate
generate_service_cert() {
    SERVICE_NAME=$1

    # Generate private key
    openssl genrsa -out ${SERVICE_NAME}-key.pem 4096

    # Generate CSR
    openssl req -new -key ${SERVICE_NAME}-key.pem -out ${SERVICE_NAME}.csr \
        -subj "/C=US/ST=State/L=City/O=Organization/CN=${SERVICE_NAME}"

    # Sign certificate with CA
    openssl x509 -req -in ${SERVICE_NAME}.csr -CA ca.pem -CAkey ca-key.pem \
        -CAcreateserial -out ${SERVICE_NAME}-cert.pem -days 365 -sha256

    # Clean up CSR
    rm ${SERVICE_NAME}.csr

    echo "Generated certificate for ${SERVICE_NAME}"
}

# Generate certificates for services
generate_service_cert "api-service"
generate_service_cert "database-service"
generate_service_cert "cache-service"

echo "mTLS certificates generated!"
```

**Using mTLS in Go**:

```go
// mtls.go - Mutual TLS implementation
package mtls

import (
    "crypto/tls"
    "crypto/x509"
    "fmt"
    "io/ioutil"
    "net/http"
)

// LoadTLSConfig loads mTLS configuration
func LoadTLSConfig(certFile, keyFile, caFile string) (*tls.Config, error) {
    // Load client certificate
    cert, err := tls.LoadX509KeyPair(certFile, keyFile)
    if err != nil {
        return nil, fmt.Errorf("failed to load certificate: %w", err)
    }

    // Load CA certificate
    caCert, err := ioutil.ReadFile(caFile)
    if err != nil {
        return nil, fmt.Errorf("failed to read CA certificate: %w", err)
    }

    caCertPool := x509.NewCertPool()
    caCertPool.AppendCertsFromPEM(caCert)

    // Create TLS configuration
    tlsConfig := &tls.Config{
        Certificates: []tls.Certificate{cert},
        RootCAs:      caCertPool,
        ClientCAs:    caCertPool,
        ClientAuth:   tls.RequireAndVerifyClientCert,
        MinVersion:   tls.VersionTLS13,
    }

    return tlsConfig, nil
}

// NewMTLSServer creates an HTTPS server with mTLS
func NewMTLSServer(addr string, handler http.Handler, tlsConfig *tls.Config) *http.Server {
    return &http.Server{
        Addr:      addr,
        Handler:   handler,
        TLSConfig: tlsConfig,
    }
}

// NewMTLSClient creates an HTTP client with mTLS
func NewMTLSClient(tlsConfig *tls.Config) *http.Client {
    return &http.Client{
        Transport: &http.Transport{
            TLSClientConfig: tlsConfig,
        },
    }
}
```

**Example Service with mTLS**:

```go
// main.go - API service with mTLS
package main

import (
    "fmt"
    "log"
    "net/http"

    "myapp/mtls"
)

func main() {
    // Load TLS configuration
    tlsConfig, err := mtls.LoadTLSConfig(
        "api-service-cert.pem",
        "api-service-key.pem",
        "ca.pem",
    )
    if err != nil {
        log.Fatalf("Failed to load TLS config: %v", err)
    }

    // Create handler
    mux := http.NewServeMux()
    mux.HandleFunc("/health", func(w http.ResponseWriter, r *http.Request) {
        // Verify client certificate
        if r.TLS == nil || len(r.TLS.PeerCertificates) == 0 {
            http.Error(w, "No client certificate provided", http.StatusUnauthorized)
            return
        }

        clientCert := r.TLS.PeerCertificates[0]
        log.Printf("Request from: %s", clientCert.Subject.CommonName)

        fmt.Fprintf(w, "OK")
    })

    // Create server with mTLS
    server := mtls.NewMTLSServer(":8443", mux, tlsConfig)

    log.Println("Starting API service with mTLS on :8443")
    if err := server.ListenAndServeTLS("", ""); err != nil {
        log.Fatalf("Server failed: %v", err)
    }
}
```

---

## Authentication and Authorization

### OAuth2 and JWT

**JWT Token Generation and Validation**:

```go
// auth/jwt.go - JWT implementation
package auth

import (
    "fmt"
    "time"

    "github.com/golang-jwt/jwt/v5"
)

type Claims struct {
    UserID   string   `json:"user_id"`
    Email    string   `json:"email"`
    Roles    []string `json:"roles"`
    jwt.RegisteredClaims
}

type JWTManager struct {
    secretKey     string
    tokenDuration time.Duration
}

func NewJWTManager(secretKey string, tokenDuration time.Duration) *JWTManager {
    return &JWTManager{
        secretKey:     secretKey,
        tokenDuration: tokenDuration,
    }
}

// Generate creates a new JWT token
func (jm *JWTManager) Generate(userID, email string, roles []string) (string, error) {
    claims := Claims{
        UserID: userID,
        Email:  email,
        Roles:  roles,
        RegisteredClaims: jwt.RegisteredClaims{
            ExpiresAt: jwt.NewNumericDate(time.Now().Add(jm.tokenDuration)),
            IssuedAt:  jwt.NewNumericDate(time.Now()),
            NotBefore: jwt.NewNumericDate(time.Now()),
            Issuer:    "myapp",
            Subject:   userID,
        },
    }

    token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
    tokenString, err := token.SignedString([]byte(jm.secretKey))
    if err != nil {
        return "", fmt.Errorf("failed to sign token: %w", err)
    }

    return tokenString, nil
}

// Validate verifies and parses a JWT token
func (jm *JWTManager) Validate(tokenString string) (*Claims, error) {
    token, err := jwt.ParseWithClaims(
        tokenString,
        &Claims{},
        func(token *jwt.Token) (interface{}, error) {
            if _, ok := token.Method.(*jwt.SigningMethodHMAC); !ok {
                return nil, fmt.Errorf("unexpected signing method: %v", token.Header["alg"])
            }
            return []byte(jm.secretKey), nil
        },
    )

    if err != nil {
        return nil, fmt.Errorf("failed to parse token: %w", err)
    }

    claims, ok := token.Claims.(*Claims)
    if !ok || !token.Valid {
        return nil, fmt.Errorf("invalid token")
    }

    return claims, nil
}

// Refresh creates a new token with extended expiration
func (jm *JWTManager) Refresh(tokenString string) (string, error) {
    claims, err := jm.Validate(tokenString)
    if err != nil {
        return "", err
    }

    // Generate new token with same claims but new expiration
    return jm.Generate(claims.UserID, claims.Email, claims.Roles)
}
```

**Authentication Middleware**:

```go
// auth/middleware.go - JWT authentication middleware
package auth

import (
    "context"
    "net/http"
    "strings"
)

type contextKey string

const ClaimsKey contextKey = "claims"

// AuthMiddleware validates JWT tokens
func (jm *JWTManager) AuthMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // Extract token from Authorization header
        authHeader := r.Header.Get("Authorization")
        if authHeader == "" {
            http.Error(w, "Missing authorization header", http.StatusUnauthorized)
            return
        }

        // Check Bearer scheme
        parts := strings.SplitN(authHeader, " ", 2)
        if len(parts) != 2 || parts[0] != "Bearer" {
            http.Error(w, "Invalid authorization header format", http.StatusUnauthorized)
            return
        }

        tokenString := parts[1]

        // Validate token
        claims, err := jm.Validate(tokenString)
        if err != nil {
            http.Error(w, "Invalid token", http.StatusUnauthorized)
            return
        }

        // Add claims to context
        ctx := context.WithValue(r.Context(), ClaimsKey, claims)

        // Call next handler
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}

// RequireRole checks if user has required role
func RequireRole(requiredRole string) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            claims, ok := r.Context().Value(ClaimsKey).(*Claims)
            if !ok {
                http.Error(w, "Unauthorized", http.StatusUnauthorized)
                return
            }

            // Check if user has required role
            hasRole := false
            for _, role := range claims.Roles {
                if role == requiredRole {
                    hasRole = true
                    break
                }
            }

            if !hasRole {
                http.Error(w, "Forbidden", http.StatusForbidden)
                return
            }

            next.ServeHTTP(w, r)
        })
    }
}

// GetClaims extracts claims from request context
func GetClaims(r *http.Request) (*Claims, bool) {
    claims, ok := r.Context().Value(ClaimsKey).(*Claims)
    return claims, ok
}
```

**Using the Authentication System**:

```go
// main.go - API with JWT authentication
package main

import (
    "encoding/json"
    "log"
    "net/http"
    "time"

    "myapp/auth"
)

func main() {
    // Create JWT manager
    jwtManager := auth.NewJWTManager("your-secret-key", 24*time.Hour)

    mux := http.NewServeMux()

    // Public endpoint: login
    mux.HandleFunc("/login", func(w http.ResponseWriter, r *http.Request) {
        // In production, validate credentials against database
        userID := "user123"
        email := "user@example.com"
        roles := []string{"user", "admin"}

        // Generate token
        token, err := jwtManager.Generate(userID, email, roles)
        if err != nil {
            http.Error(w, "Failed to generate token", http.StatusInternalServerError)
            return
        }

        json.NewEncoder(w).Encode(map[string]string{
            "token": token,
        })
    })

    // Protected endpoints
    api := http.NewServeMux()

    api.HandleFunc("/profile", func(w http.ResponseWriter, r *http.Request) {
        claims, _ := auth.GetClaims(r)
        json.NewEncoder(w).Encode(map[string]interface{}{
            "user_id": claims.UserID,
            "email":   claims.Email,
            "roles":   claims.Roles,
        })
    })

    // Admin-only endpoint
    adminMux := http.NewServeMux()
    adminMux.HandleFunc("/users", func(w http.ResponseWriter, r *http.Request) {
        // Admin functionality
        json.NewEncoder(w).Encode(map[string]string{
            "message": "Admin endpoint",
        })
    })

    // Apply authentication middleware
    api.Handle("/admin/", http.StripPrefix("/admin",
        auth.RequireRole("admin")(adminMux)))

    // Apply authentication to all /api routes
    mux.Handle("/api/", http.StripPrefix("/api",
        jwtManager.AuthMiddleware(api)))

    log.Println("Starting server on :8080")
    http.ListenAndServe(":8080", mux)
}
```

---

## Container Security

### Image Scanning with Trivy

```bash
#!/bin/bash
# scan-images.sh - Scan Docker images for vulnerabilities

# Install Trivy
# wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | apt-key add -
# echo "deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main" | tee -a /etc/apt/sources.list.d/trivy.list
# apt-get update && apt-get install trivy

# Scan image
trivy image --severity HIGH,CRITICAL myapp:latest

# Scan with specific output format
trivy image --format json --output results.json myapp:latest

# Scan with exit code on vulnerabilities
trivy image --exit-code 1 --severity CRITICAL myapp:latest

# Scan filesystem
trivy fs --security-checks vuln,config .

# Scan IaC (Terraform, CloudFormation, etc.)
trivy config --severity HIGH,CRITICAL infrastructure/
```

**Automated Scanning in CI/CD**:

```yaml
# .github/workflows/security-scan.yml
name: Security Scan

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Build Docker image
        run: docker build -t myapp:${{ github.sha }} .

      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: myapp:${{ github.sha }}
          format: 'sarif'
          output: 'trivy-results.sarif'
          severity: 'CRITICAL,HIGH'

      - name: Upload Trivy results to GitHub Security tab
        uses: github/codeql-action/upload-sarif@v2
        with:
          sarif_file: 'trivy-results.sarif'

      - name: Fail on high/critical vulnerabilities
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: myapp:${{ github.sha }}
          exit-code: '1'
          severity: 'CRITICAL,HIGH'
```

### Docker Security Best Practices

**Secure Dockerfile**:

```dockerfile
# Use specific version, not 'latest'
FROM node:18.17-alpine AS builder

# Create non-root user
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodejs -u 1001

# Set working directory
WORKDIR /app

# Copy package files
COPY package*.json ./

# Install dependencies
RUN npm ci --only=production

# Copy application code
COPY --chown=nodejs:nodejs . .

# Build application
RUN npm run build

# Production image
FROM node:18.17-alpine

# Security updates
RUN apk upgrade --no-cache

# Create non-root user
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodejs -u 1001

WORKDIR /app

# Copy built application from builder
COPY --from=builder --chown=nodejs:nodejs /app/dist ./dist
COPY --from=builder --chown=nodejs:nodejs /app/node_modules ./node_modules
COPY --from=builder --chown=nodejs:nodejs /app/package*.json ./

# Switch to non-root user
USER nodejs

# Expose port
EXPOSE 3000

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD node healthcheck.js

# Start application
CMD ["node", "dist/main.js"]
```

**Docker Compose Security**:

```yaml
# docker-compose.secure.yml
version: '3.8'

services:
  api:
    image: myapp:latest
    restart: unless-stopped

    # Run as non-root user
    user: "1001:1001"

    # Read-only root filesystem
    read_only: true

    # Temporary writable directories
    tmpfs:
      - /tmp
      - /var/run

    # Drop all capabilities, add only what's needed
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE

    # Security options
    security_opt:
      - no-new-privileges:true
      - seccomp:unconfined  # or use custom seccomp profile

    # Resource limits
    deploy:
      resources:
        limits:
          cpus: '1.0'
          memory: 512M
        reservations:
          cpus: '0.5'
          memory: 256M

    # Environment variables from secrets
    secrets:
      - db_password
      - api_key

    networks:
      - internal

secrets:
  db_password:
    file: ./secrets/db_password.txt
  api_key:
    file: ./secrets/api_key.txt

networks:
  internal:
    driver: bridge
    internal: true  # No external access
```

---

## Intrusion Detection

### Fail2ban Configuration

```bash
#!/bin/bash
# setup-fail2ban.sh - Configure Fail2ban for intrusion detection

# Install Fail2ban
apt-get update && apt-get install -y fail2ban

# Create local configuration
cat > /etc/fail2ban/jail.local <<EOF
[DEFAULT]
# Ban duration (10 minutes)
bantime = 600

# Time window to count failures (10 minutes)
findtime = 600

# Number of failures before ban
maxretry = 5

# Destination email for notifications
destemail = admin@yourdomain.com

# Action: ban + email notification
action = %(action_mwl)s

[sshd]
enabled = true
port = ssh
logpath = /var/log/auth.log
maxretry = 3
bantime = 3600

[nginx-http-auth]
enabled = true
filter = nginx-http-auth
port = http,https
logpath = /var/log/nginx/error.log
maxretry = 5

[nginx-noscript]
enabled = true
port = http,https
filter = nginx-noscript
logpath = /var/log/nginx/access.log
maxretry = 6

[nginx-badbots]
enabled = true
port = http,https
filter = nginx-badbots
logpath = /var/log/nginx/access.log
maxretry = 2

[nginx-noproxy]
enabled = true
port = http,https
filter = nginx-noproxy
logpath = /var/log/nginx/access.log
maxretry = 2

[traefik-auth]
enabled = true
filter = traefik-auth
port = http,https
logpath = /var/log/traefik/access.log
maxretry = 3
EOF

# Create Traefik filter
cat > /etc/fail2ban/filter.d/traefik-auth.conf <<EOF
[Definition]
failregex = ^<HOST> \- \S+ \[\] \"(GET|POST|HEAD).*\" 401
ignoreregex =
EOF

# Restart Fail2ban
systemctl restart fail2ban

# Check status
fail2ban-client status

# Check specific jail
fail2ban-client status sshd
```

**Monitor Fail2ban**:

```bash
#!/bin/bash
# monitor-fail2ban.sh - Monitor Fail2ban status

# Show all jails and their status
fail2ban-client status

# Show banned IPs for specific jail
fail2ban-client status sshd

# Unban an IP
fail2ban-client set sshd unbanip 192.168.1.100

# Check log
tail -f /var/log/fail2ban.log

# Get statistics
cat /var/log/fail2ban.log | grep 'Ban' | wc -l
```

---

## Security Monitoring and Incident Response

### Centralized Security Logging

```yaml
# docker-compose.security-monitoring.yml
version: '3.8'

services:
  # Centralized log collection
  loki:
    image: grafana/loki:2.9.0
    container_name: loki
    restart: unless-stopped
    ports:
      - "3100:3100"
    volumes:
      - ./loki:/etc/loki
      - loki-data:/loki
    command: -config.file=/etc/loki/loki-config.yml
    networks:
      - monitoring

  # Log aggregation
  promtail:
    image: grafana/promtail:2.9.0
    container_name: promtail
    restart: unless-stopped
    volumes:
      - /var/log:/var/log:ro
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
      - ./promtail:/etc/promtail
    command: -config.file=/etc/promtail/promtail-config.yml
    networks:
      - monitoring

  # Visualization
  grafana:
    image: grafana/grafana:10.1.0
    container_name: grafana
    restart: unless-stopped
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=secure_password
      - GF_USERS_ALLOW_SIGN_UP=false
    volumes:
      - grafana-data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning
    networks:
      - monitoring

volumes:
  loki-data:
  grafana-data:

networks:
  monitoring:
    driver: bridge
```

### Security Alerting

**Grafana Alert Rules**:

```yaml
# grafana/provisioning/alerting/security-alerts.yml
apiVersion: 1

groups:
  - name: security
    interval: 1m
    rules:
      - uid: failed_logins
        title: High Number of Failed Logins
        condition: A
        data:
          - refId: A
            queryType: ''
            relativeTimeRange:
              from: 600
              to: 0
            datasourceUid: loki
            model:
              expr: 'sum(rate({job="auth"} |= "failed login"[5m]))'
        noDataState: NoData
        execErrState: Error
        for: 5m
        annotations:
          description: 'More than 10 failed logins per minute detected'
          summary: 'Potential brute force attack'
        labels:
          severity: critical
        isPaused: false

      - uid: suspicious_ips
        title: Suspicious IP Access
        condition: A
        data:
          - refId: A
            datasourceUid: loki
            model:
              expr: 'count_over_time({job="nginx"} |~ "192.168.1.100"[5m]) > 100'
        for: 5m
        annotations:
          description: 'Known malicious IP accessing system'
        labels:
          severity: high

      - uid: privilege_escalation
        title: Privilege Escalation Attempt
        condition: A
        data:
          - refId: A
            datasourceUid: loki
            model:
              expr: '{job="audit"} |= "sudo" |= "FAILED"'
        for: 1m
        annotations:
          description: 'Failed sudo attempt detected'
        labels:
          severity: high
```

### Incident Response Playbook

```bash
#!/bin/bash
# incident-response.sh - Automated incident response

INCIDENT_LOG="/var/log/security-incidents.log"

# Function to log incidents
log_incident() {
    echo "$(date): $1" >> $INCIDENT_LOG
}

# Function to block IP
block_ip() {
    IP=$1
    REASON=$2

    # Add to UFW
    ufw deny from $IP comment "$REASON"

    # Add to Fail2ban
    fail2ban-client set sshd banip $IP

    # Log incident
    log_incident "Blocked IP: $IP - Reason: $REASON"

    # Send alert
    curl -X POST https://alerts.company.com/api/incidents \
        -H "Content-Type: application/json" \
        -d "{\"ip\": \"$IP\", \"reason\": \"$REASON\", \"timestamp\": \"$(date -Iseconds)\"}"
}

# Function to isolate compromised container
isolate_container() {
    CONTAINER=$1

    # Disconnect from networks
    docker network disconnect web $CONTAINER
    docker network disconnect internal $CONTAINER

    # Take snapshot for forensics
    docker commit $CONTAINER compromised-$CONTAINER-$(date +%Y%m%d-%H%M%S)

    # Stop container
    docker stop $CONTAINER

    log_incident "Isolated container: $CONTAINER"
}

# Function to rotate compromised credentials
rotate_credentials() {
    SERVICE=$1

    # Generate new password
    NEW_PASSWORD=$(openssl rand -base64 32)

    # Update in Vault
    vault kv put secret/$SERVICE password="$NEW_PASSWORD"

    # Restart service
    docker-compose restart $SERVICE

    log_incident "Rotated credentials for: $SERVICE"
}

# Monitor for incidents
monitor_security() {
    # Check for suspicious activity
    FAILED_LOGINS=$(grep "Failed password" /var/log/auth.log | tail -100 | wc -l)

    if [ $FAILED_LOGINS -gt 10 ]; then
        log_incident "High number of failed logins: $FAILED_LOGINS"

        # Extract and block offending IPs
        grep "Failed password" /var/log/auth.log | \
            awk '{print $(NF-3)}' | \
            sort | uniq -c | \
            while read count ip; do
                if [ $count -gt 5 ]; then
                    block_ip $ip "Brute force attack: $count failed attempts"
                fi
            done
    fi
}

# Run monitoring
monitor_security
```

---

## Compliance and Security Hardening

### CIS Benchmarks Implementation

```bash
#!/bin/bash
# cis-hardening.sh - Implement CIS security benchmarks

# 1. Filesystem hardening
echo "Hardening filesystem..."

# Disable unused filesystems
cat >> /etc/modprobe.d/cis.conf <<EOF
install cramfs /bin/true
install freevxfs /bin/true
install jffs2 /bin/true
install hfs /bin/true
install hfsplus /bin/true
install udf /bin/true
EOF

# Set secure mount options
cat >> /etc/fstab <<EOF
tmpfs /tmp tmpfs defaults,noexec,nosuid,nodev 0 0
tmpfs /dev/shm tmpfs defaults,noexec,nosuid,nodev 0 0
EOF

# 2. Kernel hardening
echo "Hardening kernel..."

cat >> /etc/sysctl.d/99-cis.conf <<EOF
# Network security
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.default.send_redirects = 0
net.ipv4.conf.all.accept_source_route = 0
net.ipv4.conf.default.accept_source_route = 0
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.default.accept_redirects = 0
net.ipv4.conf.all.secure_redirects = 0
net.ipv4.conf.default.secure_redirects = 0
net.ipv4.conf.all.log_martians = 1
net.ipv4.conf.default.log_martians = 1
net.ipv4.icmp_echo_ignore_broadcasts = 1
net.ipv4.icmp_ignore_bogus_error_responses = 1
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.rp_filter = 1
net.ipv4.tcp_syncookies = 1

# IPv6 security
net.ipv6.conf.all.accept_source_route = 0
net.ipv6.conf.default.accept_source_route = 0
net.ipv6.conf.all.accept_redirects = 0
net.ipv6.conf.default.accept_redirects = 0

# Kernel hardening
kernel.randomize_va_space = 2
kernel.dmesg_restrict = 1
kernel.kptr_restrict = 2
kernel.yama.ptrace_scope = 1
EOF

sysctl -p /etc/sysctl.d/99-cis.conf

# 3. SSH hardening
echo "Hardening SSH..."

cat >> /etc/ssh/sshd_config <<EOF
# CIS SSH Configuration
Protocol 2
LogLevel VERBOSE
PermitRootLogin no
MaxAuthTries 3
MaxSessions 2
IgnoreRhosts yes
HostbasedAuthentication no
PermitEmptyPasswords no
PermitUserEnvironment no
ClientAliveInterval 300
ClientAliveCountMax 0
LoginGraceTime 60
Banner /etc/issue.net
AllowUsers deploy admin
Ciphers chacha20-poly1305@openssh.com,aes256-gcm@openssh.com
MACs hmac-sha2-512-etm@openssh.com,hmac-sha2-256-etm@openssh.com
KexAlgorithms curve25519-sha256,curve25519-sha256@libssh.org
EOF

systemctl restart sshd

# 4. Audit logging
echo "Configuring audit logging..."

apt-get install -y auditd

cat > /etc/audit/rules.d/cis.rules <<EOF
# Monitor for suspicious activity
-w /etc/passwd -p wa -k passwd_changes
-w /etc/group -p wa -k group_changes
-w /etc/shadow -p wa -k shadow_changes
-w /etc/sudoers -p wa -k sudoers_changes
-w /var/log/auth.log -p wa -k auth_log_changes
-w /bin/su -p x -k su_execution
-w /usr/bin/sudo -p x -k sudo_execution
-w /etc/ssh/sshd_config -p wa -k sshd_config_changes
EOF

systemctl restart auditd

echo "Security hardening complete!"
```

### Automated Security Auditing

```bash
#!/bin/bash
# security-audit.sh - Automated security audit

REPORT_FILE="/var/log/security-audit-$(date +%Y%m%d).log"

echo "Security Audit Report - $(date)" > $REPORT_FILE
echo "=====================================" >> $REPORT_FILE

# Check for updates
echo -e "\n1. System Updates:" >> $REPORT_FILE
apt list --upgradable 2>/dev/null >> $REPORT_FILE

# Check open ports
echo -e "\n2. Open Ports:" >> $REPORT_FILE
ss -tuln >> $REPORT_FILE

# Check running services
echo -e "\n3. Running Services:" >> $REPORT_FILE
systemctl list-units --type=service --state=running >> $REPORT_FILE

# Check failed login attempts
echo -e "\n4. Failed Login Attempts (last 24h):" >> $REPORT_FILE
grep "Failed password" /var/log/auth.log | \
    grep "$(date +%b\ %d)" | wc -l >> $REPORT_FILE

# Check firewall status
echo -e "\n5. Firewall Status:" >> $REPORT_FILE
ufw status verbose >> $REPORT_FILE

# Check Docker containers
echo -e "\n6. Docker Containers:" >> $REPORT_FILE
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}" >> $REPORT_FILE

# Check SSL certificate expiry
echo -e "\n7. SSL Certificates:" >> $REPORT_FILE
for domain in yourdomain.com api.yourdomain.com; do
    expiry=$(echo | openssl s_client -servername $domain -connect $domain:443 2>/dev/null | \
        openssl x509 -noout -enddate | cut -d= -f2)
    echo "$domain expires: $expiry" >> $REPORT_FILE
done

# Check for world-writable files
echo -e "\n8. World-Writable Files:" >> $REPORT_FILE
find / -type f -perm -002 2>/dev/null | head -20 >> $REPORT_FILE

# Check for SUID files
echo -e "\n9. SUID Files:" >> $REPORT_FILE
find / -perm -4000 2>/dev/null >> $REPORT_FILE

# Check user accounts
echo -e "\n10. User Accounts:" >> $REPORT_FILE
awk -F: '($3 >= 1000) {print $1,$3,$6}' /etc/passwd >> $REPORT_FILE

echo "Audit complete. Report saved to: $REPORT_FILE"

# Send report via email
mail -s "Security Audit Report" admin@yourdomain.com < $REPORT_FILE
```

---

## Summary: Building a Secure Infrastructure Organism

Security is not a feature to be added—it's a fundamental attribute of healthy infrastructure. Like a biological immune system that operates continuously and adapts to new threats, infrastructure security requires:

**Defense in Depth**:
- Multiple security layers (network, application, data)
- Assume breach mentality
- Limit blast radius
- Continuous monitoring

**Automated Protection**:
- Firewall rules and network segmentation
- Secrets management and rotation
- Certificate automation
- Vulnerability scanning
- Intrusion detection

**Strong Authentication**:
- JWT tokens with proper validation
- Mutual TLS for service-to-service
- OAuth2 for user authentication
- Role-based access control

**Continuous Monitoring**:
- Centralized security logging
- Real-time alerting
- Incident response automation
- Regular security audits

**Compliance and Hardening**:
- CIS benchmarks implementation
- Security best practices
- Regular updates and patches
- Audit trail maintenance

The next chapter explores the digestive system—data processing and transformation pipelines that ingest, process, and extract value from raw data.

---

## Further Reading

- **OWASP Top 10**: https://owasp.org/www-project-top-ten/
- **CIS Benchmarks**: https://www.cisecurity.org/cis-benchmarks/
- **NIST Cybersecurity Framework**: https://www.nist.gov/cyberframework
- **HashiCorp Vault Documentation**: https://developer.hashicorp.com/vault
- **Let's Encrypt Documentation**: https://letsencrypt.org/docs/
- **Docker Security Best Practices**: https://docs.docker.com/engine/security/
- **Kubernetes Security**: https://kubernetes.io/docs/concepts/security/
