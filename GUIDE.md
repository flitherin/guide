Я прочитал задание. Ниже полное прохождение по порядку. Подставляйте свои значения: **NN** (например `03`), **X** (номер без нуля, например `3`) и **PUB** (белый IP сервера).

Двух готовых ответов в задании нет: код приложения я не видел (репозиторий открыть не удалось), а в карточке стенда есть неточности. Они отмечены ниже, их надо проверить по месту.

## 0. Подготовка (10 минут)

```bash
ssh redos@PUB
sudo -i
dnf update -y   # по желанию
setenforce 0
sed -i 's/^SELINUX=.*/SELINUX=permissive/' /etc/selinux/config
systemctl enable --now firewalld
```

SELinux проще перевести в permissive, чтобы не тратить время на отладку. Порядок работ такой: сначала сеть и туннель, потом остальное.

## 1. Адрес 172.16.X.10 (dummy-интерфейс)

```bash
nmcli con add type dummy ifname dummy0 con-name dummy0 ip4 172.16.X.10/32
nmcli con up dummy0
ip a show dummy0
```

NetworkManager сохраняет интерфейс после перезагрузки. Если NM не используется, делайте `ip link add` через systemd-networkd или unit-файл.

## 2. Модуль 2: IPsec (strongSwan, swanctl)

```bash
dnf install -y strongswan
systemctl list-unit-files | grep -i swan
```

Нужен юнит с charon-systemd. Обычно это `strongswan.service` или `strongswan-swanctl.service`. Если стоит только старый `strongswan-starter`, смотрите `dnf search strongswan`.

Конфиг `/etc/strongswan/swanctl/swanctl.conf` (на некоторых сборках путь `/etc/swanctl/swanctl.conf`):

```
connections {
  office {
    version = 2
    local_addrs  = %any
    remote_addrs = vpn.olimp.gogle.com
    proposals = aes256-sha256-modp2048
    dpd_delay = 30s
    keyingtries = 0
    local  { auth = psk
             id = standNN.olimp.gogle.com }
    remote { auth = psk
             id = vpn.olimp.gogle.com }
    children {
      office {
        local_ts  = 172.16.X.10/32
        remote_ts = 172.16.X.1/32
        esp_proposals = aes256gcm16-modp2048
        mode = tunnel
        start_action = start
        dpd_action = restart
        close_action = start
      }
    }
  }
}
secrets {
  ike-office {
    id-1 = standNN.olimp.gogle.com
    secret = "PSK_ИЗ_КАРТОЧКИ"
  }
}
```

```bash
firewall-cmd --permanent --add-service=ipsec   # 500/4500 + esp
firewall-cmd --reload
systemctl enable --now strongswan   # или strongswan-swanctl
swanctl --load-all
swanctl --list-sas
ping -I 172.16.X.10 172.16.X.1
```

Если IKE_SA есть, а ESP нет, проверьте `esp_proposals` и PFS (`modp2048` в ESP — это и есть PFS). Если не резолвится `vpn.olimp.gogle.com`, проверьте `/etc/resolv.conf`.

## 3. Модуль 1: DNS (BIND)

```bash
dnf install -y bind bind-utils
```

В `/etc/named.conf` измените или добавьте:

```
options {
  listen-on port 53 { any; };
  listen-on-v6 { none; };
  allow-query { any; };
  recursion no;
  directory "/var/named";
};
zone "standNN.olimp.gogle.com" IN {
  type master;
  file "standNN.olimp.gogle.com.zone";
};
```

Файл `/var/named/standNN.olimp.gogle.com.zone`:

```
$TTL 300
@   IN SOA ns.standNN.olimp.gogle.com. admin.standNN.olimp.gogle.com. (
        2026111101 3600 600 604800 300 )
@   IN NS  ns.standNN.olimp.gogle.com.
@   IN A   PUB
ns  IN A   PUB
app IN A   PUB
site IN A  PUB
www IN A   172.16.X.10
```

```bash
chown root:named /var/named/standNN*.zone
named-checkconf && named-checkzone standNN.olimp.gogle.com /var/named/standNN.olimp.gogle.com.zone
firewall-cmd --permanent --add-service=dns && firewall-cmd --reload
systemctl enable --now named
dig @PUB SOA standNN.olimp.gogle.com
dig @8.8.8.8 NS standNN.olimp.gogle.com
dig @1.1.1.1 SOA standNN.olimp.gogle.com
```

При правках зоны увеличивайте serial.

## 4. Docker

```bash
dnf search docker | head
dnf install -y docker-ce   # или docker / moby-engine, смотря что в репозитории RedOS
systemctl enable --now docker
docker network create olimp
```

Если Docker Hub не отвечает, настройте зеркало в `/etc/docker/daemon.json` (`registry-mirrors`).

**Важно:** порты контейнеров никогда не публикуйте на 0.0.0.0. Docker обходит firewalld, и порт окажется открытым в интернет.

### MySQL (модуль 3)

```bash
mkdir -p /srv/mysql
docker run -d --name mysql --restart always --network olimp \
  -v /srv/mysql:/var/lib/mysql \
  -v /opt/olimp/init.sql:/docker-entrypoint-initdb.d/init.sql:ro \
  -e MYSQL_ROOT_PASSWORD=RootPass123 \
  -e MYSQL_DATABASE=techstart \
  -e MYSQL_USER=techstart \
  -e MYSQL_PASSWORD=TechPass123 \
  mysql:8
docker logs -f mysql   # дождитесь "ready for connections"
docker exec mysql mysql -utechstart -pTechPass123 techstart -e 'show tables;'
```

Порт не публикуем (3.3 выполнится сам). `init.sql` выполняется только при **пустом** `/srv/mysql`. Если нужно перезалить, сделайте `docker rm -f mysql; rm -rf /srv/mysql/*` и запустите заново.

### App

```bash
cd /opt/olimp/app   # или git clone https://github.com/VadimGarifullin/olimp-app
docker build -t olimp-app .
docker run -d --name app --restart always --network olimp \
  -p 127.0.0.1:8000:8000 \
  -e STAND=standNN -e DB_HOST=mysql -e DB_PORT=3306 \
  -e DB_NAME=techstart -e DB_USER=techstart -e DB_PASSWORD=TechPass123 \
  olimp-app
curl -s localhost:8000/health
```

Нужно получить `"db":"ok"` и число сотрудников. Если приложение стартовало раньше БД, перезапустите его командой `docker restart app`.

### cAdvisor (модуль 5.2)

```bash
docker run -d --name cadvisor --restart always --privileged \
  -p 172.16.X.10:8080:8080 \
  -v /:/rootfs:ro -v /var/run:/var/run:ro -v /sys:/sys:ro \
  -v /var/lib/docker/:/var/lib/docker:ro -v /dev/disk/:/dev/disk:ro \
  --device /dev/kmsg \
  ghcr.io/google/cadvisor:latest
```

Если образ не скачивается, возьмите `gcr.io/cadvisor/cadvisor` или его зеркало. Привязка к 172.16.X.10 закрывает 8080 от интернета, так что отдельное правило не нужно. Проверка: `curl 172.16.X.10:8080/metrics | grep 'name="mysql"'`.

Чтобы после перезагрузки контейнер не стартовал раньше интерфейса, добавьте drop-in: `systemctl edit docker`, в нём `[Unit]`, `After=network-online.target NetworkManager-wait-online.service`.

## 5. Node exporter (модуль 5.1)

```bash
dnf install -y golang-github-prometheus-node-exporter || {
  # запасной вариант: бинарник с github.com
  cd /tmp
  curl -LO https://github.com/prometheus/node_exporter/releases/download/v1.8.2/node_exporter-1.8.2.linux-amd64.tar.gz
  tar xf node_exporter-*.tar.gz
  cp node_exporter-*/node_exporter /usr/local/bin/
  useradd -r -s /sbin/nologin node_exporter
  cat >/etc/systemd/system/node_exporter.service <<'EOF'
[Unit]
Description=Node Exporter
After=network-online.target
[Service]
User=node_exporter
ExecStart=/usr/local/bin/node_exporter --web.listen-address=:9100
Restart=always
[Install]
WantedBy=multi-user.target
EOF
  systemctl daemon-reload
}
systemctl enable --now node_exporter   # имя юнита может быть node-exporter
```

Он слушает все адреса, как и требуется. Закрывать будем файрволом (раздел 8).

## 6. Модуль 4: nginx и HTTPS

```bash
dnf install -y nginx certbot
mkdir -p /var/www/letsencrypt
```

### Шаг 1. Временный конфиг только для HTTP

`/etc/nginx/conf.d/olimp.conf`:

```
server {
  listen 80 default_server;
  server_name standNN.olimp.gogle.com;
  location /.well-known/acme-challenge/ { root /var/www/letsencrypt; }
  location / { return 301 https://$host$request_uri; }
}
```

```bash
firewall-cmd --permanent --add-service={http,https} && firewall-cmd --reload
nginx -t && systemctl enable --now nginx
```

### Шаг 2. Сначала staging, потом боевой сертификат

```bash
certbot certonly --webroot -w /var/www/letsencrypt \
  -d standNN.olimp.gogle.com --staging -m you@mail.ru --agree-tos -n
# если всё ок, боевой выпуск:
certbot certonly --webroot -w /var/www/letsencrypt \
  -d standNN.olimp.gogle.com --force-renewal -m you@mail.ru --agree-tos -n
```

Сертификат нужен только для apex-имени, для site и www он не нужен.

### Шаг 3. Полный конфиг

```
server {
  listen 80 default_server;
  server_name standNN.olimp.gogle.com;
  location /.well-known/acme-challenge/ { root /var/www/letsencrypt; }
  location / { return 301 https://$host$request_uri; }
}

server {
  listen 443 ssl;
  server_name standNN.olimp.gogle.com;
  ssl_certificate     /etc/letsencrypt/live/standNN.olimp.gogle.com/fullchain.pem;
  ssl_certificate_key /etc/letsencrypt/live/standNN.olimp.gogle.com/privkey.pem;
  location ^~ /internal { return 404; }
  location / {
    proxy_pass http://127.0.0.1:8000;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
  }
}

server {
  listen 80;
  server_name site.standNN.olimp.gogle.com;
  location ^~ /internal { return 404; }
  location / {
    proxy_pass http://127.0.0.1:8000;
    proxy_set_header Host $host;
  }
}

server {
  listen 172.16.X.10:80;
  server_name www.standNN.olimp.gogle.com;
  location = / { proxy_pass http://127.0.0.1:8000/internal; }
  location /    { proxy_pass http://127.0.0.1:8000; }
}
```

Что здесь важно:
- `www` слушает только 172.16.X.10, поэтому по белому IP он не отвечает (4.5).
- `default_server` на 80 порту отвечает на запросы по голому IP и не отдаёт `/internal`.
- Если у `/internal` есть ссылки на статику и она не грузится, проверьте это в браузере через туннель или `curl`.

Автообновление и перезагрузка nginx:

```bash
cat >/etc/letsencrypt/renewal-hooks/deploy/nginx.sh <<'EOF'
#!/bin/sh
systemctl reload nginx
EOF
chmod +x /etc/letsencrypt/renewal-hooks/deploy/nginx.sh
systemctl enable --now certbot-renew.timer   # если нет, добавьте cron
```

Чтобы nginx не падал при загрузке, пока нет 172.16.X.10, добавьте drop-in `After=network-online.target NetworkManager-wait-online.service`. Либо включите `net.ipv4.ip_nonlocal_bind=1` в `/etc/sysctl.d/`.

Проверка:

```bash
curl -I http://standNN.olimp.gogle.com            # 301
curl -s https://standNN.olimp.gogle.com/health
curl -I http://site.standNN.olimp.gogle.com       # 200, без редиректа
curl -I https://standNN.olimp.gogle.com/internal  # 404
curl --resolve www.standNN.olimp.gogle.com:80:PUB http://www.standNN.olimp.gogle.com   # должно НЕ работать
```

## 7. Модуль 6: NFS

```bash
dnf install -y nfs-utils
groupadd -g 2000 office
useradd -u 2000 -g 2000 -M -s /sbin/nologin office
mkdir -p /srv/share/reports
echo standNN > /srv/share/stand.txt
chown root:root /srv/share /srv/share/stand.txt
chmod 755 /srv/share; chmod 644 /srv/share/stand.txt
chown 2000:2000 /srv/share/reports; chmod 775 /srv/share/reports
```

`/etc/exports`:

```
/srv/share 172.16.X.249(rw,sync,all_squash,anonuid=2000,anongid=2000,no_subtree_check)
```

`/etc/nfs.conf`, в секции `[nfsd]`:

```
vers2=n
vers3=n
vers4=y
udp=n
```

```bash
systemctl enable --now nfs-server
exportfs -ra && exportfs -v
firewall-cmd --permanent --add-rich-rule='rule family=ipv4 source address=172.16.X.249 port port=2049 protocol=tcp accept'
firewall-cmd --reload
```

`all_squash` с anonuid 2000 даёт владельца UID 2000 для всех созданных файлов. В корень офис писать не может, он принадлежит root, поэтому `stand.txt` не изменить и не удалить.

**Неточности в задании по NFS.** Путь указан как `172.16.100.1NN:/srv/share`, а офис как `172.16.100.249`, а не `172.16.X.249`. Скорее всего это опечатки, адрес сервера должен быть 172.16.X.10. Кроме того, селектор туннеля пропускает только 172.16.X.1/32. Источник у NFS-трафика офиса может оказаться другим, вероятно .1 из-за NAT на стороне офиса. Поэтому сначала откройте доступ и для 172.16.X.1, посмотрите `ss -tn | grep 2049`, и только потом сузьте экспорт и правило файрвола до реального адреса.

## 8. Файрвол и проверки «недоступно из интернета»

```bash
# node_exporter только из туннеля (источник офиса — уточните: .1 или .249)
firewall-cmd --permanent --add-rich-rule='rule family=ipv4 source address=172.16.X.1 port port=9100 protocol=tcp accept'
firewall-cmd --permanent --add-rich-rule='rule family=ipv4 source address=172.16.X.249 port port=9100 protocol=tcp accept'
firewall-cmd --reload
firewall-cmd --list-all
```

Отдельно убедитесь, что в зоне нет лишних сервисов (`mysql`, `nfs`, `cockpit` и т.д.) и что ssh остаётся открытым.

## 9. Автозапуск и проверка перезагрузки (обязательно)

```bash
systemctl enable named strongswan nginx docker node_exporter nfs-server firewalld certbot-renew.timer
reboot
```

После перезагрузки проверьте по порядку:
1. `swanctl --list-sas` и `ip a show dummy0`.
2. `docker ps` (должны быть `mysql`, `app`, `cadvisor`).
3. `curl https://standNN.olimp.gogle.com/health`.
4. `dig @8.8.8.8 SOA standNN.olimp.gogle.com`.
5. `systemctl status nginx named node_exporter nfs-server`.
6. Снаружи (с другой машины): `nmap -p 3306,9100,8080,2049 PUB` должен показать закрытые порты.

Итоговый чек-лист по баллам:

| Модуль | Что должно быть |
|---|---|
| 1 | SOA/NS через 8.8.8.8, apex и site → PUB, www → 172.16.X.10 |
| 2 | Активные IKE_SA и CHILD_SA, ping 172.16.X.10 из офиса |
| 3 | Контейнеры `mysql` и `app`, `/health` с `db: ok`, 3306 закрыт |
| 4 | HTTPS с LE на apex, 301 с HTTP, site по HTTP, www только на 172.16.X.10, `/internal` закрыт публично |
| 5 | :9100 и 172.16.X.10:8080, снаружи закрыты |
| 6 | NFSv4 `/srv/share`, запись в reports от UID 2000, `stand.txt` неизменяем |

Если хотите, могу оформить это в виде файла (docx или markdown) либо расписать любой модуль ещё подробнее, например с отладкой типичных ошибок.
