# Setting Up Your Domain and Server with Cloudflare and Nginx

This guide will help you set up a domain, launch an EC2 instance, configure Cloudflare as a reverse proxy, and set up Nginx on your server.

## 1. Purchase a Domain
Purchase your own domain. For this guide, the domain was purchased from [domain.com](https://domain.com).
![Domain Purchase](assets/domain.png)

## 2. Launch an EC2 Instance and Assign an Elastic IP
1. Launch an EC2 instance on AWS.
2. Allocate and assign an Elastic IP address to your instance.
3. Open inbound traffic for the following ports:
   - **22**: SSH
   - **80**: HTTP
   - **443**: HTTPS

## 3. Introduction to Cloudflare

Cloudflare is a global cloud reverse proxy that offers services such as caching, DDoS mitigation, and more.

### What is a Reverse Proxy?

A reverse proxy forwards client requests to backend servers. It provides several benefits, including:
- **IP masking**: Hides the IP addresses of your servers.
- **DDoS attack mitigation**: Protects your servers from Distributed Denial of Service (DDoS) attacks.
- **Caching**: Stores web pages and static content, reducing the delay in serving pages when located near clients.

![Reverse Proxy Flow](assets/reverse-proxy.png)

## 4. Set Up Cloudflare

1. Register or log in to your Cloudflare account.
2. Add your site by entering the domain you purchased earlier.
3. Choose the **Free** plan.

## 5. Update Your Domain’s Nameservers

1. Go to **domain.com**.
2. Navigate to **Domains -> Your Domain -> Advanced Tools -> Manage**.
3. Set the nameservers to Cloudflare’s nameservers.

![Nameservers](assets/nameservers.png)

### Verify the Reverse Proxy Setup

To verify the reverse proxy setup, use the following command in your terminal:
```bash
nslookup your_domain.com
```

![NSLookup](assets/nslookup.png)

The command will show the Cloudflare IP addresses. Your computer may choose any of the IPs for the connection.

## 6. Set A-Records on Cloudflare

In Cloudflare, set the A-records to point to your EC2 instance’s Elastic IP address.

1. Go to **DNS -> DNS management** for your domain.
2. Add an A-record for your domain pointing to the Elastic IP of your EC2 instance.

![A-Records](assets/a-records.png)

## 7. Install and Configure Nginx on Your EC2 Instance

### 7.1 Install Nginx

1. Update the package list and install Nginx:
   ```bash
   sudo apt update
   sudo apt install nginx -y
   ```
2. Verify that Nginx is running:
   ```bash
   systemctl status nginx
   ```
   You should see output similar to:
   ```
   ● nginx.service - A high performance web server and a reverse proxy server
      Active: active (running)
   ```

### 7.2 Check Web Server
Visit `http://your_server_ip` in a browser to verify that the Nginx server is up and running. HTTPS is not set up yet.

![HTTP Nginx Test](assets/http.png)

### 7.3 Set Up Server Blocks for Your Website

If you want to host multiple websites, you can create multiple Nginx server blocks. Here’s how to set up a server block for your domain.

1. Create the directory for your website:
   ```bash
   sudo mkdir -p /var/www/your_domain/html
   ```
2. Set directory ownership:
   ```bash
   sudo chown -R $USER:$USER /var/www/your_domain/html
   ```
3. Adjust directory permissions:
   ```bash
   sudo chmod -R 755 /var/www/your_domain
   ```
4. Create a simple `index.html` file:
   ```bash
   sudo nano /var/www/your_domain/html/index.html
   ```
   Add the following HTML:
   ```html
   <html>
       <head>
           <title>Welcome to your_domain!</title>
       </head>
       <body>
           <h1>Success! The your_domain server block is working!</h1>
       </body>
   </html>
   ```

### 7.4 Configure Nginx

Create a new configuration file for your domain:

1. Open a new configuration file for your domain:
   ```bash
   sudo nano /etc/nginx/sites-available/your_domain
   ```
2. Paste the following configuration:
   ```bash
   server {
       listen 80;
       listen [::]:80;

       root /var/www/your_domain/html;
       index index.html index.htm;

       server_name your_domain www.your_domain;

       location / {
           try_files $uri $uri/ =404;
       }
   }
   ```

3. Enable the configuration:
   ```bash
   sudo ln -s /etc/nginx/sites-available/your_domain /etc/nginx/sites-enabled/
   ```
4. Test for syntax errors:
   ```bash
   sudo nginx -t
   ```
5. Restart Nginx to apply the changes:
   ```bash
   sudo systemctl restart nginx
   ```

You can now access your site at `http://your_domain`.

## 8. Cloudflare SSL Modes of Operation

Cloudflare offers several SSL modes:

- **Full (Strict)**: End-to-end encryption with certificate validation.
- **Full**: End-to-end encryption without certificate validation.
- **Flexible**: Encryption between visitors and Cloudflare only.

To secure the connection between Cloudflare and your server, switch to **Full (Strict)** mode and configure SSL certificates.

### 8.1 Enable SSL on Nginx (Full/Full Strict Mode)

1. Open your Nginx configuration file:
   ```bash
   sudo nano /etc/nginx/sites-available/your_domain
   ```
2. Add SSL configuration:
   ```bash
   server {
       listen 443 ssl;
       ssl_certificate /etc/ssl/your_domain.pem;
       ssl_certificate_key /etc/ssl/your_domain.key;
       ...
   }
   ```
3. Create an origin certificate on Cloudflare:
   - Go to **SSL/TLS -> Origin Server**.
   - Generate a certificate and copy it to `/etc/ssl/your_domain.pem` and your private key to `/etc/ssl/your_domain.key`.
4. Reload Nginx:
   ```bash
   sudo systemctl restart nginx
   ```

Your website should now work with HTTPS in **Full (Strict)** mode.

![HTTPS Connection](assets/https.png)

### 8.2 Self-Signed Certificate

1. Disconnect Cloudflare - switch from Cloudflare nameservers to AWS nameservers.

Create a Route 53 Hosted Zone and copy the nameservers from it to your domain's advanced settings.

Add records to point your domain to your EC2 instance:
![Route 53 Hosted Zone Records](assets/route53-hosted-zone.png)

Open `https://your_domain` and you will encounter this error:
![Https error](assets/https-error.png)

This error occurs because the self-signed certificate is not trusted by your computer since Cloudflare is disabled for your website.

The certificate is valid for your domain and its subdomains, but due to the absence of a root CA and intermediate CA in the chain, the web browser is unable to verify the certificate successfully.
![Invalid certificate](assets/invalid-cert.png)

#### Create a Self-Signed Certificate using OpenSSL

1. **CSR - Certificate Signing Request**  
   A CSR (Certificate Signing Request) is required when generating an SSL certificate. It consists of two parts:
   - **Subject Name**: Identifies the domain.
   - **Public Key**: Used to encrypt data.

   **Recommendation:**  
   Choose the CN (Common Name) according to the main domain where the certificate will be used. It could be a wildcard (`*.your_domain`) or just your domain.

2. **Delete the old certificate (Origin certificate from Cloudflare) along with its private key.**

Go to `/etc/ssl` and run the following command to create a self-signed certificate:
```bash
openssl req -x509 -newkey rsa:4096 -keyout your_domain.com.key -out your_domain.com.pem -days 365
```
![Creating self-signed cert](assets/create-s-s-cert.png)

3. **Create a passphrase file**  
   Create a file named `passphrase.pass` and store your passphrase in it.

4. **Include the passphrase file in the Nginx configuration using `ssl_password_file`:**
```nginx
server {

    listen   443;
    listen   [::]:443;

    ssl    on;
    ssl_certificate    /etc/ssl/your_domain.com.pem;
    ssl_certificate_key    /etc/ssl/your_domain.com.key;
    ssl_password_file /etc/ssl/passphrase.pass;

    server_name your_domain.com www.your_domain.com;

    root /var/www/your_domain.com/html;

    # Default file to serve
    index index.html index.htm;

    # Serve static files (images, CSS, JS, etc.)
    location / {
        try_files $uri $uri/ =404;
    }
} 
```

Here is the self-signed certificate:
![Self-signed Certificate](assets/sscert.png)

This certificate is not trusted by browsers, but it can be used with Cloudflare.

5. **Switch back to Cloudflare nameservers.**

- In **Full (Strict)** mode:
![Self-sign Cert in Full Strict Mode](assets/ssc-full-strict.png)

- In **Full** mode:
![Self-sign Cert in Full Mode](assets/ssc-full.png)

**Note:** A self-signed certificate cannot be directly served to the browser. However, it can be used between servers, such as between your web server and Cloudflare servers.
