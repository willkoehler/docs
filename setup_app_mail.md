# Setup Amazon SES and Google apps to provide email services for a Rails app.
Use Google Apps to host email for your application's domain. This allows you to receive
replies from your web app's emails and to correspond with your users. Use Amazon SES to send
email from within your application.

### 1. Sign up for a free Google Apps account for your domain
<http://www.google.com/apps/intl/en/group/index.html><br>

### 2. Verify you own the domain
Follow the steps provided by Google Apps.

### 3. Setup MX records for the domain
Using your Domain Manager (ex: GoDaddy), delete existing MX records and add the following MX
records to the domain's zone file.

<table border=1 cellspacing=0 cellpadding=3>
  <tr>
    <td width=50><b>Priority</b></td>
    <td width=50><b>Host</b></td>
    <td width=250><b>Points to</b></td>
    <td><b>TTL</b></td>
  </tr>
  <tr>
    <td>10</td>
    <td>@</td>
    <td>ASPMX.L.GOOGLE.COM</td>
    <td>1 Week</td>
  </tr>
  <tr>
    <td>20</td>
    <td>@</td>
    <td>ALT1.ASPMX.L.GOOGLE.COM</td>
    <td>1 Week</td>
  </tr>
  <tr>
    <td>30</td>
    <td>@</td>
    <td>ALT2.ASPMX.L.GOOGLE.COM</td>
    <td>1 Week</td>
  </tr>
  <tr>
    <td>40</td>
    <td>@</td>
    <td>ASPMX2.GOOGLEMAIL.COM</td>
    <td>1 Week</td>
  </tr>
  <tr>
    <td>50</td>
    <td>@</td>
    <td>ASPMX3.GOOGLEMAIL.COM</td>
    <td>1 Week</td>
  </tr>
</table>

### 4. Enable email in Google Apps
Google Apps Dashboard page / Service Settings section / Email. Enable email and inform Google
Apps the the MX records are setup. It will take Google Apps will take a few
minutes to verify your MX records after you've enabled email.

### 5. Setup a SPF record for the domain
Using your Domain Manager, add the following TXT record to the domain's zone file.
NOTE: some DNS providers, such as Route 53, allow you to setup true SPF records. However Gmail
ignores true SPF records. To be safe use TXT records. It appears true SPF records are not
widely used, or supported.

    "v=spf1 a mx include:amazonses.com include:_spf.google.com -all"

SPF records are used by email recipients to authenticate the source of emails. SPF records
list which IP addresses & domains are authorized to send email on behalf of your domain.
SPF records are stored as specially formatted TXT records in the domain's zone file.
The record above authorizes Amazon SES (include:amazonses.com) and Google Apps
(include:_spf.google.com) to send mail on behalf of the domain. It also authorizes the
server (a) and servers defined by the MX records (mx) to send email, although we don't
use these.

### 7. Setup a reply-to email
* Setup a new user in Google Apps to receive email replies. (Or setup an alias
for the primary administrator account.)
* Use the AWS Management Console to register the email address as a SES Verified Sender.
This allows you to use the email as the "from" address when sending emails via SES

### 8. Setup your application to use Amazon SES
Make changes to the following files.

`Gemfile`

    gem "aws-ses", :require => 'aws/ses'

`config/initializers/setup_mail.rb`

    # Setup Amazon SES as a delivery option
    ActionMailer::Base.add_delivery_method :ses, AWS::SES::Base,
      :access_key_id => RUBBER_CONFIG.cloud_providers['aws'].access_key,
      :secret_access_key => RUBBER_CONFIG.cloud_providers['aws'].secret_access_key
    # From address needs to be registered with AWS as a SES Verified Sender
    ActionMailer::Base.default :from => "noreply@cmscontracts.com"

`config/environments/production.rb` and `config/environments/development.rb`

    # Use AWS SES to deliver emails
    config.action_mailer.delivery_method = :ses

### 9. Test the SPF records
Send an email to `check-auth2@verifier.port25.com` from both your Google Apps email
account and from your application via Amazon SES. You will receive a reply with the results of
the SPF check. It should be "pass". More information: <http://www.port25.com/support/authentication-center/email-verification/>
