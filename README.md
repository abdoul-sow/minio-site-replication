# MinIO Multi-Site Replication with Traefik

## Overview
This project provides a production-ready deployment for multi-site MinIO replication with automatic SSL and reverse proxy configuration using Traefik.

## Key Features
- Docker Swarm and Docker Compose orchestration support
- Active-Active bidirectional replication between multiple sites
- Automated SSL with Let's Encrypt or custom certificates
- Traefik reverse proxy with automatic service discovery
- Immutable infrastructure with version-controlled configuration

## Prerequisites

```bash
# Install Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sh get-docker.sh
sudo usermod -aG docker $USER

# Install Docker Compose
sudo curl -SL https://github.com/docker/compose/releases/download/v2.29.1/docker-compose-linux-x86_64 -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose

# Initialize Docker Swarm (only if using Swarm mode)
docker swarm init --advertise-addr $(hostname -i | awk '{print $1}')
```

## Deployment Instructions

### Site A Configuration

1. **Clone the repository**
   ```bash
   git clone https://github.com/abdoul-sow/minio-site-replication.git
   cd minio-site-replication/deploy/site-a
   ```

2. **Create required directories**
   
   **Option 1: Using your own SSL certificates**
   ```bash
   mkdir -p /mnt/data/minio-site-a/{minio-data,traefik/{certificates,config}}
   
   # Copy your SSL certificates
   cp your-cert.crt /mnt/data/minio-site-a/traefik/certificates/sample.crt
   cp your-cert.key /mnt/data/minio-site-a/traefik/certificates/sample.key
   ```
   
   Create or modify the `certificates.yaml` file in `/mnt/data/minio-site-a/traefik/config/`:
   ```yaml
   tls:
     options:
       default:
         minVersion: VersionTLS12
       mintls13:
         minVersion: VersionTLS13
   
     certificates:
       - certFile: "/certificates/sample.crt"
         keyFile: "/certificates/sample.key"
   ```
   
   **Option 2: Using Let's Encrypt**
   ```bash
   mkdir -p /mnt/data/minio-site-a/traefik/letsencrypt
   touch /mnt/data/minio-site-a/traefik/letsencrypt/acme.json
   chmod 600 /mnt/data/minio-site-a/traefik/letsencrypt/acme.json
   ```

3. **Configure Traefik SSL settings**
   
   Modify your `docker-compose.withssl.yaml` based on your deployment method:
   
   ```yaml
   command:
     # Set according to your deployment method
     - "--providers.docker.swarmMode=true"  # For Swarm
     # OR
     - "--providers.docker.swarmMode=false" # For Compose
   
   # If using custom certificates, keep these lines:
   volumes:
     - ${VOLUME_PATH:?err}/traefik/certificates:/certificates
     - ${VOLUME_PATH:?err}/traefik/config/certificates.yaml:/config/certificates.yaml
   
   # If using Let's Encrypt, comment out the certificate lines above and uncomment:
   labels:
     - "traefik.http.routers.${PROJECT_NAME:-minio}-api.tls.certResolver=letsencrypt"
   ```

4. **Configure environment variables**
   ```bash
   cp .env.sample .env
   nano .env  # Update with your values
   ```

5. **Deploy Site A**
   
   For Docker Swarm:
   ```bash
   docker stack deploy -c docker-compose.yml -c docker-compose.ssl.yml minio-site-a
   ```
   
   For Docker Compose:
   ```bash
   docker-compose -f docker-compose.yml -f docker-compose.withssl.yaml up -d --pull=always --renew-anon-volumes
   ```

### Site B Configuration

1. **Clone the repository (if not already done)**
   ```bash
   git clone https://github.com/abdoul-sow/minio-site-replication.git
   cd minio-site-replication/deploy/site-b
   ```

2. **Create required directories**
   
   **Option 1: Using your own SSL certificates**
   ```bash
   mkdir -p /mnt/data/minio-site-b/{minio-data,traefik/{certificates,config}}
   
   # Copy your SSL certificates
   cp your-cert.crt /mnt/data/minio-site-b/traefik/certificates/sample.crt
   cp your-cert.key /mnt/data/minio-site-b/traefik/certificates/sample.key
   ```
   
   Create or modify the `certificates.yaml` file in `/mnt/data/minio-site-b/traefik/config/`:
   ```yaml
   tls:
     options:
       default:
         minVersion: VersionTLS12
       mintls13:
         minVersion: VersionTLS13
   
     certificates:
       - certFile: "/certificates/sample.crt"
         keyFile: "/certificates/sample.key"
   ```
   
   **Option 2: Using Let's Encrypt**
   ```bash
   mkdir -p /mnt/data/minio-site-b/traefik/letsencrypt
   touch /mnt/data/minio-site-b/traefik/letsencrypt/acme.json
   chmod 600 /mnt/data/minio-site-b/traefik/letsencrypt/acme.json
   ```

3. **Configure Traefik SSL settings**
   
   Modify your `docker-compose.withssl.yaml` using the same approach as Site A.

4. **Configure environment variables**
   ```bash
   cp .env.sample .env
   nano .env  # Update with your values (unique to Site B)
   ```

5. **Deploy Site B**
   
   For Docker Swarm:
   ```bash
   docker stack deploy -c docker-compose.yml -c docker-compose.ssl.yml minio-site-b
   ```
   
   For Docker Compose:
   ```bash
   docker-compose -f docker-compose.yml -f docker-compose.withssl.yaml up -d --pull=always --renew-anon-volumes
   ```

## Setting Up Site Replication

1. **Access Site A console**
   ```
   https://console.s3.site-a.sample.net
   ```

2. **Configure replication**
   - Navigate to Site Replication → Add Site
   - Enter Site B details:
     ```
     Endpoint:   https://s3.site-b.sample.net
     Access Key: <ROOT_USER from Site B .env>
     Secret Key: <ROOT_PASSWORD from Site B .env>
     ```

## Verifying Replication

1. Create a test bucket on Site A
2. Create a different test bucket on Site B
3. Verify that both buckets appear on both sites after replication completes

## Troubleshooting

If replication is not working:
- Check network connectivity between sites
- Verify SSL certificates are valid
- Ensure ROOT_USER and ROOT_PASSWORD match in both .env files
- Check MinIO logs for detailed error messages
