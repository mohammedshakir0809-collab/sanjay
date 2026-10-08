# AWS Single EC2 Deployment Guide

## Node.js + MySQL + Apache on One Ubuntu EC2 Server

This practical project deploys a Node.js application, MySQL database,
and Apache web server on a **single AWS EC2 Ubuntu instance**.

### Architecture

``` text
                    Internet
                       |
                    Port 80
                       |
                  +---------+
                  | Apache  |
                  | HTTPD   |
                  +---------+
                       |
                 Reverse Proxy
                       |
                  Port 3000
                       |
                +-------------+
                | Node.js API |
                +-------------+
                       |
                 localhost:3306
                       |
                  +----------+
                  |  MySQL   |
                  +----------+
```

Everything runs on the same EC2 instance.

------------------------------------------------------------------------

# 1. Project Requirements

You need:

-   AWS account
-   EC2 Ubuntu server
-   SSH access
-   Node.js application
-   MySQL database
-   Apache web server
-   Domain name (optional)
-   Git/GitHub repository (optional)

Recommended EC2 for training:

-   OS: Ubuntu Server 24.04 LTS
-   Instance: t3.small or larger for a practical demo
-   Storage: 20 GB or more
-   Security Group:
    -   SSH: TCP 22 --- My IP
    -   HTTP: TCP 80 --- Anywhere
    -   HTTPS: TCP 443 --- Anywhere
-   Do **not** expose MySQL port 3306 to the Internet unless there is a
    specific requirement.

------------------------------------------------------------------------

# 2. Create EC2 Instance

Go to AWS Console:

``` text
AWS Console
→ EC2
→ Instances
→ Launch Instance
```

Choose:

``` text
Name: node-mysql-apache-server
AMI: Ubuntu Server 24.04 LTS
Instance type: t3.small
Storage: 20 GB+
Key pair: Create/select your key pair
```

Security Group:

  Type      Port Source
  ------- ------ -----------
  SSH         22 My IP
  HTTP        80 0.0.0.0/0
  HTTPS      443 0.0.0.0/0

Do not add:

``` text
3306 from 0.0.0.0/0
```

MySQL will be used locally by Node.js.

------------------------------------------------------------------------

# 3. Connect to EC2

From Linux/macOS/WSL:

``` bash
chmod 400 my-key.pem

ssh -i my-key.pem ubuntu@YOUR_EC2_PUBLIC_IP
```

Example:

``` bash
ssh -i aws-project.pem ubuntu@13.232.52.188
```

Check the server:

``` bash
whoami
hostname
pwd
```

Expected user:

``` text
ubuntu
```

------------------------------------------------------------------------

# 4. Update Ubuntu

``` bash
sudo apt update
sudo apt upgrade -y
```

Install useful packages:

``` bash
sudo apt install -y curl wget git unzip build-essential
```

Check OS:

``` bash
cat /etc/os-release
```

------------------------------------------------------------------------

# 5. Configure EC2 Timezone

For India:

``` bash
sudo timedatectl set-timezone Asia/Kolkata
```

Check:

``` bash
timedatectl
```

------------------------------------------------------------------------

# 6. Install Node.js

For a production server, use an LTS Node.js version.

Install Node.js using NodeSource:

``` bash
curl -fsSL https://deb.nodesource.com/setup_lts.x | sudo -E bash -
sudo apt install -y nodejs
```

Check versions:

``` bash
node -v
npm -v
```

Example:

``` text
v22.x.x
10.x.x
```

The exact LTS version may change over time.

------------------------------------------------------------------------

# 7. Create Application User

For a simple training project, you can use the `ubuntu` user.

For a more controlled deployment, create a separate user:

``` bash
sudo adduser nodeapp
```

Add it to the required group if necessary:

``` bash
sudo usermod -aG www-data nodeapp
```

For this guide, the commands below use:

``` text
ubuntu
```

to keep the deployment simple.

------------------------------------------------------------------------

# 8. Install MySQL

``` bash
sudo apt install -y mysql-server
```

Check status:

``` bash
sudo systemctl status mysql
```

If it is not running:

``` bash
sudo systemctl start mysql
```

Enable MySQL after reboot:

``` bash
sudo systemctl enable mysql
```

Check:

``` bash
sudo systemctl is-enabled mysql
```

------------------------------------------------------------------------

# 9. Secure MySQL

Run:

``` bash
sudo mysql_secure_installation
```

Follow the prompts.

For a training server, choose sensible security settings and remove
anonymous users and the test database.

Never use a simple production password.

------------------------------------------------------------------------

# 10. Create MySQL Database

Login:

``` bash
sudo mysql
```

Create database:

``` sql
CREATE DATABASE awsapp;
```

Create application user:

``` sql
CREATE USER 'awsuser'@'localhost' IDENTIFIED BY 'CHANGE_THIS_STRONG_PASSWORD';
```

Grant permissions:

``` sql
GRANT ALL PRIVILEGES ON awsapp.* TO 'awsuser'@'localhost';
```

Apply:

``` sql
FLUSH PRIVILEGES;
```

Exit:

``` sql
EXIT;
```

------------------------------------------------------------------------

# 11. Test MySQL Login

``` bash
mysql -u awsuser -p
```

Enter the password.

Select database:

``` sql
USE awsapp;
```

Exit:

``` sql
EXIT;
```

------------------------------------------------------------------------

# 12. Create a Sample Table

Login:

``` bash
mysql -u awsuser -p awsapp
```

Create table:

``` sql
CREATE TABLE users (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(150) NOT NULL UNIQUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

Insert sample data:

``` sql
INSERT INTO users (name, email)
VALUES ('Hari', 'hari@example.com');
```

Check:

``` sql
SELECT * FROM users;
```

Exit:

``` sql
EXIT;
```

------------------------------------------------------------------------

# 13. Create Node.js Project

Create application directory:

``` bash
mkdir -p ~/aws-node-app
cd ~/aws-node-app
```

Initialize Node.js:

``` bash
npm init -y
```

Install packages:

``` bash
npm install express mysql2 dotenv
```

Install development tool:

``` bash
npm install --save-dev nodemon
```

------------------------------------------------------------------------

# 14. Create Application Files

Project structure:

``` text
aws-node-app/
├── node_modules/
├── .env
├── .gitignore
├── package.json
├── server.js
└── public/
    └── index.html
```

Create files:

``` bash
touch server.js
touch .env
touch .gitignore
mkdir public
touch public/index.html
```

------------------------------------------------------------------------

# 15. Create Node.js Server

Open:

``` bash
nano server.js
```

Add:

``` javascript
const express = require("express");
const mysql = require("mysql2/promise");
require("dotenv").config();

const app = express();

app.use(express.json());
app.use(express.static("public"));

const pool = mysql.createPool({
    host: process.env.DB_HOST,
    user: process.env.DB_USER,
    password: process.env.DB_PASSWORD,
    database: process.env.DB_NAME,
    waitForConnections: true,
    connectionLimit: 10,
    queueLimit: 0
});

app.get("/api/health", async (req, res) => {
    try {
        await pool.query("SELECT 1");

        res.json({
            status: "OK",
            message: "Node.js and MySQL are connected"
        });
    } catch (error) {
        console.error(error);

        res.status(500).json({
            status: "ERROR",
            message: "Database connection failed"
        });
    }
});

app.get("/api/users", async (req, res) => {
    try {
        const [rows] = await pool.query(
            "SELECT * FROM users ORDER BY id DESC"
        );

        res.json(rows);
    } catch (error) {
        console.error(error);

        res.status(500).json({
            error: "Unable to fetch users"
        });
    }
});

app.post("/api/users", async (req, res) => {
    try {
        const { name, email } = req.body;

        if (!name || !email) {
            return res.status(400).json({
                error: "Name and email are required"
            });
        }

        const [result] = await pool.query(
            "INSERT INTO users (name, email) VALUES (?, ?)",
            [name, email]
        );

        res.status(201).json({
            message: "User created",
            id: result.insertId
        });
    } catch (error) {
        console.error(error);

        res.status(500).json({
            error: "Unable to create user"
        });
    }
});

const PORT = process.env.PORT || 3000;

app.listen(PORT, "127.0.0.1", () => {
    console.log(`Server running on port ${PORT}`);
});
```

Important:

``` javascript
app.listen(PORT, "127.0.0.1")
```

means Node.js listens only locally.

Apache will receive public requests and proxy them to Node.js.

------------------------------------------------------------------------

# 16. Configure Environment Variables

Open:

``` bash
nano .env
```

Add:

``` env
PORT=3000

DB_HOST=127.0.0.1
DB_USER=awsuser
DB_PASSWORD=CHANGE_THIS_STRONG_PASSWORD
DB_NAME=awsapp
```

Protect the file:

``` bash
chmod 600 .env
```

Never commit `.env` to GitHub.

------------------------------------------------------------------------

# 17. Create .gitignore

``` bash
nano .gitignore
```

Add:

``` text
node_modules/
.env
npm-debug.log
```

------------------------------------------------------------------------

# 18. Create Frontend Page

Open:

``` bash
nano public/index.html
```

Add:

``` html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">

    <title>AWS Node.js Project</title>

    <style>
        body {
            font-family: Arial, sans-serif;
            max-width: 900px;
            margin: 50px auto;
            padding: 20px;
        }

        input,
        button {
            padding: 10px;
            margin: 5px;
        }

        li {
            margin: 8px 0;
        }
    </style>
</head>

<body>

    <h1>AWS Node.js + MySQL</h1>

    <p id="status">Checking database...</p>

    <h2>Add User</h2>

    <input id="name" placeholder="Name">
    <input id="email" placeholder="Email">

    <button onclick="addUser()">
        Add User
    </button>

    <h2>Users</h2>

    <ul id="users"></ul>

    <script>

        async function checkHealth() {

            const response =
                await fetch("/api/health");

            const data =
                await response.json();

            document.getElementById("status")
                .textContent = data.message;
        }


        async function loadUsers() {

            const response =
                await fetch("/api/users");

            const users =
                await response.json();

            const list =
                document.getElementById("users");

            list.innerHTML = "";

            users.forEach(user => {

                const li =
                    document.createElement("li");

                li.textContent =
                    `${user.name} - ${user.email}`;

                list.appendChild(li);
            });
        }


        async function addUser() {

            const name =
                document.getElementById("name").value;

            const email =
                document.getElementById("email").value;

            const response =
                await fetch("/api/users", {

                    method: "POST",

                    headers: {
                        "Content-Type": "application/json"
                    },

                    body: JSON.stringify({
                        name,
                        email
                    })
                });

            const data =
                await response.json();

            alert(data.message || data.error);

            loadUsers();
        }


        checkHealth();
        loadUsers();

    </script>

</body>
</html>
```

------------------------------------------------------------------------

# 19. Add Start Script

Open:

``` bash
nano package.json
```

Change the scripts section to:

``` json
"scripts": {
    "start": "node server.js",
    "dev": "nodemon server.js"
}
```

------------------------------------------------------------------------

# 20. Test Node.js Locally

Start application:

``` bash
npm start
```

Expected:

``` text
Server running on port 3000
```

Open another SSH session and test:

``` bash
curl http://127.0.0.1:3000
```

Test API:

``` bash
curl http://127.0.0.1:3000/api/health
```

Expected:

``` json
{
    "status": "OK",
    "message": "Node.js and MySQL are connected"
}
```

Test users:

``` bash
curl http://127.0.0.1:3000/api/users
```

Stop Node.js:

``` text
CTRL + C
```

------------------------------------------------------------------------

# 21. Install Apache

``` bash
sudo apt install -y apache2
```

Check:

``` bash
sudo systemctl status apache2
```

Enable at boot:

``` bash
sudo systemctl enable apache2
```

Start:

``` bash
sudo systemctl start apache2
```

------------------------------------------------------------------------

# 22. Enable Apache Proxy Modules

Node.js will run on port 3000.

Apache will receive requests on port 80.

Enable proxy modules:

``` bash
sudo a2enmod proxy
sudo a2enmod proxy_http
sudo a2enmod headers
```

Restart Apache:

``` bash
sudo systemctl restart apache2
```

------------------------------------------------------------------------

# 23. Create Apache Virtual Host

Create configuration:

``` bash
sudo nano /etc/apache2/sites-available/aws-node-app.conf
```

Add:

``` apache
<VirtualHost *:80>

    ServerName YOUR_DOMAIN.com
    ServerAlias www.YOUR_DOMAIN.com

    ProxyPreserveHost On

    ProxyPass /api http://127.0.0.1:3000/api
    ProxyPassReverse /api http://127.0.0.1:3000/api

    ProxyPass / http://127.0.0.1:3000/
    ProxyPassReverse / http://127.0.0.1:3000/

    ErrorLog ${APACHE_LOG_DIR}/aws-node-app-error.log
    CustomLog ${APACHE_LOG_DIR}/aws-node-app-access.log combined

</VirtualHost>
```

If you do not have a domain yet, you can temporarily use the EC2 public
IP by removing or changing the `ServerName`/`ServerAlias` lines.

------------------------------------------------------------------------

# 24. Enable Apache Site

``` bash
sudo a2ensite aws-node-app.conf
```

Disable the default site:

``` bash
sudo a2dissite 000-default.conf
```

Check configuration:

``` bash
sudo apache2ctl configtest
```

Expected:

``` text
Syntax OK
```

Restart:

``` bash
sudo systemctl restart apache2
```

------------------------------------------------------------------------

# 25. Start Node.js Again

``` bash
cd ~/aws-node-app
npm start
```

Now visit:

``` text
http://YOUR_EC2_PUBLIC_IP
```

You should see:

``` text
AWS Node.js + MySQL
```

The request path is:

``` text
Browser
   ↓
EC2 Port 80
   ↓
Apache
   ↓
Node.js Port 3000
   ↓
MySQL Port 3306
```

------------------------------------------------------------------------

# 26. Important: Node.js Must Run Continuously

If you close SSH while running:

``` bash
npm start
```

Node.js can stop.

Do not use `npm start` directly for a production-style deployment.

Use PM2.

------------------------------------------------------------------------

# 27. Install PM2

Install globally:

``` bash
sudo npm install -g pm2
```

Check:

``` bash
pm2 -v
```

------------------------------------------------------------------------

# 28. Start Node.js with PM2

Go to project:

``` bash
cd ~/aws-node-app
```

Start:

``` bash
pm2 start server.js --name aws-node-app
```

Check:

``` bash
pm2 status
```

View logs:

``` bash
pm2 logs aws-node-app
```

Stop logs:

``` text
CTRL + C
```

------------------------------------------------------------------------

# 29. Configure PM2 Startup

Save processes:

``` bash
pm2 save
```

Generate startup command:

``` bash
pm2 startup
```

PM2 will display a command.

Copy and run the command it gives you.

Then:

``` bash
pm2 save
```

Now Node.js should automatically restart after an EC2 reboot.

------------------------------------------------------------------------

# 30. Test PM2

``` bash
pm2 status
```

Expected:

``` text
aws-node-app    online
```

Test:

``` bash
curl http://127.0.0.1:3000/api/health
```

Then:

``` bash
curl http://YOUR_EC2_PUBLIC_IP/api/health
```

------------------------------------------------------------------------

# 31. Check Apache

``` bash
sudo systemctl status apache2
```

Check configuration:

``` bash
sudo apache2ctl configtest
```

Check Apache access log:

``` bash
sudo tail -f /var/log/apache2/aws-node-app-access.log
```

Check errors:

``` bash
sudo tail -f /var/log/apache2/aws-node-app-error.log
```

------------------------------------------------------------------------

# 32. Check Node.js Logs

``` bash
pm2 logs aws-node-app
```

Only errors:

``` bash
pm2 logs aws-node-app --err
```

Show recent logs:

``` bash
pm2 logs aws-node-app --lines 100
```

------------------------------------------------------------------------

# 33. Check MySQL

``` bash
sudo systemctl status mysql
```

Test:

``` bash
mysql -u awsuser -p -e "SELECT COUNT(*) FROM awsapp.users;"
```

------------------------------------------------------------------------

# 34. Check All Three Services

Run:

``` bash
sudo systemctl status apache2
```

``` bash
pm2 status
```

``` bash
sudo systemctl status mysql
```

Expected:

``` text
Apache → active (running)
Node.js → online
MySQL → active (running)
```

------------------------------------------------------------------------

# 35. Port Check

Check listening ports:

``` bash
sudo ss -tulpn
```

You should see services similar to:

``` text
*:80
127.0.0.1:3000
127.0.0.1:3306
```

Important:

``` text
80     → Public Apache
3000   → Local Node.js
3306   → Local MySQL
```

Do not expose 3000 or 3306 publicly unless required.

------------------------------------------------------------------------

# 36. Optional: Configure Domain

Suppose your domain is:

``` text
example.com
```

Create an A record:

``` text
Type: A
Name: @
Value: YOUR_EC2_ELASTIC_IP
```

For www:

``` text
Type: A
Name: www
Value: YOUR_EC2_ELASTIC_IP
```

It is strongly recommended to use an **Elastic IP** rather than relying
on a changing EC2 public IPv4 address.

------------------------------------------------------------------------

# 37. Update Apache for Domain

Edit:

``` bash
sudo nano /etc/apache2/sites-available/aws-node-app.conf
```

Use:

``` apache
<VirtualHost *:80>

    ServerName example.com
    ServerAlias www.example.com

    ProxyPreserveHost On

    ProxyPass /api http://127.0.0.1:3000/api
    ProxyPassReverse /api http://127.0.0.1:3000/api

    ProxyPass / http://127.0.0.1:3000/
    ProxyPassReverse / http://127.0.0.1:3000/

    ErrorLog ${APACHE_LOG_DIR}/aws-node-app-error.log
    CustomLog ${APACHE_LOG_DIR}/aws-node-app-access.log combined

</VirtualHost>
```

Check:

``` bash
sudo apache2ctl configtest
```

Restart:

``` bash
sudo systemctl restart apache2
```

------------------------------------------------------------------------

# 38. Add HTTPS with Let's Encrypt

Install Certbot:

``` bash
sudo apt install -y certbot python3-certbot-apache
```

Run:

``` bash
sudo certbot --apache -d example.com -d www.example.com
```

Follow the prompts.

Test certificate renewal:

``` bash
sudo certbot renew --dry-run
```

After HTTPS:

``` text
https://example.com
```

The architecture becomes:

``` text
Internet
   |
HTTPS 443
   |
Apache
   |
Node.js 3000
   |
MySQL 3306
```

------------------------------------------------------------------------

# 39. Application Deployment from GitHub

If the application is in GitHub:

``` bash
cd ~
git clone https://github.com/YOUR_USERNAME/YOUR_REPOSITORY.git aws-node-app
```

Enter directory:

``` bash
cd aws-node-app
```

Install dependencies:

``` bash
npm ci
```

Create `.env`:

``` bash
nano .env
```

Add your production database settings.

Start with PM2:

``` bash
pm2 start server.js --name aws-node-app
```

Save:

``` bash
pm2 save
```

------------------------------------------------------------------------

# 40. Updating the Application

After pushing new code:

``` bash
cd ~/aws-node-app
```

Pull:

``` bash
git pull origin main
```

Install updated packages:

``` bash
npm ci
```

Restart:

``` bash
pm2 restart aws-node-app
```

Check:

``` bash
pm2 status
```

------------------------------------------------------------------------

# 41. Database Backup

Create a backup directory:

``` bash
mkdir -p ~/mysql-backups
```

Backup:

``` bash
mysqldump -u awsuser -p awsapp > ~/mysql-backups/awsapp.sql
```

Check:

``` bash
ls -lh ~/mysql-backups/
```

Restore:

``` bash
mysql -u awsuser -p awsapp < ~/mysql-backups/awsapp.sql
```

For production systems, use a proper backup strategy and store backups
outside the same EC2 instance.

------------------------------------------------------------------------

# 42. Useful MySQL Commands

Login:

``` bash
mysql -u awsuser -p
```

Show databases:

``` sql
SHOW DATABASES;
```

Select database:

``` sql
USE awsapp;
```

Show tables:

``` sql
SHOW TABLES;
```

Describe table:

``` sql
DESCRIBE users;
```

View data:

``` sql
SELECT * FROM users;
```

Exit:

``` sql
EXIT;
```

------------------------------------------------------------------------

# 43. Useful Linux Commands

Current directory:

``` bash
pwd
```

List files:

``` bash
ls -la
```

Disk usage:

``` bash
df -h
```

Memory:

``` bash
free -h
```

CPU/processes:

``` bash
top
```

Network ports:

``` bash
sudo ss -tulpn
```

IP:

``` bash
curl ifconfig.me
```

System uptime:

``` bash
uptime
```

------------------------------------------------------------------------

# 44. Service Management

Apache:

``` bash
sudo systemctl start apache2
sudo systemctl stop apache2
sudo systemctl restart apache2
sudo systemctl status apache2
```

MySQL:

``` bash
sudo systemctl start mysql
sudo systemctl stop mysql
sudo systemctl restart mysql
sudo systemctl status mysql
```

Node.js with PM2:

``` bash
pm2 start server.js --name aws-node-app
pm2 stop aws-node-app
pm2 restart aws-node-app
pm2 delete aws-node-app
pm2 status
pm2 logs aws-node-app
```

------------------------------------------------------------------------

# 45. Troubleshooting

## Problem: Website does not open

Check EC2 Security Group:

``` text
TCP 80 → allowed
```

Check Apache:

``` bash
sudo systemctl status apache2
```

Check port:

``` bash
sudo ss -tulpn | grep :80
```

Check Apache logs:

``` bash
sudo tail -50 /var/log/apache2/aws-node-app-error.log
```

------------------------------------------------------------------------

## Problem: Apache gives 502/503

Check Node.js:

``` bash
pm2 status
```

Test directly:

``` bash
curl http://127.0.0.1:3000
```

If this fails:

``` bash
pm2 logs aws-node-app
```

Restart:

``` bash
pm2 restart aws-node-app
```

------------------------------------------------------------------------

## Problem: MySQL connection failed

Check:

``` bash
sudo systemctl status mysql
```

Test:

``` bash
mysql -u awsuser -p awsapp
```

Check `.env`:

``` bash
cat .env
```

Make sure:

``` env
DB_HOST=127.0.0.1
DB_USER=awsuser
DB_PASSWORD=YOUR_PASSWORD
DB_NAME=awsapp
```

Restart Node.js after changing `.env`:

``` bash
pm2 restart aws-node-app
```

------------------------------------------------------------------------

## Problem: Port 3000 cannot be accessed from browser

This is expected if Node.js is bound to:

``` text
127.0.0.1
```

Test from EC2:

``` bash
curl http://127.0.0.1:3000
```

Users should access:

``` text
http://YOUR_DOMAIN
```

through Apache.

------------------------------------------------------------------------

## Problem: Apache configuration error

Run:

``` bash
sudo apache2ctl configtest
```

If you see:

``` text
Syntax error
```

check:

``` bash
sudo nano /etc/apache2/sites-available/aws-node-app.conf
```

After fixing:

``` bash
sudo apache2ctl configtest
sudo systemctl restart apache2
```

------------------------------------------------------------------------

# 46. Security Checklist

Before calling this a production deployment:

-   [ ] Use an Elastic IP
-   [ ] Allow SSH only from trusted IP addresses
-   [ ] Do not expose MySQL 3306 publicly
-   [ ] Do not expose Node.js 3000 publicly
-   [ ] Use HTTPS
-   [ ] Use a strong database password
-   [ ] Keep `.env` out of Git
-   [ ] Keep Ubuntu updated
-   [ ] Configure regular database backups
-   [ ] Monitor disk usage
-   [ ] Monitor memory usage
-   [ ] Use least-privilege database users
-   [ ] Consider AWS Systems Manager instead of public SSH
-   [ ] Consider AWS RDS for production MySQL
-   [ ] Consider separating application and database servers as traffic
    grows

------------------------------------------------------------------------

# 47. Complete Final Architecture

``` text
                         USERS
                           |
                           |
                    Internet / HTTPS
                           |
                           v
                  +----------------+
                  |   AWS EC2      |
                  | Ubuntu Server  |
                  |                |
                  |  Apache :80    |
                  |      |         |
                  |      v         |
                  | Node.js :3000  |
                  |      |         |
                  |      v         |
                  | MySQL :3306    |
                  +----------------+
```

Internal communication:

``` text
Browser
   |
   | HTTPS :443
   v
Apache
   |
   | Reverse Proxy
   v
Node.js
   |
   | MySQL connection
   v
MySQL
```

------------------------------------------------------------------------

# 48. Final Verification Commands

Run these commands one by one:

``` bash
sudo systemctl status apache2
```

``` bash
sudo systemctl status mysql
```

``` bash
pm2 status
```

``` bash
curl http://127.0.0.1:3000/api/health
```

``` bash
curl http://127.0.0.1/api/health
```

Then open:

``` text
http://YOUR_EC2_PUBLIC_IP
```

or:

``` text
https://YOUR_DOMAIN.com
```

Test:

``` text
1. Website loads
2. Database status says connected
3. Existing users appear
4. Add User works
5. Refresh page
6. New user remains in MySQL
```

------------------------------------------------------------------------

# 49. Student Practice Tasks

After completing the deployment, students should modify the project.

## Task 1 --- Add Phone Number

Add:

``` text
phone
```

to the MySQL table.

Update:

``` text
POST /api/users
GET /api/users
HTML form
```

------------------------------------------------------------------------

## Task 2 --- Create Products

Create:

``` sql
CREATE TABLE products (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(150) NOT NULL,
    price DECIMAL(10,2) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

Create APIs:

``` text
GET /api/products
POST /api/products
PUT /api/products/:id
DELETE /api/products/:id
```

------------------------------------------------------------------------

## Task 3 --- Login System

Create:

``` text
users
roles
authentication
sessions/JWT
```

Use password hashing.

Never store plain-text passwords.

------------------------------------------------------------------------

## Task 4 --- Admin Dashboard

Build:

``` text
Dashboard
Users
Products
Orders
Reports
Logout
```

------------------------------------------------------------------------

## Task 5 --- Deployment Automation

Create a deployment script:

``` bash
git pull
npm ci
pm2 restart aws-node-app
```

Then learn how to automate it with GitHub Actions.

------------------------------------------------------------------------

# 50. Important Production Note

Running Node.js, MySQL, and Apache on one EC2 instance is excellent for:

-   Training
-   Classroom projects
-   Small applications
-   Proof-of-concept projects
-   Learning AWS deployment
-   Development environments

For larger production workloads, a common architecture is:

``` text
                    Internet
                       |
                  Load Balancer
                       |
                Node.js EC2/ECS
                       |
                     RDS
                    MySQL
```

This separates the application and database layers and makes scaling,
backups, maintenance, and high availability easier.

------------------------------------------------------------------------

# 51. One-Page Command Cheat Sheet

## Server

``` bash
sudo apt update
sudo apt upgrade -y
```

## Node.js

``` bash
curl -fsSL https://deb.nodesource.com/setup_lts.x | sudo -E bash -
sudo apt install -y nodejs
node -v
npm -v
```

## MySQL

``` bash
sudo apt install -y mysql-server
sudo systemctl enable --now mysql
sudo mysql
```

## Node Project

``` bash
mkdir ~/aws-node-app
cd ~/aws-node-app
npm init -y
npm install express mysql2 dotenv
```

## Run Node

``` bash
npm start
```

## PM2

``` bash
sudo npm install -g pm2
pm2 start server.js --name aws-node-app
pm2 save
pm2 startup
```

## Apache

``` bash
sudo apt install -y apache2
sudo a2enmod proxy proxy_http headers
sudo apache2ctl configtest
sudo systemctl restart apache2
```

## Logs

``` bash
pm2 logs
sudo tail -f /var/log/apache2/aws-node-app-error.log
sudo journalctl -u mysql -f
```

## Database Backup

``` bash
mysqldump -u awsuser -p awsapp > awsapp.sql
```

------------------------------------------------------------------------

# Project Complete

You have now deployed:

``` text
AWS EC2
 ├── Ubuntu
 ├── Apache
 │    └── Reverse Proxy
 ├── Node.js
 │    └── Express API
 └── MySQL
      └── awsapp database
```

This gives students a complete hands-on AWS deployment project using a
single EC2 server.
