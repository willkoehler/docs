# Install Ruby from scratch on Mac OS X

### Download libyaml and install. This is a prerequisite for Ruby

    curl -O http://pyyaml.org/download/libyaml/yaml-0.1.4.tar.gz
    tar xzvf yaml-0.1.4.tar.gz
    cd yaml-0.1.4
    ./configure --prefix=/usr/local
    make
    sudo make install

### Download Ruby source, build and install

    curl -O http://ftp.ruby-lang.org//pub/ruby/1.9/ruby-1.9.3-p194.tar.gz
    tar -xvzf ruby-1.9.3-p194.tar.gz
    cd ruby-1.9.3-p194/
    ./configure --prefix=/usr/local/ruby --disable-install-doc
    make
    sudo make install
    
### Setup path to the Ruby you just installed

Edit /etc/paths and add `/usr/local/ruby/bin` to the first line so the new version
of ruby is the default. Exit and open the terminal window to apply changes.

### Verify Ruby is working and version is 1.9.3p194

    ruby -v

### Update gem to latest version and install Bundler

    sudo gem update --system
    sudo gem install bundler

### Build rdoc documentation (optional)

This is an optional step if you want the rdoc documentation. Looks like the makefile only
builds ri docs. The only option for building html-formatted docs is manually running rdoc.
First need to remove the empty doc folder.

    sudo rm -r /usr/local/ruby/share/doc
    sudo ./bin/rdoc --no-force-update --all --op /usr/local/ruby/share/doc

# Uninstall all gems on the system

A pretty slick technique to uninstall all gems in case it's ever needed.

    gem list --no-version | xargs gem uninstall -aIx

<http://jamespmcgrath.com/how-to-uninstall-all-ruby-gems-in-one-line/>
