# Renew GoDaddy SSL Certificate

When renewing a GoDaddy SSL certificate, you don't need to generate a new key or csr. GoDaddy
keeps the old csr on file and allows you to reuse it. You just need to renew the cerificate
with the old csr, download the new crt file and redeploy. Need to document the specific steps next
time I renew.

# Setup SSL certificate for a Rails app.

These instructions assume you're using Rubber to manage the app's server configuration

### 1. Generate a private SSL key

    openssl genrsa -des3 -out domain.com.key 2048

The name of the key should be yourappdomain.com.key This is the default key name that
Rubber expects. Make sure the domain name you use here matches the domain configured in
`config/rubber.yml`.

Enter a pass phase when requested. This will be cleared in the next step so
you don't need to remember it for long.

### 2. Clear pass phrase on SSL key

The pass phrase is intended to prevent someone from stealing your key and using it.
However if there is a pass phrase on the key, Nginx will ask you the pass phrase
each time it starts which will prevent Nginx from starting on reboot, deploy, etc.

    cp domain.com.key domain.com.key.secure
    openssl rsa -in domain.com.key.secure -out domain.com.key

### 3. Generate Certificate Signing Request (CSR)

    openssl req -new -sha256 -key domain.com.key -out domain.com.csr

You will be asked for several pieces of information. The most important is the
`Common Name`. This is your domain name. If the certificate is for a single
domain, include the www i.e. www.domain.com. For wildcard certificates, use
a * i.e. *.domain.com

### 4. Request SSL certificate from RapidSSLOnline or equivalent

Make sure to copy and paste the full CSR into the request including the header
"`-----BEGIN CERTIFICATE REQUEST-----`" and footer "`-----END CERTIFICATE REQUEST-----`"

Select "Apache 2" for the server type. 

### 5. Download the certificate bundle and append RapidSSLOnline chain file to the crt file

The certificate bundle will contain three files: `ServerCertificate.cer`, `CACertificate-INTERMEDIATE.cer`,
and `CACertificate-ROOT.cer`. The last two are the SSL chain files. They links your certificate back to the
original certificate authority. Without the chain file, some browsers will complain the certificate is invalid.

Since Nginx doesn't have a way to specify the SSL chain file in the configuration,
you must append the SSL chain files to the end of your certificate. Just copy and paste
the contents of the chain files to the end of `ServerCertificate.cer`. `CACertificate-ROOT.cer`
should be at the bottom.

rename `ServerCertificate.cer` to `domain.com.crt`

### 6. Copy key and crt files into app secrets

Place `domain.com.key` and the amended `domain.com.crt` into `config/secrets/certs`

### 7. Configure Nginx to use SSL certificate

Edit `config/rubber/role/passenger_nginx/passenger_nginx.conf` and
`config/rubber/role/web_tools/nginx-tools.conf` and update the SSL sections of the
files as shown below. There are three sections total.

    <% if rubber_env.use_ssl_key %>
      ssl_certificate <%= Rubber.root %>/config/secrets/certs/<%= rubber_env.domain %>.crt;
      ssl_certificate_key <%= Rubber.root %>/config/secrets/certs/<%= rubber_env.domain %>.key;
    <% else %>
      ssl_certificate  /etc/ssl/certs/ssl-cert-snakeoil.pem;
      ssl_certificate_key  /etc/ssl/private/ssl-cert-snakeoil.key;
    <% end %>
    
Edit `config/rubber/rubber-passenger_nginx.yml` and make sure SSL is enabled

    use_ssl_key: true

### 8. Configure Rails app to use SSL connection

Tell rails to redirect all requests to SSL and use secure cookies. Edit
`config/environments/production.rb` and enable SSL. (Requires Rails 3.1)

    config.force_ssl = true

### 9. Test the SSL certificate

<https://www.ssllabs.com/ssltest/>
