# Harbor Docker Registry Implementation Guide

This document provides a step-by-step guide to implement **Harbor**, an open-source Docker Registry, on a Linux server. It includes setup, configuration, integration with a Jenkins server, and troubleshooting steps for common issues like `connection refused` errors when connecting to the registry.

## Prerequisites
- **Server**: A Linux server (e.g., Ubuntu 20.04/22.04) with:
  - 4GB RAM, 2 CPU cores, 40GB disk space.
  - IP address (e.g., `192.168.18.153` for Harbor server).
- **Software**:
  - Docker (version 20.10 or later).
  - Docker Compose (version 2.x or later).
- **Network**:
  - Ports 80 (HTTP) and/or 443 (HTTPS) open on the Harbor server.
  - Optional: Domain name (e.g., `harbor.example.com`) with DNS configured.
- **SSL Certificates**: Required for HTTPS in production (self-signed for testing).
- **Access**: Root or sudo privileges on both Harbor and Jenkins servers.
- **Jenkins Server**: A separate or same server with Docker installed for pushing/pulling images.

## Implementation Steps

### 1. Install Docker and Docker Compose
On both the Harbor and Jenkins servers, ensure Docker and Docker Compose are installed.

```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install Docker
sudo apt install -y docker.io
sudo systemctl start docker
sudo systemctl enable docker

# Install Docker Compose
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
docker-compose --version
```

### 2. Download Harbor
Download the Harbor offline installer from the [official GitHub releases page](https://github.com/goharbor/harbor/releases).

On the Harbor server (`192.168.18.153`):
```bash
# Download latest version (e.g., 2.11.0)
wget https://github.com/goharbor/harbor/releases/download/v2.11.0/harbor-offline-installer-v2.11.0.tgz
tar xzvf harbor-offline-installer-v2.11.0.tgz
cd harbor
```

### 3. Configure Harbor
Edit the Harbor configuration file to set up the registry.

1. **Copy Template**:
   ```bash
   cp harbor.yml.tmpl harbor.yml
   nano harbor.yml
   ```

2. **Update Key Settings**:
   Example configuration for HTTP (testing):
   ```yaml
   hostname: 192.168.18.153
   http:
     port: 80
   harbor_admin_password: Harbor12345
   database:
     password: root123
     max_idle_conns: 50
     max_open_conns: 100
   data_volume: /data/harbor
   ```

   For HTTPS (production):
   - Generate or obtain SSL certificates:
     ```bash
     mkdir -p /etc/ssl/harbor
     openssl req -x509 -newkey rsa:4096 -nodes -sha256 -keyout /etc/ssl/harbor/cert.key -out /etc/ssl/harbor/cert.crt -days 365
     ```
   - Update `harbor.yml`:
     ```yaml
     hostname: 192.168.18.153
     https:
       port: 443
       certificate: /etc/ssl/harbor/cert.crt
       private_key: /etc/ssl/harbor/cert.key
     harbor_admin_password: Harbor12345
     database:
       password: root123
       max_idle_conns: 50
       max_open_conns: 100
     data_volume: /data/harbor
     ```

3. **Create Data Directory**:
   ```bash
   sudo mkdir -p /data/harbor
   ```

### 4. Install and Start Harbor
Run the preparation and installation scripts:
```bash
./prepare
./install.sh
```

Verify containers are running:
```bash
docker-compose ps
```
Expected output: Containers like `harbor-core`, `harbor-portal`, `harbor-db`, and `nginx` in `Up` state.

### 5. Configure Firewall
Ensure ports 80 and/or 443 are open on the Harbor server:
```bash
sudo ufw allow 80
sudo ufw allow 443
sudo ufw status
```

### 6. Configure Docker on Jenkins Server
On the Jenkins server, configure Docker to trust the Harbor registry.

1. **Edit `/etc/docker/daemon.json`**:
   ```bash
   sudo nano /etc/docker/daemon.json
   ```
   Add `insecure-registries` for HTTP:
   ```json
   {
     "dns": ["8.8.8.8", "8.8.4.4"],
     "insecure-registries": ["http://192.168.18.153:80"]
   }
   ```

2. **For HTTPS**:
   Copy the Harbor certificate to the Jenkins server:
   ```bash
   sudo mkdir -p /etc/docker/certs.d/192.168.18.153
   scp <harbor-user>@192.168.18.153:/etc/ssl/harbor/cert.crt /etc/docker/certs.d/192.168.18.153/ca.crt
   ```

3. **Restart Docker**:
   ```bash
   sudo systemctl restart docker
   ```

4. **Add Jenkins User to Docker Group**:
   ```bash
   sudo usermod -aG docker jenkins
   ```
   Log out and back in as the `jenkins` user.

### 7. Test Harbor Access
1. **Web UI**:
   - Open `http://192.168.18.153` or `https://192.168.18.153` in a browser.
   - Log in with `admin` and `Harbor12345`.

2. **Docker Login**:
   On the Jenkins server, log in to Harbor:
   ```bash
   echo "Harbor12345" | docker login http://192.168.18.153:80 --username admin --password-stdin
   ```

3. **Create a Project**:
   In the Harbor UI, create a project named `myapp`.

### 8. Push and Pull Images
Test the registry by pushing an image:
```bash
# Pull a test image
docker pull hello-world

# Tag the image
docker tag hello-world 192.168.18.153/myapp/hello-world:latest

# Push the image
docker push 192.168.18.153/myapp/hello-world:latest
```

Verify in the Harbor UI under **Projects > myapp**.

### 9. Enable Vulnerability Scanning
- In the Harbor UI, go to **Configuration > Vulnerability**.
- Enable **Scan on Push**.
- Scan images manually: **Projects > myapp > Repositories > hello-world:latest > Scan**.

### 10. Jenkins Integration
To push images from a Jenkins pipeline:
1. Install the Docker plugin in Jenkins.
2. Add Harbor credentials in **Manage Jenkins > Manage Credentials**.
3. Create a pipeline:
   ```groovy
   pipeline {
       agent any
       stages {
           stage('Push to Harbor') {
               steps {
                   script {
                       docker.withRegistry('http://192.168.18.153', 'harbor-credentials') {
                           def image = docker.build("192.168.18.153/myapp/my-image:latest")
                           image.push()
                       }
                   }
               }
           }
       }
   }
   ```

### 11. Management
- **Start/Stop Harbor**:
  ```bash
  cd ~/harbor
  docker-compose down
  docker-compose up -d
  ```
- **View Logs**:
  ```bash
  docker-compose logs
  ```
- **Backup**:
  ```bash
  sudo tar -czvf harbor-backup-$(date +%F).tar.gz /data/harbor
  ```

## Troubleshooting
### Issue: `connection refused` on `192.168.18.153:80` or `:443`
- **Check Harbor Status**:
  ```bash
  cd ~/harbor
  docker-compose ps
  ```
  Start Harbor if stopped: `docker-compose up -d`.
- **Verify Ports**:
  ```bash
  sudo netstat -tuln | grep -E ':80|:443'
  ```
- **Test Connectivity**:
  From the Jenkins server:
  ```bash
  ping 192.168.18.153
  telnet 192.168.18.153 80
  curl -v http://192.168.18.153:80/v2/
  ```
- **Firewall**:
  Ensure ports are open on the Harbor server:
  ```bash
  sudo ufw allow 80
  sudo ufw allow 443
  ```
  Allow outgoing connections from Jenkins:
  ```bash
  sudo ufw allow out to 192.168.18.153 port 80 proto tcp
  ```
- **Check `harbor.yml`**:
  Verify `hostname` and `http.port` or `https` settings.
- **HTTPS Issues**:
  Ensure certificates are valid and copied to `/etc/docker/certs.d/192.168.18.153/ca.crt` on the Jenkins server.

### Other Issues
- **Login Fails**: Verify the admin password in `harbor.yml`.
- **Push Errors**: Ensure the `myapp` project exists and the `admin` user has push permissions.
- **Docker Permissions**: Ensure the `jenkins` user is in the `docker` group.

## Production Recommendations
- Use HTTPS with valid SSL certificates (e.g., Let’s Encrypt).
- Configure external storage (e.g., S3) in `harbor.yml`.
- Integrate LDAP/AD or OIDC for authentication.
- Set up replication for high availability.

## References
- [Harbor Official Documentation](https://goharbor.io/docs/)
- [Harbor GitHub](https://github.com/goharbor/harbor)
