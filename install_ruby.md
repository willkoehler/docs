# Install Ruby from scratch on Mac OS X

This has been tested on OS X Sierra with Ruby 2.3.3

## Update some libraries needed by Ruby.

OS X has old versions of readline and openssl. libyaml is not installed in OS X
Install Homebrew first if needed. <http://mxcl.github.io/homebrew/>

    brew update
    brew install readline openssl libyaml

## Download Ruby source, build and install

It's ok to ignore the warning in the configure step:
`WARNING: unrecognized options: --with-openssl-dir, --with-readline-dir`

    cd /tmp
    curl -O https://cache.ruby-lang.org/pub/ruby/2.3/ruby-2.3.3.tar.gz
    tar -xvzf ruby-2.3.3.tar.gz
    cd ruby-2.3.3/
    ./configure --prefix=/usr/local/ruby --disable-install-doc --with-openssl-dir=`brew --prefix openssl` --with-readline-dir=`brew --prefix readline`
    make
    sudo make install
    
## Setup path to the Ruby you just installed

Edit /etc/paths and add `/usr/local/ruby/bin` to the first line so the new version
of ruby is the default. Exit and open the terminal window to apply changes.

## Verify Ruby is working and version is 2.3.3

    ruby -v

## Update gem to latest version and install Bundler

    sudo gem update --system --no-document
    sudo gem install bundler --no-document

## Fix for rubygems certificate problems.

When running `bundle` or `gem install` you may see errors like

`Unable to download data from https://rubygems.org/ - SSL_connect returned=1 errno=0 state=SSLv3
read server certificate B: certificate verify failed`

or

`Gem::RemoteFetcher::FetchError: SSL_connect returned=1 errno=0 state=SSLv3 read server certificate
B: certificate verify failed`

To fix this you need to update the security certificates used by OpenSSL. The easiest way to do this
is to install a small script that syncs a homebrew installed OpenSSL CA pem with the certificates
found in the OSX Keychain.

    brew tap raggi/ale
    brew install openssl-osx-ca
    openssl-osx-ca

See <https://github.com/raggi/openssl-osx-ca> for more details

## Fix for errors when building Nokogiri gem

When bundler installs the Nokogiri gem you may see errors like `An error occurred while installing nokogiri
(1.6.6.2), and Bundler cannot continue` along with `libxml2 version 2.6.21 or later is required!` or
`The file "/usr/include/iconv.h" is missing in your build environment`

To fix this configure build settings for Nokogiri. Note that the `with-xml2-include` path is specific
to El Capitan and this will break on a future OS update

    bundle config build.nokogiri --use-system-libraries=true --with-xml2-include=/Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX10.12.sdk/usr/include/libxml2

## Alternate Instructions

chruby has also documented installation of Ruby from source. These may be better / more up
to date than my instructions.

<https://github.com/postmodern/chruby/wiki/Ruby>

