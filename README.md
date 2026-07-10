# SwimCoach Infra

Docker Compose stack for the SwimCoach production Droplet. Clone this repo to `/infra` on the Droplet.

## One-time Droplet setup

These steps are not automated and must be repeated if the Droplet is ever rebuilt.

### 1. Install Docker
```bash
curl -fsSL https://get.docker.com | sh
```

### 2. Create deploy user
```bash
useradd -m -s /bin/bash deploy
usermod -aG docker deploy
mkdir -p /home/deploy/.ssh
cat <swimcoach_deploy.pub contents> >> /home/deploy/.ssh/authorized_keys
chmod 700 /home/deploy/.ssh
chmod 600 /home/deploy/.ssh/authorized_keys
chown -R deploy:deploy /home/deploy/.ssh
```

### 3. Clone this repo
```bash
git clone https://github.com/AFresnedo/swim-coach-infra.git /infra
chown -R deploy:deploy /infra
```

### 4. Create .env
Copy `.env.example` to `.env` and fill in all values.

### 5. Start postgres
```bash
docker compose -f /infra/docker-compose.yml up -d postgres
```

### 6. Enable pgvector
```bash
docker compose -f /infra/docker-compose.yml exec postgres psql -U postgres -d swimcoach -c "CREATE EXTENSION IF NOT EXISTS vector;"
```

### 7. Start redis
```bash
docker compose -f /infra/docker-compose.yml up -d redis
```
The CD pipeline's deploy step runs `docker compose up -d --no-deps backend`, which never
starts `redis` itself - it must already be running and healthy or `backend` will never
satisfy its `depends_on: condition: service_healthy` and fail to start.

### 8. Enable unattended security upgrades with auto-reboot
Edit `/etc/apt/apt.conf.d/50unattended-upgrades` and uncomment:
```
Unattended-Upgrade::Automatic-Reboot "true";
Unattended-Upgrade::Automatic-Reboot-Time "03:00";
```

### 9. Start Caddy
```bash
docker compose -f /infra/docker-compose.yml up -d --no-deps caddy
```

The backend is deployed automatically by the CD pipeline after this setup is complete.
