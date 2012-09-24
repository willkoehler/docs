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
    
### Build rdoc documentation.

This is an optional step if you want the rdoc documentation. Looks like the makefile only
builds ri docs. The only option for building html-formatted docs is manually running rdoc.
First need to remove the empty doc folder.

    sudo rm -r /usr/local/ruby/share/doc
    sudo ./bin/rdoc --no-force-update --all --op /usr/local/ruby/share/doc

### Setup path to the Ruby you just installed
Edit /etc/paths and add "/usr/local/ruby/bin" to first line so new version of ruby is the
default. Exit and open the terminal window to apply changes.

    mate /etc/paths

### Verify Ruby is working and version is 1.9.3p194

    ruby -v

### Update gem to latest version and install Bundler

    sudo gem update --system
    sudo gem install bundler
