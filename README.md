# 🚀 Rails Deploy Init

**One command to deploy any Rails app to AWS EC2.**

Zero manual steps. Just answer a few prompts and your Rails app is live.

```bash
cd my-rails-app
rails-deploy-init
```

## Features

- ✅ **Fully automated** - No manual SSH, no copying files, no editing configs
- ✅ **Auto-detection** - App name, database, Ruby version, Git repo
- ✅ **SSH key management** - Your local keys are automatically copied to server
- ✅ **Bundler 2.x compatible** - Works with modern Ruby/Bundler
- ✅ **Production-ready** - Nginx, Puma, PostgreSQL, systemd
- ✅ **One command deploys** - `bin/deploy` for subsequent deploys

## Requirements

### Local Machine
- macOS or Linux
- Ruby with Bundler
- Git
- SSH key for GitHub (`~/.ssh/id_ed25519_github` or `~/.ssh/id_ed25519`)

### AWS EC2
- Ubuntu 22.04+ instance
- SSH access via `.pem` key
- Ports open: 22 (SSH), 80 (HTTP), 443 (HTTPS)

## Installation

```bash
# Download the script
curl -o rails-deploy-init https://raw.githubusercontent.com/AhelFliz/rails-ec2-deploy/main/rails-deploy-init

# Make it executable and move to PATH
chmod +x rails-deploy-init
sudo mv rails-deploy-init /usr/local/bin/
```

Or with wget:
```bash
sudo wget -O /usr/local/bin/rails-deploy-init https://raw.githubusercontent.com/AhelFliz/rails-ec2-deploy/main/rails-deploy-init
sudo chmod +x /usr/local/bin/rails-deploy-init
```

## Usage

### First Deployment

```bash
cd your-rails-app
rails-deploy-init
```

You'll be prompted for:

| Prompt | Example | Notes |
|--------|---------|-------|
| AWS EC2 IP | `54.123.45.67` | Your instance's public IP |
| Domain | `myapp.com` | Or press Enter to use IP |
| Path to .pem | `~/.ssh/my-key.pem` | AWS key pair file |
| Rails password | (Enter for default) | Password for `rails` user |
| Sudo password | (Enter for same) | Usually same as rails password |

The script will:
1. Generate all deployment configs
2. Copy files to server
3. Install Ruby, PostgreSQL, Nginx
4. Configure Puma as systemd service
5. Deploy your app
6. Start everything

### Subsequent Deploys

```bash
bin/deploy           # Deploy latest code
bin/deploy logs      # Tail production logs
bin/deploy console   # Rails console on server
bin/deploy restart   # Restart Puma
bin/deploy status    # Check Puma status
```

## What Gets Installed

### On Server
- **Ruby** (version from your `.ruby-version`)
- **PostgreSQL** (with app database)
- **Nginx** (reverse proxy)
- **Puma** (app server via systemd)
- **rbenv** (Ruby version manager)
- **Node.js + Yarn** (for asset compilation)

### Generated Files

```
your-app/
├── config/
│   ├── deploy.rb          # Mina deployment config
│   └── puma.rb            # Puma production config
├── deployment/
│   ├── nginx/your_app     # Nginx site config
│   ├── systemd/puma-*.service
│   ├── config/database.yml
│   └── server_setup.sh    # Server bootstrap
└── bin/
    └── deploy             # Deploy helper script
```

## Configuration

### Auto-Detected Values

| Value | Source |
|-------|--------|
| App name | Directory name (snake_case) |
| Database | `{app_name}_production` |
| Ruby version | `.ruby-version` file |
| Git repo | `.git/config` (converted to SSH) |

### Server Paths

| What | Where |
|------|-------|
| App root | `/home/rails/{app_name}` |
| Current release | `/home/rails/{app_name}/current` |
| Shared files | `/home/rails/{app_name}/shared` |
| Nginx config | `/etc/nginx/sites-available/{app_name}` |
| Puma service | `/etc/systemd/system/puma-{app_name}.service` |
| Logs | `/home/rails/{app_name}/shared/log/` |

### Default Credentials

| What | Default |
|------|---------|
| Server user | `rails` |
| Password | `ZvpII4PiQF7L3X9JYjwqoz` (customizable) |
| DB user | `rails` |
| DB password | Same as server password |

## SSH Keys

The script handles two types of SSH keys automatically:

### 1. Server Access Key
Your local SSH public key (`~/.ssh/id_ed25519.pub`) is copied to the server so you can:
```bash
ssh rails@YOUR_IP
```

### 2. GitHub Key (Required Setup)

The script copies your GitHub SSH key to the server so it can clone your private repos.

**The script looks for these files (in order):**
```
~/.ssh/id_ed25519_github      # Preferred (dedicated GitHub key)
~/.ssh/id_ed25519             # Fallback
~/.ssh/id_rsa                 # Fallback
```

#### If you don't have a GitHub SSH key yet:

```bash
# 1. Generate a dedicated GitHub key
ssh-keygen -t ed25519 -C "github@yourdomain.com" -f ~/.ssh/id_ed25519_github

# 2. Add to SSH agent
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519_github

# 3. Configure SSH to use this key for GitHub
cat >> ~/.ssh/config << 'EOF'
Host github.com
  HostName github.com
  IdentityFile ~/.ssh/id_ed25519_github
  User git
  IdentitiesOnly yes
EOF

# 4. Copy the public key
cat ~/.ssh/id_ed25519_github.pub
```

#### Add the key to GitHub:

1. Go to [GitHub Settings → SSH Keys](https://github.com/settings/keys)
2. Click "New SSH key"
3. Paste your public key
4. Save

#### Test the connection:
```bash
ssh -T git@github.com
# Should output: "Hi username! You've successfully authenticated..."
```

**Note:** This is a one-time setup. The same key works for all your repos and all deployments.

## SSL/HTTPS Setup

After deployment, set up SSL with Let's Encrypt:

```bash
ssh rails@YOUR_IP
sudo certbot --nginx -d yourdomain.com -d www.yourdomain.com
```

## Custom Domain Setup

### Step 1: Configure DNS (Namecheap, GoDaddy, Cloudflare, etc.)

Go to your domain provider's DNS settings and add these records:

| Type | Host | Value | TTL |
|------|------|-------|-----|
| A Record | @ | YOUR_SERVER_IP | Automatic |
| A Record | www | YOUR_SERVER_IP | Automatic |

**Wait 5-10 minutes** for DNS propagation. You can check with:
```bash
dig yourdomain.com +short
# Should return your server IP
```

### Step 2: Update Nginx Configuration

```bash
ssh rails@YOUR_IP
sudo nano /etc/nginx/sites-available/your_app
```

Change the `server_name` line:
```nginx
server_name yourdomain.com www.yourdomain.com;
```

Test and reload:
```bash
sudo nginx -t
sudo systemctl reload nginx
```

### Step 3: Install SSL Certificate

```bash
sudo certbot --nginx -d yourdomain.com -d www.yourdomain.com
```

Certbot will:
- Obtain SSL certificate from Let's Encrypt
- Configure Nginx for HTTPS
- Add automatic HTTP → HTTPS redirect
- Set up auto-renewal (certificates expire every 90 days)

### Step 4: Verify SSL Auto-Renewal

```bash
sudo certbot renew --dry-run
```

### Troubleshooting DNS

```bash
# Check DNS propagation
dig yourdomain.com +short
nslookup yourdomain.com

# Check from external service
curl -I http://yourdomain.com
```

If DNS isn't propagating, try:
- Clear local DNS cache: `sudo dscacheutil -flushcache` (macOS)
- Wait longer (can take up to 48 hours in rare cases)
- Verify records are correct in your DNS provider

## Troubleshooting

### Check Service Status
```bash
ssh rails@YOUR_IP
sudo systemctl status puma-your_app
sudo systemctl status nginx
```

### View Logs
```bash
# Puma logs
tail -f /home/rails/your_app/shared/log/puma.log
tail -f /home/rails/your_app/shared/log/puma_error.log

# Rails logs
tail -f /home/rails/your_app/shared/log/production.log

# Nginx logs
sudo tail -f /var/log/nginx/your_app_error.log
```

### Restart Services
```bash
sudo systemctl restart puma-your_app
sudo systemctl restart nginx
```

### Database Issues
```bash
cd /home/rails/your_app/current
RAILS_ENV=production bundle exec rails db:migrate
RAILS_ENV=production bundle exec rails db:seed
```

### Bundle Issues
```bash
cd /home/rails/your_app/current
bundle config set --local deployment true
bundle install
```

## Environment Variables

Add environment variables to `/home/rails/{app_name}/shared/.env`:

```bash
ssh rails@YOUR_IP
nano /home/rails/your_app/shared/.env
```

```env
SECRET_KEY_BASE=your_secret_key
STRIPE_API_KEY=sk_live_xxx
REDIS_URL=redis://localhost:6379
```

## Multiple Apps on Same Server

The script supports multiple apps on the same server. Each app gets:
- Its own database
- Its own Nginx config
- Its own Puma systemd service
- Its own directory under `/home/rails/`

Just run `rails-deploy-init` in each app directory with the same server IP.

## Customization

### Change Git Branch

Edit `config/deploy.rb`:
```ruby
set :branch, 'production'  # or 'master', 'develop', etc.
```

### Add Environment Variables to Deploy

Edit `config/deploy.rb`, add to shared_files:
```ruby
set :shared_files, fetch(:shared_files, []).push(
  'config/database.yml', 'config/master.key', '.env',
  'config/application.yml'  # for figaro
)
```

### Custom Nginx Config

Edit `deployment/nginx/{app_name}` before first deploy, or edit directly on server:
```bash
sudo nano /etc/nginx/sites-available/your_app
sudo nginx -t && sudo systemctl reload nginx
```

## How It Works

```
┌─────────────────────────────────────────────────────────────────┐
│  LOCAL MACHINE                                                   │
├─────────────────────────────────────────────────────────────────┤
│  rails-deploy-init                                               │
│    ├── Detects: app name, ruby version, git repo                │
│    ├── Reads: SSH keys from ~/.ssh/                             │
│    ├── Generates: deploy.rb, nginx, puma, systemd configs       │
│    ├── Copies: deployment/ folder to server via .pem            │
│    └── Runs: server_setup.sh via SSH                            │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  AWS EC2 SERVER                                                  │
├─────────────────────────────────────────────────────────────────┤
│  server_setup.sh                                                 │
│    ├── Installs: Ruby, PostgreSQL, Nginx, Node.js              │
│    ├── Creates: rails user with your SSH keys                   │
│    ├── Configures: GitHub SSH access                            │
│    ├── Creates: database and app directories                    │
│    └── Enables: Puma systemd service                            │
│                                                                  │
│  Then Mina deploys:                                              │
│    ├── git clone from GitHub                                     │
│    ├── bundle install                                            │
│    ├── rails db:migrate                                          │
│    ├── rails assets:precompile                                   │
│    └── systemctl restart puma                                    │
└─────────────────────────────────────────────────────────────────┘
```

## Contributing

1. Fork the repo
2. Create your feature branch (`git checkout -b feature/amazing`)
3. Commit your changes (`git commit -am 'Add amazing feature'`)
4. Push to the branch (`git push origin feature/amazing`)
5. Open a Pull Request

## License

MIT License - see [LICENSE](LICENSE) file.

## Credits

Built with ❤️ for the Rails community.

Uses:
- [Mina](https://github.com/mina-deploy/mina) - Fast deployer
- [Puma](https://github.com/puma/puma) - Ruby web server
- [rbenv](https://github.com/rbenv/rbenv) - Ruby version manager