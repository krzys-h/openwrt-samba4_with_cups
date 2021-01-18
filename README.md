# samba4 with CUPS support for OpenWRT
Note: This is mostly just a reference of how I did it. Feel free to send pull requests with improvements.

## Building and installation
```sh
# Download the OpenWRT SDK
# (you may need to select a different one, matching your device - the remaining steps are the same for all of them)
wget https://downloads.openwrt.org/releases/19.07.5/targets/ath79/generic/openwrt-sdk-19.07.5-ath79-generic_gcc-7.5.0_musl.Linux-x86_64.tar.xz
tar xvf openwrt-sdk-19.07.5-ath79-generic_gcc-7.5.0_musl.Linux-x86_64.tar.xz
cd openwrt-sdk-19.07.5-ath79-generic_gcc-7.5.0_musl.Linux-x86_64.tar.xz

# Add the package feed
echo "src-git samba4_with_cups https://github.com/krzys-h/openwrt-samba4_with_cups.git" >> feeds.conf.default
# Alternatively, for a local directory:
echo "src-link samba4_with_cups /absolute/path/to/samba4_with_cups" >> feeds.conf.default

# TODO: You have to use the local directory variant and patch the absolute path to python3 makefile in samba4/Makefile ... i can't get the path to work otherwise due to symlinks...

# Update the package index, making sure that we use our version of samba4 rather than the builtin one
# I'm not sure which one would be chosen otherwise
# (note: cups is not packaged in the default repositories at all, so we don't have to worry about it)
./scripts/feeds update -a
./scripts/feeds install -a
./scripts/feeds uninstall samba4
./scripts/feeds install -p samba4_with_cups samba4

# Enable the packages
make menuconfig
# In the UI:
# Network > Printing > cups, set to M (module)
# Network > samba4, set to M (module)
# Select Save and then Exit

# Build
# (this will take a while - it needs to build all dependencies as well)
make -j9 package/cups/compile package/samba4/compile

# Upload to device
scp bin/packages/mips_24kc/samba4_with_cups/* root@openwrt:.

# Install the packages
ssh root@router
opkg remove samba4-libs samba4-server  # if you have the normal version already installed
opkg install libcups_2.3.0-2_mips_24kc.ipk
opkg install cups_2.3.0-2_mips_24kc.ipk
opkg install samba4-libs_4.11.17-1_mips_24kc.ipk
opkg install samba4-server_4.11.17-1_mips_24kc.ipk

# Create the spooler and driver dirs
mkdir -p /srv/samba/print_drivers
mkdir -p /srv/samba/print_spool
chmod 0777 /srv/samba/print_spool

# Go to http://openwrt:631/ and configure your printer. You'll probably want to use the default
# raw queue as routers generally don't have enough processing power to render jobs
# (and we don't install anything else anyway)

# Remember to enable "printer sharing" if you intend to access it from CUPS clients

# Restart smbd after configuring the printer in CUPS. Everything else should be already set up.
/etc/init.d/samba4 restart

# Don't forget to create an user, or reconfigure samba for anonymous access!
```

# Configuring CUPS clients
Configure the printer using the following path: `ipp://openwrt:631/printers/printer_name_on_the_server`

You have to choose the driver manually.

# Configuring Windows clients
You should be able to navigate to `\\openwrt` and configure the printer using the normal Windows UI

You have to choose the driver manually. In theory it's possible to serve the driver using samba but I couldn't get it to work.

# References
https://github.com/TheMMcOfficial/lede-cups
https://github.com/openwrt/packages/tree/openwrt-19.07/net/samba4
