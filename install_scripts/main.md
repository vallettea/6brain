# This file describes how to install a 6element sensor under Raspbian(RPi2)
#### Note : You can run this automatically by typing `sh main.md`

## Update the system

    sudo apt-get update -y;
    sudo apt-get upgrade -y;
    sudo apt-get install -y git-core build-essential vim;

## Install some dependencies/utils
    
    sudo apt-get install -y psmisc libudev-dev bluetooth tcpdump iw;
    

## Install X11 and chromium
###### Note : copy the full code block, not just line per line


    uname -a | grep hypriot > /dev/null && \
    (curl -sSL https://github.com/hypriot/x11-on-HypriotOS/raw/master/install-x11-basics.sh | bash; \
    curl -sSL https://github.com/hypriot/x11-on-HypriotOS/raw/master/install-chromium-browser.sh | bash);

    wget http://ftp.us.debian.org/debian/pool/main/libg/libgcrypt11/libgcrypt11_1.5.0-5+deb7u3_armhf.deb; \
    wget http://launchpadlibrarian.net/218525709/chromium-browser_45.0.2454.85-0ubuntu0.14.04.1.1097_armhf.deb; \
    wget http://launchpadlibrarian.net/218525711/chromium-codecs-ffmpeg-extra_45.0.2454.85-0ubuntu0.14.04.1.1097_armhf.deb; \
    sudo dpkg -i libgcrypt11_1.5.0-5+deb7u3_armhf.deb; \
    sudo dpkg -i chromium-codecs-ffmpeg-extra_45.0.2454.85-0ubuntu0.14.04.1.1097_armhf.deb; \
    sudo dpkg -i chromium-browser_45.0.2454.85-0ubuntu0.14.04.1.1097_armhf.deb;
    rm libgcrypt11_1.5.0-5+deb7u3_armhf.deb chromium-browser_45.0.2454.85-0ubuntu0.14.04.1.1097_armhf.deb;
    rm chromium-codecs-ffmpeg-extra_45.0.2454.85-0ubuntu0.14.04.1.1097_armhf.deb;


## Install node and npm

    wget http://nodejs.org/dist/v4.2.3/node-v4.2.3-linux-armv7l.tar.gz
    tar xvzf node-v4.2.3-linux-armv7l.tar.gz
    sudo mkdir /opt/node
    sudo cp -r node-v4.2.3-linux-armv7l/* /opt/node
    sudo ln -s /opt/node/bin/node /usr/local/bin/node
    sudo ln -s /opt/node/bin/npm /usr/local/bin/npm
    sudo npm install npm@3.5.2 -g

## Install wvdial and configure it

    sudo apt-get install -y wvdial
    sudo sh -c "cat ./wvdial.conf > /etc/wvdial.conf"
    sudo mkdir /usr/lib/systemd/system/
    sudo touch /usr/lib/systemd/system/wvdial.service
    cat ./script_NAME.service | \
    sed 's/NAME/wvdial/' | \
    sed 's/COMMAND/\/usr\/bin\/wvdial 3G/' \
    > ~/wvdial.service.tmp;
    echo -e '\n[Unit]\nBefore=6brain.service\n' >> ~/wvdial.service.tmp
    sudo mv ~/wvdial.service.tmp /usr/lib/systemd/system/wvdial.service


## Install pppd (pptpd on debian)

    sudo apt-get install -y pptpd;
    sudo ln -s `which pptpd` `which pptpd | sed s/pptpd/pppd/`;
    sudo sh -c 'cat /etc/ppp/options | sed s/^auth/#auth/ > pppoptions.tmp && mv pppoptions.tmp /etc/ppp/options';
    sudo sh -c 'echo "replacedefaultroute" >> /etc/ppp/options';
    sudo sh -c 'echo "noipdefault" >> /etc/ppp/options';

## Remove wlan0 and wlan1 at startup (otherwise, it would wait infinitely)

    cat ./script_init_interfaces.sh | bash -
    
## Remove these packets for a good network conf

    sudo apt-get remove -y ifplugd dhcpcd5
    
## rotate screen and enhance usb current

    sudo sh -c "echo 'lcd_rotate=2' >> /boot/config.txt"
    sudo sh -c "echo 'max_usb_current=1' >> /boot/config.txt"

## Download the two scripts to change the ethernet mode

    cat ./script_to_dhcp.sh \
    > ~/Desktop/to_dhcp && \
    chmod 555 ~/Desktop/to_dhcp;
    
    cat ./script_to_static.sh \
    > ~/Desktop/to_static && \
    chmod 555 ~/Desktop/to_static;

## Add a reboot cron job and a reconnect cron

    ( crontab -l 2>/dev/null | grep -Fv ntpdate ; printf -- "0 5 * * * /sbin/reboot\n" ) | crontab
    ( crontab -l 2>/dev/null | grep -Fv ntpdate ; printf -- "5 * * * * /bin/systemctl restart wvdial\n" ) | crontab
    
## Install 6brain

    git clone https://github.com/anthill/6brain.git;
    cd 6brain;
    npm install || sudo npm install;
    cd ..;
    
### TODO BY HAND: create the PRIVATE.json.


## Make chromium start at startup

    mkdir ~/.config/autostart/;
    touch ~/.config/autostart/6bin.desktop;
    cat ./script_APPNAME.desktop | \
    sed 's/APPNAME/6bin/' | \
    sed 's/COMMAND/\/usr\/bin\/chromium-browser --kiosk --safebrowsing-disable-auto-update --bwsi --disable-background-networking --disable-extensions --disable-sync --no-experiments --no-first-run --no-pings --disable-quic --disable-infobars 127.0.0.1:3000/' \
    > ~/.config/autostart/6bin.desktop;

## Make 6brain start at startup

    sudo mkdir /usr/lib/systemd/system/
    sudo touch /usr/lib/systemd/system/6brain.service
    cat ./script_NAME.service | \
    sed 's/NAME/6brain/' | \
    sed 's/COMMAND/\/usr\/local\/bin\/node \/home\/pi\/6brain\/index.js/' \
    > ~/6brain.service.tmp;
    sudo mv ~/6brain.service.tmp /usr/lib/systemd/system/6brain.service
    sudo systemctl enable 6brain
    
## Make the PRIVATE filler start at sartup

    cat ./script_NAME.service | \
    grep -v Restart | \
    sed 's/root/pi/' | \
    sed 's/NAME/GetSimId/' | \
    sed 's/COMMAND/\/usr\/local\/bin\/node \/home\/pi\/6brain\/install_scripts\/getSimId.js/' \
    > ~/getSimId.service.tmp;
    echo -e '\n[Unit]\nBefore=wvdial.service\n' >> ~/getSimId.service.tmp;
    sudo mv ~/getSimId.service.tmp /usr/lib/systemd/system/getSimId.service
    sudo systemctl enable getSimId

### TODO BY HAND: create a ssh profile for sensorSSH

    mkdir -p /root/.ssh
    echo -e "Host kerrigan\nUser sensorSSH\nHostname 62.210.245.148\nPort 9999\nStrictHostKeyChecking no\n" >> /root/.ssh/config
    echo "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQCynvUCNh7m3P3zHYKpRkx2iF4zywVWc10Ykyq171NelWBP5y7YUvvd+/cCf4ZIhn1YuLRmbdAjscwEnkQm239FPutIWOZbuRIAoGZP8Fgcx8gzDx4/CQ2dydMErI3J+jen1JX6oSG8Q1DPgvHBcKN6cW2t976oHnTsM8eX5Zy2I1HB3RsE4XMZaYdLbtwEhb349txH1J3oE9YmVkqVWvDocyvoy6rTMnZlVEwhVGMgB39uA32ife8e06upoDBpJWWFlvDPKMkEpQJBtQHGjhGhQyydyB1t1B8yWVrpO+DR+3Q8I8egHmj8z3acvN8e3YG7QVt1dxYCQctkWz7UExA7JvLM6rFCDQUOSSCcVskcdZ9WjKf7EBFhquzXTqtj59V7fIfwDEAhtlXIfxqyxmrC12OSaM7AON094P4VaR3+flxkEo23REG0cLWIlPRyMHTE8v0Epv2+z+YbATwNIWj1ZmlxKmjjH/UiyZ5ipIPlzWK1spQG9a6OA52lGgsSnW0Bl5c/kImICDCF9as96jQNX2E5r1KMhUH3g2IvJGsOQK8xvovk+v2NYaDLaoViqBb8Pe7akDigYHf+dSRET93Ek0vrNTw61qjyavLGkDJlLW4Oxe/41QIwGbeoHSaoL6AmF+c99fyp4R6//C7jJvgB7DdjZhHUlRv1zqjIq5JKdw== sensorSSH@antsM1" >> /root/.ssh/authorized_keys
    ssh-keygen -t rsa
    # add sensor id_rsa.pub to server
    

### TODO BY HAND : Change the RPi password

### TODO BY HAND : Change the hostname

### TODO BY HAND : Add your id_rsa.pub to know_hosts
