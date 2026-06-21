# Fildah VPS Deployment Guide (Beginner Friendly)

This folder contains the configuration needed to run your entire stack (`fildah-api`, `fildah-web`, and `rxchat-web`) on your own Virtual Private Server (VPS) instead of using Vercel.

## 1. Prerequisites (Before touching the VPS)

Before we start the setup, you need to link your domain names to your VPS. 
Go to where you bought your domains (like GoDaddy, Namecheap, Cloudflare) and add **A Records** that point to your VPS's public IP address for:
- `www.fildah.com`
- `fildah.com`
- `rxchat.fildah.com`
- `api.fildah.com`

## 2. Connecting to Your VPS

You need to log into your VPS using SSH (Secure Shell). 
Open your terminal and type:
```bash
ssh username@your_vps_ip_address
```
*(Note: If you use the Antigravity IDE, the AI assistant can do this for you! Just provide the SSH command and password/key).*

## 3. Installing Docker on the VPS

Your VPS needs Docker to run the containers. Run these commands on the VPS to install Docker:
```bash
# Download the Docker installation script
curl -fsSL https://get.docker.com -o get-docker.sh
# Run the script
sudo sh get-docker.sh
# Allow your user to run docker commands without typing 'sudo' every time
sudo usermod -aG docker $USER
```
*(After running the last command, log out of the VPS and log back in for it to take effect).*

## 4. Getting Your Code onto the VPS

Create a folder on your VPS to hold your code:
```bash
sudo groupadd -f fildah
sudo mkdir -p /srv/fildah
sudo chown -R root:fildah /srv/fildah
sudo chmod -R g+rwX /srv/fildah
sudo find /srv/fildah -type d -exec chmod g+s {} \;
cd /srv/fildah
```

For every VPS user who should help deploy this app, add them to both groups:
```bash
sudo usermod -aG fildah,docker username
```
Then log out and back in for the new group access to apply.

Next, you need to get your code into this folder. You can clone your repositories using Git. Make sure they are named exactly like this:
```bash
git clone https://github.com/fildahs/fildah-api.git
git clone https://github.com/fildahs/fildah-web.git
git clone https://github.com/fildahs/rxchat-web.git
```

Since the `deploy` folder is currently untracked locally, you will also need to copy it from your local machine to the VPS using `scp`. 
**Run this from your local computer:**
```bash
scp -r ~/dev/deploy username@your_vps_ip_address:/srv/fildah/deploy
```

## 5. Setting up Environment Variables

Go into the deploy folder on the VPS:
```bash
cd /srv/fildah/deploy
```
Copy the example environment file to create your real one:
```bash
cp .env.example .env
```
Open the `.env` file using a text editor like `nano`:
```bash
nano .env
```
Fill in the real passwords, API keys, and settings. Save and exit (in nano, press `Ctrl+O`, `Enter`, then `Ctrl+X`).

## 6. Starting With a Fresh VPS Database

This deployment starts with a new empty Postgres database on the VPS.
Start the database containers:
```bash
docker compose -f compose.yml up -d postgres qdrant
```

Run Django migrations to create the tables:
```bash
docker compose -f compose.yml run --rm api python manage.py migrate
```

Create your first admin user:
```bash
docker compose -f compose.yml run --rm api python manage.py createsuperuser
```

Collect Django static files for the admin:
```bash
docker compose -f compose.yml run --rm api python manage.py collectstatic --noinput
```

## 7. Starting With an Empty Qdrant

This deployment starts with an empty self-hosted Qdrant instance. The app uses the collection name:
```bash
RxChat
```

To check the current collections:
```bash
docker compose -f compose.yml run --rm api python manage.py shell -c "from rxchat.qdrant_service import _get_client; print([x.name for x in _get_client().get_collections().collections])"
```

## 8. Start the Entire Application

Now that data is restored, start everything up!
```bash
docker compose -f compose.yml up -d --build
```

## 9. Smoke Tests
Visit your domains in the browser to make sure everything works:
- `https://api.fildah.com/health/` returns OK.
- `https://www.fildah.com` loads.
- `https://rxchat.fildah.com` loads and can call the API.

## Useful Commands (Cheat Sheet)
- **View logs (to see errors):** `docker compose -f compose.yml logs -f`
- **Restart the whole stack:** `docker compose -f compose.yml restart`
- **Stop everything:** `docker compose -f compose.yml down`


## CI/CD Cleanup Policy

The production deploy workflow keeps deployment storage small automatically:

- VPS backups are stored under `/srv/fildah/backups` and only the newest 3 `deploy-*` backup folders are kept.
- GHCR image cleanup keeps the newest 3 package versions for each deployed app image.
- The cleanup also protects the `main` tag and the exact image tag being deployed so production is not left pointing at a deleted image.

For GHCR cleanup, add `GHCR_CLEANUP_TOKEN` to the deploy repo's `production` environment secrets. It should be a GitHub personal access token that can read and delete package versions.
