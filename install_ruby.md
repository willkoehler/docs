# Install Ruby from scratch on Mac OS X

## Update some libraries needed by Ruby.

OS X has old versions of readline and openssl. libyaml is not installed in OS X
Install Homebrew first if needed. <http://mxcl.github.io/homebrew/>

    brew update
    brew install readline openssl libyaml

## Download Ruby source, build and install

It's ok to ignore the warning in the configure step:
`WARNING: unrecognized options: --with-openssl-dir, --with-readline-dir`

    curl -O http://ftp.ruby-lang.org/pub/ruby/2.0/ruby-2.0.0-p247.tar.gz
    tar -xvzf ruby-2.0.0-p247.tar.gz
    cd ruby-2.0.0-p247/
    ./configure --prefix=/usr/local/ruby --disable-install-doc --with-openssl-dir=`brew --prefix openssl` --with-readline-dir=`brew --prefix readline`
    make
    sudo make install
    
## Setup path to the Ruby you just installed

Edit /etc/paths and add `/usr/local/ruby/bin` to the first line so the new version
of ruby is the default. Exit and open the terminal window to apply changes.

## Verify Ruby is working and version is 2.0.0p247

    ruby -v

## Update gem to latest version and install Bundler

    sudo gem update --system
    sudo gem install bundler

## Build rdoc documentation (optional)

This is an optional step if you want the rdoc documentation. Looks like the makefile only
builds ri docs. The only option for building html-formatted docs is manually running rdoc.
First need to remove the empty doc folder.

    sudo rm -r /usr/local/ruby/share/doc
    sudo ./bin/rdoc --no-force-update --all --op /usr/local/ruby/share/doc
