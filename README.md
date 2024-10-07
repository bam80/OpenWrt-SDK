### Prerequisites
Container Engine and [Compose Spec](https://compose-spec.io/) tool of your choose:
- [Podman](https://podman.io/) (prefferable, [Installation](https://podman.io/getting-started/installation)), [Podman Compose](https://github.com/containers/podman-compose) ([Installation](https://github.com/containers/podman-compose#installation))  
or
- [Docker](https://docs.docker.com/engine/) ([Installation](https://docs.docker.com/engine/install/)), [Docker Compose V1](https://github.com/docker/compose) ([Installation](https://docs.docker.com/compose/install/#install-compose)) or [Docker Compose V2](https://docs.docker.com/compose/cli-command/) ([Installation](https://docs.docker.com/compose/cli-command/#install-on-linux))

### Usage
```sh
git clone https://invent.kde.org/butirsky/plasma-mobile-sdk.git && cd openwrt-kroks
podman-compose run --no-deps --rm openwrt
```
This will run interactive script to build firmware. Press a key when asked.

The built firmware will be placed in `bin/targets/ramips/mt76x8/`

###### NB:  
- If you prefer Docker, replace `podman` in the commands above with `docker`. For Docker Compose V2, use `docker compose` instead of `docker-compose`.
- Workaround for snapped Firefox's firmware image access problem:
  ```sh
  sudo chown bam bin/targets/ramips/mt76x8 bin/targets/ramips/mt76x8/openwrt-ramips-mt76x8-kroks_*-squashfs-sysupgrade.bin
  chmod g+w bin/targets/ramips/mt76x8 bin/targets/ramips/mt76x8/openwrt-ramips-mt76x8-kroks_*-squashfs-sysupgrade.bin 
  ```
- Find your router's IPv6 Link-Local Address:
  ```sh
  ip -6 neigh 
  # fe80::eef:afff:fecf:6fdf dev wlx060c0009f5ea lladdr 0c:ef:af:cf:6f:df router STALE

  # to parse:
  # lladdr=$(ip -6 neigh | cut -f 1,3 -d\  --output-delimiter=%)
  ```
  You can also use `rdisc6 <interface>` and `ping ip6-allnodes`

  It's recommended to save it in `/etc/hosts` under some name.

#### Flashing procedure:
- Get the router's full IPv6 Link-Local Address postfixed with `%` and the interface name and save it to a variable:
  ```sh
  lladdr=$(ping -c 1 modem | grep -oP '(?<=\().*(?=\):)') ; echo $lladdr
  # fe80::eef:afff:fecf:6fdf%wlx060c0009f5ea
  ```
  If it gives IPv4 address, that's OK, too.

  To check a name of the host currently in `$lladdr`:
  ```sh
  nslookup ${lladdr%\%*}
  # f.d.f.6.f.c.e.f.f.f.f.a.f.e.e.0.0.0.0.0.0.0.0.0.0.0.0.0.0.8.e.f.ip6.arpa	name = modem.
  ```
- Put image to the router:
  ```sh
  ssh root@$lladdr 'cat > /tmp/fw.bin' < ~/Downloads/openwrt-*kroks_*-squashfs-sysupgrade.bin
  ```
- (optional) Put configuration archive to the router:
  - If you have no LAN connection and there is no locally saved configuration yet, you might need to enable WiFi on the first boot before flashing firmware:  
  https://openwrt.org/docs/guide-user/installation/flashing_openwrt_with_wifi_enabled_on_first_boot
    ```sh
    tar -C files/ -czv etc/uci-defaults/97-enable_wifi | ssh root@$lladdr 'cat > /tmp/cf.tgz'
    ```
  - Alternatively, if you have locally saved configuration but the router is not configured yet:
    ```sh
    ssh root@$lladdr 'cat > /tmp/cf.tgz' < ~/Downloads/backup-OpenWrt-*
    ```
- Flashing the image:
  - If you passed configuration above, flash it along with the image:
    ```sh
    ssh root@$lladdr sysupgrade -n -f /tmp/cf.tgz /tmp/fw.bin
    ```
  - In other cases, just pass firmware image to the command, it will restore current configuration unless `-n` option is present:
    ```sh
    ssh root@$lladdr sysupgrade /tmp/fw.bin
    ```
  You might need forcing option first time:  
  > ```
  > 	-F | --force
  > 	             Flash image even if image checks fail, this is dangerous!
  > 	--ignore-minor-compat-version
  > 		     Flash image even if the minor compat version is incompatible.

#### Restoring procedure:
- (optional) Hard factory reset:  
  https://openwrt.org/docs/guide-user/troubleshooting/failsafe_and_factory_reset#jffs2_reset
  ```sh
  ssh -t root@$lladdr 'umount /overlay && jffs2reset -y && reboot'
  ```
- (optional) Configuration restore:
  ```sh
  ssh -o StrictHostKeyChecking=no root@$lladdr 'sysupgrade -r - && (ifup lan &)' < ~/Downloads/backup-OpenWrt* &&
  sleep 1 &&
  ssh root@$lladdr ifconfig
  ```
  Now you can access LuCI in browser using stable `fd00::/8` Global address again.
- (optional) If you have no SIM card in local SIM1 reader slot, there is not enough packages installed yet to switch the SIM so you might need to do it manually:

  -
    ```sh
    ssh root@$lladdr << "EOF"
    tail /sys/class/gpio/modem1*/value
    set -x

    echo 0 | tee /sys/class/gpio/modem1power/value
    sleep 1

    echo 0 | tee /sys/class/gpio/modem1*sim*/value

    set +x
    tail /sys/class/gpio/modem1*/value
    set -x

    echo 1 | tee /sys/class/gpio/modem1$(cat /etc/sim)/value
    sleep 1

    echo 1 | tee /sys/class/gpio/modem1power/value

    ifup wan

    set +x
    tail /sys/class/gpio/modem1*/value
    EOF
    ```
  - Alternatively, you might want to set up a temporal managed connection to your existing WiFi hotspot (e.g. on mobile) to download missing packages:  
https://openwrt.org/docs/guide-user/network/wifi/connect_client_wifi#command-line_instructions
    ```sh
    ssh root@$lladdr 'sh && service system reload && (ifup lan &)' < files/etc/uci-defaults/98-client_wifi-dhcp_client &&
    sleep 1 &&
    ssh root@$lladdr ifconfig
    ```
    You can also access the router by hostname now.
  - Alternatively, hybrid WiFi AP+STA mode can be used:  
https://openwrt.org/docs/guide-user/network/wifi/ap_sta
    ```sh
    ssh root@$lladdr 'sh; ifup wan' < files/etc/uci-defaults/99-ap_sta
    ```
- Packages to install:
  ```sh
  ssh root@$lladdr << "EOF"
  grep -q IceG_repo /etc/opkg/customfeeds.conf || echo 'src/gz IceG_repo https://github.com/4IceG/Modem-extras/raw/main/myrepo' >> /etc/opkg/customfeeds.conf
  wget https://github.com/4IceG/Modem-extras/raw/main/myrepo/IceG-repo.pub -O /tmp/IceG-repo.pub
  opkg-key add /tmp/IceG-repo.pub
  wget http://openwrt.132lan.ru/packages/21.02/packages/add.sh -O - | sh

  opkg update

  opkg install \
  kmod-usb-serial-option \
  chat \
  luci-proto-qmi \
  luci-app-atinout \
  luci-app-modeminfo \
  modeminfo-serial-quectel \
  modeminfo-qmi \
  qmi-utils \
  hostapd-utils \
  luci-app-commands \
  bash \
  luci-app-sms-tool-js \
  luci-app-attendedsysupgrade \
  # luci-app-smstools3
  EOF
  ```

### Credits
Many thanks to:
- Kroks for publishing [OpenWrt sources](https://github.com/kroks-free/openwrt) for their routers and being responsive on Helpdesk
- [MrFuture](https://4pda.to/forum/index.php?showuser=9662718) for creating a [topic on 4PDA](https://4pda.to/forum/index.php?showtopic=994528) and publishing SIM switching method via GPIO
- [GlshchnkLx](https://4pda.to/forum/index.php?showuser=7254859) and [linaro](https://4pda.to/forum/index.php?showuser=1857905) for GPIO fix and overall support
- other members of the Kroks topic on 4PDA
