# Docs-Keepalived-Haproxy

## Mô Hình
- 2 Server HAproxy:
  + HAproxyMaster: 172.32.0.11 (Master)
  + HAproxyBackup: 172.32.0.12 (Backup)
  * VIP: 172.32.0.10
- LoadBalance 3 server backend (ex: port 8080)
  + 172.32.0.20:8080
  + 172.32.0.21:8080
  + 172.32.0.22:8080

## Cài đặt và cấu hình : (Centos 7)

Bước 1: `Trên 2 server HAproxy` Chỉnh sửa file `/etc/sysctl.conf`:

> `sudo vim /etc/sysctl.conf`

  - Thêm hoặc chỉnh sửa dòng sau:

  `net.ipv4.ip_nonlocal_bind=1`

Bước 2: `Trên 2 server HAproxy` Cài Keepalived, HAproxy:

> `sudo yum install  -y vim elpel-release`

> `sudo yum install -y keepalived haproxy`

Bước 3: `Trên server HAproxyMaster` Config keepalived.

> `sudo mv /etc/keepalived/keepalived.conf /etc/keepalivedkeepalived.conf.bak`

> `sudo vim /etc/keepalived/keepalived.conf`

```conf
global_defs {
    router_id keepalived1 #khai báo route_id của keepalived
}
vrrp_script chk_haproxy {
    script "pkill -0 haproxy"
    interval 2
    weight 2
}
vrrp_instance VI_1 {
    virtual_router_id 51
    advert_int 1
    priority 100
    state MASTER
    interface ens33 ## Lưu Ý: edit đúng interface
    virtual_ipaddress {
      172.32.0.10 dev ens33 ## Khai VIP và interface tương ứng
    }
    authentication {
      auth_type PASS
      auth_pass 123456 #Password này phải khai báo giống nhau giữa các server keepalived
    }
  track_script {
    chk_haproxy
  }
}
```

> `sudo systemctl start keepalived`

> `sudo systemctl enable keepalived`

Bước 4: `Trên server HAproxyBackup` Config keepalived.

> `sudo mv /etc/keepalived/keepalived.conf /etc/keepalivedkeepalived.conf.bak`

> `sudo vim /etc/keepalived/keepalived.conf`

```conf
global_defs {
    router_id keepalived2 #khai báo route_id của keepalived
}
vrrp_script chk_haproxy {
    script "pkill -0 haproxy"
    interval 2
    weight 2
}
vrrp_instance VI_1 {
    virtual_router_id 51
    advert_int 1
    priority 99
    state BACKUP
    interface ens33 ## Lưu Ý: edit đúng interface
    virtual_ipaddress {
      172.32.0.10 dev ens33 ## Khai VIP và interface tương ứng
    }
    authentication {
      auth_type PASS
      auth_pass 123456 #Password này phải khai báo giống nhau giữa các server keepalived
    }
  track_script {
    chk_haproxy
  }
}
```

> `sudo systemctl start keepalived`

> `sudo systemctl enable keepalived`

Bước 5: `Trên 2 server HAproxy ` Config haproxy:

> `sudo mv /etc/haproxy/haproxy.cfg /etc/haproxy/haproxy.cfg.bak`

> `sudo vim /etc/haproxy/haproxy.cfg`

```conf
global
        daemon
        maxconn 256

    defaults
        mode http
        timeout connect 5000ms
        timeout client 50000ms
        timeout server 50000ms

    frontend http-in
        bind *:80
        default_backend app
    backend static
        balance roundrobin
        server static 172.32.0.10:80
    backend app
        balance roundrobin
        server app1 172.32.0.20:8080 check
        server app2 172.32.0.21:8080 check
        server app3 172.32.0.22:8080 check
```

> `sudo systemctl start keepalived`

> `sudo systemctl enable keepalived`
