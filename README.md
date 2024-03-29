# Homelab Media Center

Hi! This is my take at implementing a fully automated media center solution based on the *Arr family (https://wiki.servarr.com/). \
The aim of the implementation is to have all the applications running behind Gluetun connected to the Surfshark VPN. \
The whole solution (gluetun + qbittorrent + radarr + sonarr + prowlarr + flaresolverr + bazaar + emby) runs on top of Docker which is installed in a Proxmox LXC container.

## Proxmox Host Configuration

My media center server runs with a WD Red 3Tb hard drive attached as external disk and mounted through */etc/fstab* to the */downloads* mount point:

    <file system>                             <mount point> <type> <options>               <dump> <pass>
    /dev/pve/root                             /             ext4   errors=remount-ro       0      1
    UUID=9FA9-C277                            /boot/efi     vfat   defaults                0      1
    /dev/pve/swap                             none          swap   sw                      0      0
    proc                                      /proc         proc   defaults                0      0
    UUID=7d3736ec-df24-485d-bd82-9f9f79fd7ac0 /downloads           defaults,noatime,nofail 0      0



## Proxmox LXC Configuration

The whole solution runs within a Promox LXC container configured as follows:

    arch: amd64
    cores: 8
    features: keyctl=1,nesting=1
    hostname: Docker
    memory: 8192
    mp0: /downloads,mp=/downloads
    net0: name=eth0,bridge=vmbr0,firewall=1,hwaddr=9A:78:BC:F4:23:2B,ip=dhcp,type=veth
    onboot: 1
    ostype: debian
    rootfs: local-lvm:vm-102-disk-0,size=60G
    startup: order=2
    swap: 2048
    lxc.cgroup2.devices.allow: c 10:200 rwm
    lxc.mount.entry: /dev/net dev/net none bind,create=dir
At the time of writing this article, my Proxmox installation is based on the 8.1.4 release. \
If for whatever reason you get an error on the tun interface, refer to the Proxmox wiki as different Proxmox releases might have different syntax to address the issue: https://pve.proxmox.com/wiki/OpenVPN_in_LXC

## Docker Compose
Below my Docker Compose file used to start up the whole solution:

```yml
    version: "3.8"
    services:
      gluetun:
        image: qmcgaw/gluetun:latest
        cap_add:
          - NET_ADMIN
        environment:
          - VPN_SERVICE_PROVIDER=surfshark
          # OpenVPN Configuration
          # - VPN_TYPE=openvpn
          # - OPENVPN_USER=[username_from_surfshark_portal]
          # - OPENVPN_PASSWORD=[password_from_surfshark_portal]
          # End of OpenVPN Configuration
          # Wireguard Configuration
          - VPN_TYPE=wireguard
          - WIREGUARD_PRIVATE_KEY=[private_key_from_surfshark_portal]
          - WIREGUARD_ADDRESSES=[ip_range_from_surfshark_portal]
          # End of Wireguard Configuration
          # Other VPN parameters
          - SERVER_COUNTRIES=Italy,Germany,Netherlands
        volumes:
          - ./gluetun/config:/config
        ports:
          # qbittorrent ports
          - 8080:8080
          - 6881:6881
          - 6881:6881/udp
          # deluge ports
          # - 8112:8112
          # - 6881:6881
          # - 6881:6881/udp
          # radarr ports
          - 7878:7878
          # sonarr ports
          - 8989:8989
          # prowlarr ports
          - 9696:9696
          # flaresolverr ports
          - 8191:8191
          # bazarr ports
          - 6767:6767
        restart: always

      qbittorrent:
        image: lscr.io/linuxserver/qbittorrent:latest
        container_name: qbittorrent
        network_mode: "service:gluetun"
        environment:
          - PUID=1000
          - PGID=1000
          - TZ=Europe/Rome
          - WEBUI_PORT=8080
        volumes:
          - ./qbittorrent/config:/config
          - /downloads:/downloads
        depends_on:
          gluetun:
            condition: service_healthy

      radarr:
        image: linuxserver/radarr:latest
        container_name: radarr
        network_mode: "service:gluetun"
        environment:
          - PUID=1000
          - PGID=1000
          - TZ=Europe/Rome
        volumes:
          - ./radarr/config:/config
          - /downloads:/downloads
          - /downloads/movies:/movies
        restart: unless-stopped

      sonarr:
        image: linuxserver/sonarr:latest
        container_name: sonarr
        network_mode: "service:gluetun"
        environment:
          - PUID=1000
          - PGID=1000
          - TZ=Europe/Rome
        volumes:
          - ./sonarr/config:/config
          - /downloads:/downloads
          - /downloads/tvseries:/tv
        restart: unless-stopped

      prowlarr:
        image: lscr.io/linuxserver/prowlarr:latest
        container_name: prowlarr
        network_mode: "service:gluetun"
        environment:
          - PUID=1000
          - PGID=1000
          - TZ=Europe/Rome
        volumes:
          - ./prowlarr/config:/config
        restart: unless-stopped

      flaresolverr:
        image: ghcr.io/flaresolverr/flaresolverr:latest
        container_name: flaresolverr
        network_mode: "service:gluetun"
        environment:
          - LOG_LEVEL=${LOG_LEVEL:-info}
          - LOG_HTML=${LOG_HTML:-false}
          - CAPTCHA_SOLVER=${CAPTCHA_SOLVER:-none}
          - TZ=Europe/Rome
        restart: unless-stopped

      bazarr:
        image: lscr.io/linuxserver/bazarr:latest
        container_name: bazarr
        network_mode: "service:gluetun"
        environment:
          - PUID=1000
          - PGID=1000
          - TZ=Europe/Rome
        volumes:
          - ./bazarr/config:/config
          - /downloads/movies:/movies
          - /downloads/tvseries:/tv
        restart: unless-stopped

      emby:
        image: emby/embyserver
        container_name: embyserver
        #runtime: nvidia # Expose NVIDIA GPUs
        network_mode: host # Enable DLNA and Wake-on-Lan
        environment:
          - UID=1000 # The UID to run emby as (default: 2)
          - GID=100 # The GID to run emby as (default 2)
          - GIDLIST=100 # A comma-separated list of additional GIDs to run emby as (default: 2)
        volumes:
          - ./emby/config:/config
          - /downloads/tvseries:/tvseries
          - /downloads/movies:/movies
        ports:
          - 8096:8096 # HTTP port
          - 8920:8920 # HTTPS port
        restart: unless-stopped

    #  deluge:
    #    image: lscr.io/linuxserver/deluge:latest
    #    container_name: deluge
    #    network_mode: "service:gluetun"
    #    environment:
    #      - PUID=1000
    #      - PGID=1000
    #      - TZ=Europe/Rome
    #      - DELUGE_LOGLEVEL=error #optional
    #    volumes:
    #      - ./deluge/config:/config
    #      - /downloads:/downloads
    #    ports:
    #      - 8112:8112
    #      - 6881:6881
    #      - 6881:6881/udp
    #      - 58846:58846 #optional
    #    depends_on:
    #      gluetun:
    #        condition: service_healthy

    #  plex:
    #    image: lscr.io/linuxserver/plex:latest
    #    container_name: plex
    #    network_mode: host
    #    environment:
    #      - PUID=1000
    #      - PGID=1000
    #      - VERSION=docker
    #      - PLEX_CLAIM=claim-[redacted]
    #    volumes:
    #      - ./plex/config:/config
    #      - /downloads:/data/media
    #    restart: unless-stopped

    #  overseerr:
    #    image: lscr.io/linuxserver/overseerr:latest
    #    container_name: overseerr
    #    environment:
    #      - PUID=1000
    #      - PGID=1000
    #      - TZ=Europe/Rome
    #    volumes:
    #      - ./overseerr/config:/config
    #    ports:
    #      - 5055:5055
    #    restart: unless-stopped
```

The Gluetun section provides the syntax to configure Surfshark using either Wireguard or OpenVPN. For more information you can check the official Gluetun documentation at: https://github.com/qdm12/gluetun-wiki/blob/main/setup/providers/surfshark.md \
I initially based the implementation on Wireguard, but, for some still not clear reasons, qbittorrent tends to hang all the ongoing connections and it is not able to recover the downloads when that happens unless the whole stack gets restarted.  It seems other people are experiencing the same, and the workaround to that is to switch over to the OpenVPN protocol. \
I am currently using Emby as media handler and qbittorrent as Torrent client, but I left Plex, Overseerr and Deluge configurations in there for your own convenience. \
