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

    openssl req -new -key domain.com.key -out domain.com.csr

You will be asked for several pieces of information. The most important is the
`Common Name`. This is your domain name. If the certificate is for a single
domain, include the www i.e. www.domain.com. For wildcard certificates, use
a * i.e. *.domain.com

### 4. Request SSL certificate from GoDaddy or equivalent
Make sure to copy and paste the full CSR into the request including the header
"`-----BEGIN CERTIFICATE REQUEST-----`" and footer "`-----END CERTIFICATE REQUEST-----`"

### 5. Download the certificate bundle and append GoDaddy SSL chain file to the crt file
Select Apache server type when downloading. The certificate bundle will contain two
files `domain.com.crt` and `gd_bundle.crt`. `gd_bundle.crt` is the SSL chain file.
It links your certificate back to the original certificate authority. Without the chain
file, some browsers will complain the certificate is invalid.

Since Nginx doesn't have a way to specify the SSL chain file in the configuration,
you must append the SSL chain file to the end of your certificate. Just copy and paste
the contents of `gd_bundle.crt` to the end of `domain.com.crt`.

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
<http://www.ssltest.net> or <https://www.ssllabs.com/ssldb/>
