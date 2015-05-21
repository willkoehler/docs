# Setup rmate for editing remote server files with TextMate

### Install rmate on the server

    curl -Lo /usr/bin/rmate https://raw.github.com/aurora/rmate/master/rmate
    chmod +x /usr/bin/rmate

### Configure TextMate to accept rmate connections.

Check the "Allow rmate connections" box in the "Terminal" settings in TextMate preferences.

### Setup a tunnel between the dev system and the server

Run on the dev system. Leave running in an open terminal window.

    ssh -R 52698:localhost:52698 your_server

### Edit/create files on the server using rmate

    rmate file_you_want_to_edit

### Read more

<https://github.com/aurora/rmate>