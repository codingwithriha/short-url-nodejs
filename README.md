# 🚀 Node.js Deployment on AWS EC2

> A complete step-by-step guide to deploy a Node.js / Express application on **AWS EC2 Ubuntu** using **PM2**, **NGINX**, a custom domain, **Elastic IP**, and **Let's Encrypt SSL**.

---

## 📌 Deployment Architecture

```text
                         Internet
                            |
                            |
                     yourdomain.com
                            |
                            v
                       Elastic IP
                            |
                            v
                    AWS EC2 Ubuntu
                            |
                     +------+------+
                     |             |
                   :80           :443
                   HTTP          HTTPS
                     |             |
                     +------+------+
                            |
                           NGINX
                            |
                     Reverse Proxy
                            |
                            v
                    localhost:8001
                            |
                            v
                       Node.js App
                            |
                            v
                           PM2
```

### What does each component do?

| Component          | Purpose                              |
| ------------------ | ------------------------------------ |
| **AWS EC2**        | Provides the cloud server            |
| **Ubuntu**         | Operating system running on EC2      |
| **Security Group** | AWS-level firewall                   |
| **Elastic IP**     | Stable public IP address             |
| **Node.js**        | Runs the backend application         |
| **PM2**            | Keeps Node.js application running    |
| **NGINX**          | Reverse proxy and web server         |
| **Domain**         | Human-readable website address       |
| **Certbot**        | Automatically obtains/configures SSL |
| **Let's Encrypt**  | Provides free SSL/TLS certificates   |

---

# 1. Create an AWS Account

Create an AWS account:

https://aws.amazon.com/

After creating the account, open:

```text
AWS Console
    ↓
EC2
```

---

# 2. Create an EC2 Instance

Go to:

```text
AWS Console
→ EC2
→ Instances
→ Launch Instance
```

### Recommended configuration for a learning project

```text
AMI:
Ubuntu Server 24.04 LTS

Architecture:
64-bit

Instance Type:
Choose a suitable small instance for your workload

Key Pair:
Create a new key pair
```

Download the `.pem` key and **keep it safe**.

You will need this key to SSH into your server.

---

# 3. Configure the Security Group

During EC2 creation, configure inbound rules.

| Type  | Port | Source        | Purpose       |
| ----- | ---: | ------------- | ------------- |
| SSH   |   22 | My IP         | SSH access    |
| HTTP  |   80 | Anywhere IPv4 | HTTP traffic  |
| HTTPS |  443 | Anywhere IPv4 | HTTPS traffic |

For SSH, it is safer to use:

```text
My IP
```

instead of:

```text
0.0.0.0/0
```

You normally **do not need to expose your Node.js port**, for example:

```text
8001
```

because NGINX will communicate with Node.js through localhost.

---

# 4. Connect to EC2 Using SSH

After the instance starts, copy its public IP address.

Example:

```text
3.XX.XX.XX
```

On your local machine, go to the folder containing your `.pem` file.

Set the correct permissions:

```bash
chmod 400 your-key.pem
```

Then connect:

```bash
ssh -i "your-key.pem" ubuntu@YOUR_EC2_PUBLIC_IP
```

Example:

```bash
ssh -i "demo-server.pem" ubuntu@3.XX.XX.XX
```

For Ubuntu, the default username is normally:

```text
ubuntu
```

Check the connection:

```bash
whoami
```

Expected:

```text
ubuntu
```

---

# 5. Update Ubuntu

After connecting:

```bash
sudo apt update
sudo apt upgrade -y
```

Install some useful packages:

```bash
sudo apt install -y git curl
```

Verify Git:

```bash
git --version
```

---

# 6. Install Node.js

## Recommended: Install Node.js using NVM

Older tutorials may use:

```bash
curl -sL https://deb.nodesource.com/setup_18.x | sudo -E bash -
```

Node.js 18 is now outdated/EOL, so don't use that command for a new deployment.

Instead, install NVM.

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.7/install.sh | bash
```

Reload the shell:

```bash
source ~/.bashrc
```

Check NVM:

```bash
nvm --version
```

Install Node.js 24:

```bash
nvm install 24
```

Use it:

```bash
nvm use 24
```

Make it the default:

```bash
nvm alias default 24
```

Check:

```bash
node --version
npm --version
```

Example:

```text
v24.x.x
11.x.x
```

> If your project requires another Node.js version, use the version specified by the project.

---

# 7. Clone Your Project from GitHub

Go to your home directory:

```bash
cd ~
```

Clone your repository:

```bash
git clone https://github.com/YOUR_USERNAME/YOUR_REPOSITORY.git
```

Example:

```bash
git clone https://github.com/piyushgargdev-01/short-url-nodejs
```

Enter the project:

```bash
cd YOUR_REPOSITORY
```

Check files:

```bash
ls
```

---

# 8. Configure Environment Variables

If your project uses environment variables, create a `.env` file:

```bash
nano .env
```

Example:

```env
PORT=8001
NODE_ENV=production
MONGO_URI=your_mongodb_connection_string
JWT_SECRET=your_secret
```

Save the file:

```text
CTRL + O
ENTER
CTRL + X
```

Check the file:

```bash
ls -la
```

You should see:

```text
.env
```

### ⚠️ Important

Never commit secrets to GitHub.

Your `.gitignore` should contain:

```gitignore
.env
node_modules/
```

---

# 9. Install Project Dependencies

Inside your project directory:

```bash
npm install
```

This installs the packages from:

```text
package.json
```

and:

```text
package-lock.json
```

---

# 10. Test the Application

Before using PM2 or NGINX, make sure the application itself works.

Depending on your project, you may use:

```bash
npm start
```

or:

```bash
node index.js
```

For example:

```bash
node index.js
```

If your application runs on:

```text
8001
```

test it from the EC2 server:

```bash
curl http://localhost:8001
```

or:

```bash
curl http://127.0.0.1:8001
```

If you receive a response, your Node.js application is running.

Stop the application:

```text
CTRL + C
```

---

# 11. Install PM2

PM2 keeps your Node.js application running in the background.

Install PM2 globally:

```bash
sudo npm install -g pm2
```

Check the installation:

```bash
pm2 --version
```

---

# 12. Start the Application with PM2

If your entry file is:

```text
index.js
```

run:

```bash
pm2 start index.js --name my-app
```

If your application starts using:

```bash
npm start
```

you can use:

```bash
pm2 start npm --name my-app -- start
```

Check PM2:

```bash
pm2 status
```

You should see something similar to:

```text
┌────┬──────────┬─────────┬──────┬────────┐
│ id │ name     │ mode    │ ↺    │ status │
├────┼──────────┼─────────┼──────┼────────┤
│ 0  │ my-app   │ fork    │ 0    │ online │
└────┴──────────┴─────────┴──────┴────────┘
```

---

# 13. Useful PM2 Commands

### Check application status

```bash
pm2 status
```

### Show application information

```bash
pm2 show my-app
```

### View logs

```bash
pm2 logs my-app
```

### Restart application

```bash
pm2 restart my-app
```

### Stop application

```bash
pm2 stop my-app
```

### Delete application from PM2

```bash
pm2 delete my-app
```

### Clear logs

```bash
pm2 flush
```

---

# 14. Configure PM2 to Start After Reboot

Without configuration, your application may not automatically start after an EC2 reboot.

Run:

```bash
pm2 startup
```

PM2 will display a command similar to:

```bash
sudo env PATH=$PATH:/home/ubuntu/.nvm/versions/node/v24.x.x/bin pm2 startup systemd -u ubuntu --hp /home/ubuntu
```

**Do not blindly copy the example above.**

Run the exact command that PM2 gives you.

Then save the current PM2 processes:

```bash
pm2 save
```

Now PM2 knows that `my-app` should start automatically.

---

# 15. Test PM2 After Reboot

You can reboot the EC2 server:

```bash
sudo reboot
```

Your SSH connection will close.

Wait a little and reconnect:

```bash
ssh -i "your-key.pem" ubuntu@YOUR_EC2_PUBLIC_IP
```

Then:

```bash
pm2 status
```

Your application should show:

```text
online
```

---

# 16. Configure Firewall

There are two important firewall layers:

```text
Internet
    ↓
AWS Security Group
    ↓
Ubuntu UFW
    ↓
NGINX / Application
```

AWS Security Group is the primary AWS network firewall.

Check UFW:

```bash
sudo ufw status
```

If you want to enable UFW, **allow SSH first**:

```bash
sudo ufw allow OpenSSH
```

Allow HTTP:

```bash
sudo ufw allow 80/tcp
```

Allow HTTPS:

```bash
sudo ufw allow 443/tcp
```

Then:

```bash
sudo ufw enable
```

Check:

```bash
sudo ufw status
```

Expected:

```text
22/tcp
80/tcp
443/tcp
```

> Make sure SSH is allowed before enabling UFW, otherwise you may lock yourself out.

---

# 17. Install NGINX

Install NGINX:

```bash
sudo apt update
sudo apt install -y nginx
```

Check NGINX:

```bash
sudo systemctl status nginx
```

If it isn't running:

```bash
sudo systemctl start nginx
```

Enable NGINX at startup:

```bash
sudo systemctl enable nginx
```

Now open:

```text
http://YOUR_EC2_PUBLIC_IP
```

You should see the NGINX welcome page.

---

# 18. Configure NGINX as Reverse Proxy

Open the default NGINX configuration:

```bash
sudo nano /etc/nginx/sites-available/default
```

Configure the server block:

```nginx
server {
    listen 80;
    listen [::]:80;

    server_name yourdomain.com www.yourdomain.com;

    location / {
        proxy_pass http://127.0.0.1:8001;

        proxy_http_version 1.1;

        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_cache_bypass $http_upgrade;
    }
}
```

Replace:

```text
yourdomain.com
```

with your actual domain.

Also replace:

```text
8001
```

with your application's actual port.

---

# 19. Understand the Reverse Proxy

The user does **not** directly access:

```text
http://yourdomain.com:8001
```

Instead:

```text
Browser
   |
   | http://yourdomain.com
   ↓
NGINX :80
   |
   | proxy_pass
   ↓
Node.js :8001
```

NGINX receives the public request and forwards it internally to Node.js.

This is why port `8001` doesn't need to be publicly exposed.

---

# 20. Test NGINX Configuration

Before restarting/reloading NGINX:

```bash
sudo nginx -t
```

If everything is correct, you should see:

```text
syntax is ok
test is successful
```

Then reload:

```bash
sudo systemctl reload nginx
```

You can also use:

```bash
sudo nginx -s reload
```

---

# 21. Configure Your Domain

Go to the DNS provider where your domain is managed.

Create an A record:

```text
Type: A
Name: @
Value: YOUR_EC2_PUBLIC_IP
```

For `www`:

```text
Type: A
Name: www
Value: YOUR_EC2_PUBLIC_IP
```

Example:

```text
yourdomain.com
        ↓
A Record
        ↓
3.XX.XX.XX
```

DNS propagation may take some time.

---

# 22. Use an Elastic IP

For a long-running server, it is useful to have a stable public IP.

An EC2 public IPv4 address can change when the instance is stopped and started.

An Elastic IP provides a stable address that you can associate with your EC2 instance.

Go to:

```text
AWS Console
→ EC2
→ Elastic IPs
→ Allocate Elastic IP
```

Then associate it with your EC2 instance.

Your architecture becomes:

```text
Domain
   ↓
Elastic IP
   ↓
EC2 Instance
   ↓
NGINX
   ↓
Node.js
```

Update your DNS A records to point to the Elastic IP.

### ⚠️ Important

AWS pricing for public IPv4/Elastic IP resources can change.

Before leaving an Elastic IP allocated, check the current AWS pricing for your account/region.

---

# 23. Verify DNS

From your local machine:

```bash
nslookup yourdomain.com
```

or:

```bash
dig yourdomain.com
```

The result should resolve to your EC2/Elastic IP.

Then test:

```text
http://yourdomain.com
```

---

# 24. Test HTTP Before SSL

Before installing SSL, make sure this works:

```text
http://yourdomain.com
```

Your request should follow:

```text
Browser
   ↓
yourdomain.com
   ↓
EC2
   ↓
NGINX :80
   ↓
Node.js :8001
```

Only move to SSL after HTTP works correctly.

---

# 25. Install Certbot

Certbot is used to obtain and configure Let's Encrypt SSL certificates.

The old tutorial may contain:

```bash
sudo add-apt-repository ppa:certbot/certbot
```

This is an outdated approach for a new setup.

Install Snap:

```bash
sudo apt update
sudo apt install -y snapd
```

Install Certbot:

```bash
sudo snap install --classic certbot
```

Create the command symlink:

```bash
sudo ln -s /snap/bin/certbot /usr/local/bin/certbot
```

Check:

```bash
certbot --version
```

---

# 26. Generate SSL Certificate

Run:

```bash
sudo certbot --nginx -d yourdomain.com -d www.yourdomain.com
```

Certbot will:

```text
Verify domain
      ↓
Request Let's Encrypt certificate
      ↓
Configure NGINX
      ↓
Enable HTTPS
      ↓
Configure redirect if selected
```

After successful configuration:

```text
https://yourdomain.com
```

should open your application.

---

# 27. Test SSL Renewal

Let's Encrypt certificates are short-lived, so automatic renewal is important.

Test the renewal process:

```bash
sudo certbot renew --dry-run
```

If the test succeeds, you can be confident the renewal configuration is working.

---

# 28. Final Architecture

After everything is configured:

```text
                         INTERNET
                            |
                            |
                     yourdomain.com
                            |
                            v
                       Elastic IP
                            |
                            v
                    AWS EC2 Ubuntu
                            |
                 +----------+----------+
                 |                     |
               HTTP                   HTTPS
               :80                    :443
                 |                     |
                 +----------+----------+
                            |
                          NGINX
                            |
                     Reverse Proxy
                            |
                            v
                    127.0.0.1:8001
                            |
                            v
                       Node.js App
                            |
                            v
                           PM2
```

---

# 29. Updating Your Application

Suppose you make changes locally.

Push them to GitHub:

```bash
git add .
git commit -m "Update application"
git push
```

SSH into EC2:

```bash
ssh -i "your-key.pem" ubuntu@YOUR_EC2_PUBLIC_IP
```

Go to your project:

```bash
cd ~/YOUR_REPOSITORY
```

Pull the latest code:

```bash
git pull
```

Install dependencies if necessary:

```bash
npm install
```

Restart PM2:

```bash
pm2 restart my-app
```

Check:

```bash
pm2 status
```

Check logs:

```bash
pm2 logs my-app
```

---

# 30. Complete Deployment Workflow

Whenever you deploy a Node.js project to AWS EC2, remember this sequence:

```text
1. Create AWS Account
          ↓
2. Launch EC2 Ubuntu
          ↓
3. Configure Security Group
          ↓
4. SSH into EC2
          ↓
5. Update Ubuntu
          ↓
6. Install Node.js
          ↓
7. Clone GitHub Repository
          ↓
8. Create .env
          ↓
9. npm install
          ↓
10. Test Node.js App
          ↓
11. Install PM2
          ↓
12. Start App with PM2
          ↓
13. pm2 startup
          ↓
14. pm2 save
          ↓
15. Install NGINX
          ↓
16. Configure Reverse Proxy
          ↓
17. Configure Domain DNS
          ↓
18. Configure Elastic IP
          ↓
19. Test HTTP
          ↓
20. Install Certbot
          ↓
21. Generate SSL
          ↓
22. Test HTTPS
          ↓
23. Test SSL Renewal
          ↓
24. Deployment Complete 🚀
```

---

# 31. Troubleshooting

## ❌ Problem: Website is not opening

First check PM2:

```bash
pm2 status
```

Then:

```bash
pm2 logs my-app
```

Test Node directly:

```bash
curl http://127.0.0.1:8001
```

Check NGINX:

```bash
sudo systemctl status nginx
```

Test configuration:

```bash
sudo nginx -t
```

Check firewall:

```bash
sudo ufw status
```

Also check the AWS Security Group.

---

# 32. ❌ Problem: 502 Bad Gateway

A `502 Bad Gateway` usually means NGINX cannot communicate with your Node.js application.

Check:

```bash
pm2 status
```

Then:

```bash
curl http://127.0.0.1:8001
```

If this fails, the problem is probably with the Node.js application or PM2.

Check:

```bash
pm2 logs my-app
```

Also verify your NGINX configuration:

```nginx
proxy_pass http://127.0.0.1:8001;
```

Make sure `8001` matches your actual Node.js port.

---

# 33. ❌ Problem: NGINX Configuration Error

Run:

```bash
sudo nginx -t
```

If there is an error, NGINX will tell you the configuration file and line where the problem occurred.

After fixing it:

```bash
sudo nginx -t
```

Then:

```bash
sudo systemctl reload nginx
```

---

# 34. ❌ Problem: SSL Certificate Failed

Make sure:

```text
http://yourdomain.com
```

works first.

Check DNS:

```bash
nslookup yourdomain.com
```

Check NGINX:

```bash
sudo nginx -t
```

Check that AWS Security Group allows:

```text
Port 80
Port 443
```

If UFW is enabled:

```bash
sudo ufw status
```

Make sure these are allowed:

```text
80/tcp
443/tcp
```

---

# 35. ❌ Problem: Application Stops After Closing SSH

If you started your app like:

```bash
node index.js
```

it is tied to your terminal session.

Instead use PM2:

```bash
pm2 start index.js --name my-app
```

Then:

```bash
pm2 save
```

and:

```bash
pm2 startup
```

---

# 36. ❌ Problem: Application Doesn't Start After Reboot

Check:

```bash
pm2 status
```

If it isn't running:

```bash
pm2 resurrect
```

Also check whether PM2 startup was configured:

```bash
pm2 startup
```

Then:

```bash
pm2 save
```

---

# 37. ❌ Problem: Application Works Locally But Not on EC2

Check Node:

```bash
node -v
```

Check npm:

```bash
npm -v
```

Check environment variables:

```bash
ls -la
```

Check PM2:

```bash
pm2 status
```

Check logs:

```bash
pm2 logs my-app
```

Check listening ports:

```bash
sudo ss -ltnp
```

Test the application:

```bash
curl http://127.0.0.1:8001
```

---

# 38. ❌ Problem: Domain Doesn't Work

Check:

```bash
nslookup yourdomain.com
```

Make sure the domain points to:

```text
EC2 Public IP
```

or preferably your configured:

```text
Elastic IP
```

Also check your NGINX configuration:

```nginx
server_name yourdomain.com www.yourdomain.com;
```

Then:

```bash
sudo nginx -t
```

and:

```bash
sudo systemctl reload nginx
```

---

# 39. Useful Linux Commands

### Current directory

```bash
pwd
```

### List files

```bash
ls
```

### List hidden files

```bash
ls -la
```

### Change directory

```bash
cd folder-name
```

### Go back

```bash
cd ..
```

### Go home

```bash
cd ~
```

### Edit file

```bash
nano filename
```

### Check running processes

```bash
ps aux
```

---

# 40. Git Commands

Clone:

```bash
git clone REPOSITORY_URL
```

Check status:

```bash
git status
```

Pull changes:

```bash
git pull
```

---

# 41. Node.js Commands

Check Node:

```bash
node -v
```

Check npm:

```bash
npm -v
```

Install dependencies:

```bash
npm install
```

Start application:

```bash
npm start
```

Run JavaScript file:

```bash
node index.js
```

---

# 42. PM2 Commands

```bash
pm2 start index.js --name my-app
```

```bash
pm2 status
```

```bash
pm2 show my-app
```

```bash
pm2 logs my-app
```

```bash
pm2 restart my-app
```

```bash
pm2 stop my-app
```

```bash
pm2 delete my-app
```

```bash
pm2 save
```

```bash
pm2 startup
```

```bash
pm2 flush
```

---

# 43. NGINX Commands

Check status:

```bash
sudo systemctl status nginx
```

Start:

```bash
sudo systemctl start nginx
```

Stop:

```bash
sudo systemctl stop nginx
```

Restart:

```bash
sudo systemctl restart nginx
```

Reload:

```bash
sudo systemctl reload nginx
```

Test configuration:

```bash
sudo nginx -t
```

View error logs:

```bash
sudo tail -f /var/log/nginx/error.log
```

View access logs:

```bash
sudo tail -f /var/log/nginx/access.log
```

---

# 44. Certbot Commands

Check version:

```bash
certbot --version
```

Generate SSL:

```bash
sudo certbot --nginx -d yourdomain.com -d www.yourdomain.com
```

Test renewal:

```bash
sudo certbot renew --dry-run
```

Check certificates:

```bash
sudo certbot certificates
```

---

# 45. Network Debugging Commands

Check application response:

```bash
curl http://127.0.0.1:8001
```

Check listening ports:

```bash
sudo ss -ltnp
```

Check DNS:

```bash
nslookup yourdomain.com
```

Check DNS using `dig`:

```bash
dig yourdomain.com
```

---

# 46. Security Best Practices

## Never expose secrets

Do not put:

```text
.env
AWS credentials
Database passwords
JWT secrets
API keys
Private keys
```

inside GitHub.

Use:

```gitignore
.env
node_modules/
```

---

## Don't expose your Node.js port unnecessarily

Avoid opening:

```text
8001
```

to the entire internet.

Instead:

```text
Internet
   ↓
80 / 443
   ↓
NGINX
   ↓
127.0.0.1:8001
   ↓
Node.js
```

---

## Restrict SSH

Instead of:

```text
0.0.0.0/0
```

use your own IP where practical.

---

# 47. Stop vs Terminate EC2

This is extremely important.

## STOP

When you stop an EC2 instance:

```text
Running
   ↓
STOP
   ↓
Stopped
   ↓
START
   ↓
Running
```

The EC2 instance still exists.

You can start the same instance again.

However, other resources such as EBS storage and certain public IPv4 resources can still incur charges.

---

## TERMINATE

When you terminate an EC2 instance:

```text
Running
   ↓
TERMINATE
   ↓
Deleted
```

Termination cannot be undone.

You cannot later click:

```text
Start Instance
```

and bring that exact terminated instance back.

You would have to create a new EC2 instance.

---

# 48. What Happens to the Elastic IP?

If you are using an Elastic IP, don't assume that terminating an EC2 instance automatically means all related resources are gone.

Check:

```text
EC2
→ Elastic IPs
```

If the Elastic IP is no longer needed, release it.

Also check:

```text
EBS Volumes
Snapshots
Load Balancers
Databases
Other AWS resources
```

AWS billing can come from resources other than the EC2 compute instance itself.

---

# 49. Safe Cleanup After a Demo

When your project/demo is finished, review the resources you created.

Checklist:

```text
[ ] EC2 Instance
[ ] EBS Volumes
[ ] Elastic IP / Public IPv4 resources
[ ] Snapshots
[ ] Load Balancers
[ ] Databases
[ ] Other AWS resources
```

If you don't need the server anymore:

```text
Terminate EC2
        ↓
Remove unused resources
        ↓
Release unused Elastic IP
        ↓
Check AWS Billing
```

### ⚠️ Before terminating

Make sure you don't need the server for:

* Mentor demonstration
* Assignment submission
* Client demo
* Portfolio demonstration
* Testing
* Further learning

If you still need it, **do not terminate it**.

---

# 50. Final Deployment Checklist

Use this checklist whenever deploying:

```text
AWS
[ ] AWS account created
[ ] EC2 Ubuntu instance created
[ ] Key pair downloaded
[ ] Security Group configured
[ ] SSH connection working

SERVER
[ ] Ubuntu updated
[ ] Git installed
[ ] Node.js installed
[ ] npm working

APPLICATION
[ ] Repository cloned
[ ] .env configured
[ ] npm install completed
[ ] Application tested
[ ] Application port confirmed

PM2
[ ] PM2 installed
[ ] Application started
[ ] pm2 status checked
[ ] pm2 logs checked
[ ] pm2 startup configured
[ ] pm2 save completed
[ ] Reboot tested

NGINX
[ ] NGINX installed
[ ] Reverse proxy configured
[ ] nginx -t successful
[ ] NGINX reloaded

DOMAIN
[ ] Domain purchased/configured
[ ] A record configured
[ ] DNS points to server
[ ] HTTP working

IP
[ ] Elastic IP configured if needed
[ ] DNS updated to correct IP

SSL
[ ] Certbot installed
[ ] SSL certificate generated
[ ] HTTPS working
[ ] HTTP → HTTPS redirect working
[ ] certbot renew --dry-run successful

SECURITY
[ ] .env not committed
[ ] Secrets protected
[ ] Node.js port not publicly exposed
[ ] SSH restricted where possible

FINAL
[ ] Website opens
[ ] API works
[ ] PM2 online
[ ] NGINX working
[ ] HTTPS working
[ ] Logs checked
```

---

# 🧠 Quick Revision

The entire deployment can be remembered with this simple flow:

```text
AWS
 ↓
EC2
 ↓
Ubuntu
 ↓
SSH
 ↓
Node.js
 ↓
Git Clone
 ↓
.env
 ↓
npm install
 ↓
Test App
 ↓
PM2
 ↓
PM2 Startup
 ↓
PM2 Save
 ↓
NGINX
 ↓
Domain
 ↓
Elastic IP
 ↓
HTTP
 ↓
Certbot
 ↓
SSL / HTTPS
 ↓
🚀 LIVE
```

---

# 🎯 The Most Important Commands

If you already understand the concepts, these are the commands you'll use most often:

```bash
# Connect
ssh -i "your-key.pem" ubuntu@YOUR_EC2_IP

# Update
sudo apt update && sudo apt upgrade -y

# Node
node -v
npm -v

# Clone
git clone YOUR_REPOSITORY_URL

# Install
npm install

# PM2
sudo npm install -g pm2
pm2 start index.js --name my-app
pm2 status
pm2 logs my-app
pm2 save
pm2 startup

# NGINX
sudo apt install -y nginx
sudo nginx -t
sudo systemctl reload nginx

# SSL
sudo snap install --classic certbot
sudo certbot --nginx -d yourdomain.com -d www.yourdomain.com
sudo certbot renew --dry-run
```

---

## ✅ Final Result

After completing this guide, your Node.js application will be running on:

```text
AWS EC2
   ↓
Ubuntu
   ↓
Node.js
   ↓
PM2
   ↓
NGINX
   ↓
Domain
   ↓
Let's Encrypt SSL
   ↓
HTTPS
   ↓
🚀 Production Application
```
