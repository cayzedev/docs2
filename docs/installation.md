---
sidebar_position: 1
---

# Install Skyport

| Operating System | Version |     Supported      | Notes                                                       |
|------------------|---------|:------------------:|-------------------------------------------------------------|
| **Ubuntu**       | 18.04 LTS  | :warning:          | Not recommended due to it being 6 years old now.            |
|                  | 20.04 LTS  | :warning:          | Not recommended due to it being 4 years old now.            |
|                  | 22.04 LTS  | :white_check_mark: | Documentation written assuming Ubuntu 22.04 as the base OS. |
|                  | 24.04 LTS  | :white_check_mark: |                                                             |
| **CentOS**       | Linux 7       | :white_check_mark: |                                                             |
|                  | Linux 8       | :white_check_mark: |      
|                  | Stream 8/9 | :white_check_mark:   |                                                             |
| **Debian**       | 11      | :white_check_mark:      |                                                             |
|                  | 12      | :white_check_mark:      |                                                             |
| **Windows**      | 10 23H2      | :white_check_mark: |                                                            |
|                  | 11 23H2      | :white_check_mark: |                                                            |
| **macOS**        | 11 Big Sur  | :white_check_mark:  |                                                            |
|                  | 12 Monterey | :white_check_mark:  |                                                             |
|                  | 13 Ventura  | :white_check_mark:  |                                                             |
|                  | 14 Sonoma   | :white_check_mark:  |                                                             |

### What you'll need / Dependencies

- [Node.js](https://nodejs.org/en/download/) version 20.0 or above
- [Nginx](https://nginx.org/en/)

### Example Dependency Installation

The commands below are simply an example of how you might install these dependencies on Ubuntu/Debian and CentOS. Please consult with your
operating system's package manager to determine the correct packages to install.

``` bash
# Ubuntu/Debian
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://deb.nodesource.com/gpgkey/nodesource-repo.gpg.key | sudo gpg --dearmor -o /etc/apt/keyrings/nodesource.gpg
echo "deb [signed-by=/etc/apt/keyrings/nodesource.gpg] https://deb.nodesource.com/node_20.x nodistro main" | sudo tee /etc/apt/sources.list.d/nodesource.list

sudo apt update
sudo apt install -y nodejs nginx

# CentOS
sudo yum install https://rpm.nodesource.com/pub_20.x/nodistro/repo/nodesource-release-nodistro-1.noarch.rpm -y
sudo yum install nodejs nginx -y
```

### Download Skyport

You'll now have all of the dependencies installed. To confirm this, check that Nginx is running with `systemctl status nginx` and that you've got Node.js installed - run `node -v`.

To download Skyport, run the following commands:
``` bash
cd /etc

git clone https://github.com/skyport/panel # You can swap this out for a release if you don't want to use the canary / dev version
mv panel skyport

cd skyport
```

Skyport will now downloaded in the `/etc/skyport` directory. If you would like to change the port, you can do so now by editing the `config.json` file and modifying the value for `port`.

### Almost done

To get Skyport running, run the following commands.

``` bash
# Install modules
npm install

# Install pm2 - this is if you want to keep your Skyport instance running 24/7, which most people will
npm install pm2

# IMPORTANT: Seed the Skyport images
npm run seed

# IMPORTANT: Create an admin account so you can log in
npm run createUser

# You may need to open a new terminal for pm2 to work
pm2 start index.js

# or if you want to test that it works:
node .
```

You are now done! Skyport is accessible on the port that you chose in the `config.json` - it is `3000` by default if you didn't change it.

### Proxy configuration

``` bash
# Remove default Nginx config
rm /etc/nginx/sites-enabled/default

# For SSL certificate - this is different if you are using Cloudflare Origin SSL
apt install certbot -y # or yum for CentOS, you'll need to run: yum install epel-release before doing it

certbot certonly -d <domain>
```

Edit the following file and put it into `/etc/nginx/sites-enabled/skyport.conf`:
``` nginx
server {
	listen 80;
	server_name <domain>;
	return 301 https://$server_name$request_uri;
}

server {
	listen 443 ssl http2;

	location /console {
		proxy_http_version 1.1;
		proxy_set_header Upgrade $http_upgrade;
		proxy_set_header Connection "upgrade";
		proxy_pass "http://localhost:<port>/console";
	}

	location /stats {
		proxy_http_version 1.1;
		proxy_set_header Upgrade $http_upgrade;
		proxy_set_header Connection "upgrade";
		proxy_pass "http://localhost:<port>/stats";
	}

	server_name <domain>;
	ssl_certificate /etc/letsencrypt/live/<domain>/fullchain.pem;
	ssl_certificate_key /etc/letsencrypt/live/<domain>/privkey.pem;
	ssl_session_cache shared:SSL:10m;
	ssl_protocols SSLv3 TLSv1 TLSv1.1 TLSv1.2;
	ssl_ciphers HIGH:!aNULL:!MD5;
	ssl_prefer_server_ciphers on;

	location / {
		proxy_pass http://localhost:<port>/;
		proxy_buffering off;
		proxy_set_header X-Real-IP $remote_addr;
	}
}
```