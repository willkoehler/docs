# Setup SSL certificate for a Rails app using RapidSSLOnline and OpsWorks

### 1. Generate a private SSL key

    openssl genrsa -des3 -out domain.com.key 2048

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
domain, include the `www` i.e. `www.domain.com`. For wildcard certificates, use
a `*` i.e. `*.domain.com`

### 4. Request SSL certificate from RapidSSLOnline

Make sure to copy and paste the full CSR into the request including the header
"`-----BEGIN CERTIFICATE REQUEST-----`" and footer "`-----END CERTIFICATE REQUEST-----`"

- Choose "File Based Authentication". **NOTE:** Make sure to put the file `fileauth.txt`
  at `/.well-known/pki-validation/fileauth.txt`, not at the server root.
- Select "Apache 2" for the Server Type.
- Select "SHA2" for the Signature Algorithm

### 5. Download the certificate bundle and append RapidSSLOnline chain file to the crt file

The certificate bundle will contain three files: `ServerCertificate.cer`,
`CACertificate-INTERMEDIATE.cer`, and `CACertificate-ROOT.cer`. The last two are the SSL
chain files. They link your certificate back to the original certificate authority. Without
the chain files, browsers will not trust your certificate. The root certificate should already
be in the user's browser / OS and should not be included in your certificate chain. However
the intermediate certificate will need to be included in your certificate chain.

Since Nginx doesn't have a way to specify the SSL chain files in the configuration,
you must append the SSL chain files to the end of your certificate. Copy and paste
the contents of the intermediate certificate `CACertificate-INTERMEDIATE.cer` to the end
of `ServerCertificate.cer`.

rename `ServerCertificate.cer` to `domain.com.crt`

### 6. Add the SSL .key and .crt files to the OpsWorks App config

Enable SSL in the application config and put the SSL certificate and SSL key in the
corresponding fields in the SSL Settings section of the application config.

Deploy the app to apply changes.

### 7. Configure Rails app to use SSL connection

Tell rails to redirect all requests to SSL and use secure cookies. Edit
`config/environments/production.rb` and enable SSL. (Requires Rails 3.1)

    config.force_ssl = true

### 8. Test the SSL certificate

<https://www.ssllabs.com/ssltest/>
