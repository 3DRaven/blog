---
layout: post
title: "Личное облако на Proxmox: сеть"
date: 2025-11-03 19:25:22 +0000
tags: ["Proxmox", "opensense", "xray", "3x-ui", "vps", "домашний сервер"]
description: "Как‑то я писал статью «Личное облако на Proxmox» где описывал как установить Proxmox разными способами и в целом, что это такое, обещал, что когда то, под настроение опишу продолжение про сетевую часть домашнего облака с сервисами, доступными из интернет на домашнем сервере, вот, пожалуйста Прошлая статья была посвящена базовой настройке. Что сейчас я имею в итоге, немного изменив то, что делал я получил:"
---

Как-то я писал статью "Личное облако на Proxmox" где описывал как установить Proxmox разными способами и в целом, что это такое, обещал, что когда то, под настроение опишу продолжение про сетевую часть домашнего облака с сервисами, доступными из интернет на домашнем сервере, вот, пожалуйста.

Прошлая [статья]({{ site.baseurl }}{% post_url 2024-03-02-proxmox-personal-cloud %}) была посвящена базовой настройке. Что сейчас я имею в итоге:

1.  Домашний сервер Asus NUC, на нем установлен Proxmox, правда я не стал ставить на десктопный Debian, так как передумал делать ему красивую панельку с экранчиком, а значит графический интерфейс ему не нужен и можно просто поставить с iso образа, который предоставляют авторы Proxmox.

2.  Роутер к которому подключен сервер и домашняя техника, например мой ноутбук. На роутере OpenWRT. Интернет обычный домашний, без статического IP адреса.

3.  VPS в интернете за три копейки с ubuntu 25.04 server. Проверьте что бы у VPS IP адрес в базах спамеров и прочих блок листах не был, а то сайты вас пускать не будут :) Проверить можно [тут](https://mxtoolbox.com/SuperTool.aspx), там есть \`blacklist check\`.

Задача:

1.  Установить домашний сервис в изолированном окружении.

2.  Предоставить доступ к домашним сервисам из интернета.

3.  Сохранить доступ к админкам из домашней сети.

Начнем. Заранее предупрежу, что вероятно некоторые настройки, описанные в статье так как я их помню, но должны быть сделаны раньше чем в статье, уж как помню, прочитайте статью целиком прежде чем начать делать.

Обратите внимание, что так как нейронки сейчас отлично могут советовать настройки, если что не ясно, спросите их, просто скопипастив кусок статьи. В случае возникновения проблем, проще всего диагностировать их так, заходите в терминал всех сервисов и серверов, по ssh или в proxmox console и на каждом интерфейсе каждого сервиса запускаете tcpdump -i \${название интерфейсов сетевых}, после чего делаете коннект к VPS или ping и смотрите через какие интерфейсы и что пролетело. Можно отфильтровать лишнее например ssh коннект свой из ловли, что бы контекст нейронки не переполнить, они в целом хорошо понимают до 3к строк текста всего, обычно, по моему опыту. Причем даже если прислать больше они нередко отрезают первые 5к строк кодом на питоне самодельным, так что делите на части. Потом наловленное скармливаете нейронке и просите проанализировать. Она быстро вам опишет где цепочка передачи данных оборвалась.

При этом, нейронку можно попросить написать скрипты, которые скинут в файл настройки фаерволов, таблицы маршрутизации и интерфейсы сетевые, что бы ей можно было промпты потом из этого делать быстро, файл итоговый приложил и все. Я такой скрипт на своему ноутбуке запускаю, он подключается ко всем серверам, что описаны ниже и всю нужную информацию собирает в файл, это экономит дни диагностики проблем.

Первое, что нужно сделать это настроить OpenWRT так что бы br-lan, интерфейс, который там создается сам, обслуживал две VLAN. В одной будет домашняя техника, во второй будет Proxmox. Это нужно что бы изолировать и обезопасить домашнюю технику. Заходим в LuCI в настройки Network-\>Interfaces-\>Devices-\>br-lan-\>Configure-\>Bridge VLAN filtering. Там создаем VLAN 100 и VLAN 200, разрешаем VLAN как таковые. В VLAN 200 будет Proxmox, сервер кабелем подключен к какому то LAN порту, вот его туда и добавить как Untagged. Untagged значит, что на стороне самого Proxmox никаких VLAN настроек мы делать не будем, он не будет ставить tags. Остальные порты добавляем в VLAN 100 так же Untagged. Этим мы разделим домашнюю сеть на две подсети, в которых будут отдельно обитать наши сервисы и домашняя техника. Так же нужно добавить пару правил firewall. Я не буду расписывать кнопки это долго, приведу конфиги, которые получатся в итоге и нейронка вам расскажет, что жать.

Это конфиг сети:

Скрытый текст

    root@OpenWrt:~# cat /etc/config/network

    config interface 'loopback'
        option device 'lo'
        option proto 'static'
        option ipaddr '127.0.0.1'
        option netmask '255.0.0.0'

    config globals 'globals'
        option ula_prefix 'fd61:eb10:9fdd::/48'
        option packet_steering '1'

    config device
        option name 'br-lan'
        option type 'bridge'
        list ports 'lan1'
        list ports 'lan2'
        list ports 'lan3'
        list ports 'lan4'
        option ipv6 '0'

    config interface 'lan'
        option device 'br-lan.100'
        option proto 'static'
        option ipaddr '192.168.1.1'
        option netmask '255.255.255.0'
        option ip6assign '60'
        option ipv6 'off'

    config interface 'wan'
        option device 'wan'
        option proto 'dhcp'
        option peerdns '0'
        list dns '9.9.9.9'
        list dns '149.112.112.112'
        option ipv6 'off'

    config device
        option name 'phy1-ap0'
        option ipv6 '0'

    config device
        option name 'phy0-ap0'
        option ipv6 '0'

    config device
        option name 'wan'
        option ipv6 '0'

    config device
        option name 'miireg'
        option ipv6 '0'

    config device
        option name 'lan4'
        option ipv6 '0'

    config device
        option name 'lan3'
        option ipv6 '0'

    config device
        option name 'lan2'
        option ipv6 '0'

    config device
        option name 'lan1'
        option ipv6 '0'

    config device
        option name 'br-wan'
        option type 'bridge'
        list ports 'wan'
        option mtu '1500'
        option ipv6 '0'

    config bridge-vlan
        option device 'br-lan'
        option vlan '100'
        list ports 'lan1'
        list ports 'lan2'
        list ports 'lan3'

    config bridge-vlan
        option device 'br-lan'
        option vlan '200'
        list ports 'lan4'

    config interface 'proxmox'
        option proto 'static'
        option device 'br-lan.200'
        option ipaddr '192.168.2.1'
        option netmask '255.255.255.0'

Это конфиг firewall:

Скрытый текст

    root@OpenWrt:~# cat /etc/config/firewall

    config defaults
        option input 'DROP'
        option output 'DROP'
        option forward 'REJECT'
        option synflood_protect '1'
        option drop_invalid '1'

    config zone
        option name 'lan'
        option input 'ACCEPT'
        option output 'ACCEPT'
        option forward 'ACCEPT'
        list network 'lan'

    config zone
        option name 'wan'
        option input 'DROP'
        option output 'ACCEPT'
        option forward 'DROP'
        option mtu_fix '1'
        option masq '1'
        list network 'wan'

    config forwarding
        option src 'lan'
        option dest 'wan'

    config zone
        option name 'proxmox'
        option input 'REJECT'
        option output 'ACCEPT'
        option forward 'REJECT'
        list network 'proxmox'

    config forwarding
        option src 'lan'
        option dest 'proxmox'

    config rule
        option name 'Allow LAN to Proxmox Management'
        option src 'lan'
        option dest 'proxmox'
        option dest_ip '192.168.2.2'
        option dest_port '8006'
        option proto 'tcp'
        option target 'ACCEPT'

    config rule
        option name 'Allow LAN to Proxmox SSH'
        option src 'lan'
        option dest 'proxmox'
        option dest_ip '192.168.2.2'
        option dest_port '22'
        option proto 'tcp'
        option target 'ACCEPT'

    config rule
        option name 'Allow LAN to OPNsense Web'
        option src 'lan'
        option dest 'proxmox'
        option dest_ip '192.168.2.3'
        option dest_port '443'
        option proto 'tcp'
        option target 'ACCEPT'

    config rule
        option name 'Allow LAN to OPNsense HTTP'
        option src 'lan'
        option dest 'proxmox'
        option dest_ip '192.168.2.3'
        option dest_port '80'
        option proto 'tcp'
        option target 'ACCEPT'

    config rule
        option src 'lan'
        option dest 'proxmox'
        option name 'Allow LAN to Proxmox icmp'
        list proto 'icmp'
        option target 'ACCEPT'

    config forwarding
        option src 'proxmox'
        option dest 'wan'

    config rule
        option src 'proxmox'
        option name 'Allow-ICMP-from-Proxmox'
        list proto 'icmp'
        option target 'ACCEPT'

    config rule
        option src 'proxmox'
        option target 'ACCEPT'
        option name 'Proxmox DNS to OpenWRT'
        option dest_port '53'
        list proto 'tcp'
        list proto 'udp'

    config rule
        option src 'proxmox'
        option dest 'lan'
        option name 'Block proxmox to LAN'
        list proto 'all'
        option target 'REJECT'

Тут нужно отметить, что Proxmox сервер у меня настроен с статическим адресом, выдавать серверу адрес по DHCP кажется излишним. Некоторые правила, не относящиеся к задаче я убрал.

По умолчанию в OpenWRT используется для локальнных адресов зона lan, я заменил ее на более правильную home.arpa, а так же добавил DNS записи для нужных мне локальных доменов. Это делается в настройках DNS, если нужно разберетесь, это просто удобство. Так что теперь я могу писать что то вроде proxmox.home.arpa в браузере и открывать админку. Но тут важно то, что Caddy понимает, что home.arpa это локальный адрес, а вот lan она не понимает и потому L4 Proxy, который нам нужен будет, с lan не будет работать.

Далее нужно создать в Proxmox сетевой мост, который будет изолирован от всего. Это позволит обеспечить полную изоляцию наших сервисов, которые мы выставим в интернет, не только внутри VLAN OpenWRT, но и внутри Proxmox хоста. Вот так будет в итоге выглядеть сеть в Proxmox, это все делается с UI, но итог проще показать в виде конфига:

Скрытый текст

    root@pve:~# cat /etc/network/interfaces
    auto lo
    iface lo inet loopback

    iface enp114s0 inet manual

    iface enx000ec65482c4 inet manual

    iface wlo1 inet manual

    auto vmbr0
    iface vmbr0 inet static
        address 192.168.2.2/24
        gateway 192.168.2.1
        bridge-ports enx000ec65482c4
        bridge-stp off
        bridge-fd 0
        nameservers 192.168.2.1

    auto services
    iface services inet manual
        bridge-ports none
        bridge-stp off
        bridge-fd 0
    #Internal services network

В прошлой статье я уже описывал как делать в Proxmox виртуалки, так что сейчас не стану делать это подбробно. Следующим нужно сделать виртуальную машину и установить туда [OPNSense](https://opnsense.org/) он, по моему скромному опыту, намного удобнее и стабильнее чем pf-sense, который я собирался использовать в прошлой статье, главное UI не глючный и все очень ясно описано. Это универсальный комбайн для firewall и, что нам понадобится, он имеет community plugin Caddy, это реверси прокси. Вот такая конфигурация VM где установлен OPNSense:

Скрытый текст

    root@pve:~# qm config 100
    boot: order=scsi0
    cores: 4
    cpu: x86-64-v2-AES
    efidisk0: local-btrfs:100/vm-100-disk-0.raw,efitype=4m,pre-enrolled-keys=1,size=528K
    machine: q35
    memory: 8192
    meta: creation-qemu=10.0.2,ctime=1761775163
    name: opensense
    net0: virtio=MAC,bridge=vmbr0,firewall=1,queues=4
    net1: virtio=MAC,bridge=services,firewall=1,queues=4
    numa: 0
    onboot: 1
    ostype: l26
    parent: P4
    scsi0: local-btrfs:100/vm-100-disk-1.raw,discard=on,iothread=1,size=64G
    scsihw: virtio-scsi-single
    smbios1: uuid=f0313399-c474-416a-9134-8b64b1585a5b
    sockets: 1
    vmgenid: edf40b71-8f32-46de-b31b-831768f16de4

В процессе установки OPNSense нужно в качестве WAN выбрать vmbr0 от Proxmox, а в качестве LAN выбрать мост services, изолирующий наши внутренние сервисы. В консоли OPNSense они будут называться vtnet0 vtnet1, соответственно. Для vtnet0 нужно задать статический адрес так же как для Proxmox, так как такому важному участнику сети выдавать адрес по DHCP кажется не разумным. В целом для VLAN 200 я DHCP отключил на OpenWRT, потому, что в процессе жизни этой инфраструктуры в VLAN 200 больше не появится никого нового.

Настройки firewall, приведенные выше, разрешают доступ из VLAN 100 то есть с домашнего ноутбука к VLAN 200, но не наоборот. Так что после того как VM OPNSense будет запущена можно будет с ноутбука зайти на адрес, заданный для OPNSense, в моем случае 192.168.2.3 и увидеть его админку. Напомню, что я настроил DNS имена в OpenWRT, так что я могу в браузере зайти на opnsense.home.arpa и попасть в админку. Вот такой у OPNSense итоговый конфиг сети на этом этапе настройки:

Скрытый текст

    root@OPNsense:~ # ifconfig
    vtnet0: flags=1008843<UP,BROADCAST,RUNNING,SIMPLEX,MULTICAST,LOWER_UP> metric 0 mtu 1500
        description: WAN (wan)
        options=80028<VLAN_MTU,JUMBO_MTU,LINKSTATE>
        ether bc:24:11:99:e5:59
        inet 192.168.2.3 netmask 0xffffff00 broadcast 192.168.2.255
        media: Ethernet autoselect (10Gbase-T <full-duplex>)
        status: active
        nd6 options=29<PERFORMNUD,IFDISABLED,AUTO_LINKLOCAL>
    vtnet1: flags=1008843<UP,BROADCAST,RUNNING,SIMPLEX,MULTICAST,LOWER_UP> metric 0 mtu 1500
        description: LAN (lan)
        options=80028<VLAN_MTU,JUMBO_MTU,LINKSTATE>
        ether bc:24:11:44:2a:46
        inet 10.0.0.1 netmask 0xffffff00 broadcast 10.0.0.255
        media: Ethernet autoselect (10Gbase-T <full-duplex>)
        status: active
        nd6 options=29<PERFORMNUD,IFDISABLED,AUTO_LINKLOCAL>

Когда это сделано мы получаем ситуацию когда OPNSense является шлюзом для доступа во внешний мир для всех изолированных сервисов за мостом services. Так что весь их трафик проходит через него. Так как у нас есть изоляция OpenWRT firewall+VLAN, изоляция services внутри моста и OPNSense в качестве firewall, я отключил Proxmox firewall, слишком много было бы изоляций, да и к тому же Proxmox firewall постоянно ломался.

Следующим шагом нужно установить первый сервис, в качестве первого сервиса я выбрал xray. Для того что бы он работал нужно не так и много, есть отличная серия статей на эту тему [статья1](https://habr.com/ru/articles/839656/), [статья2](https://habr.com/ru/articles/770400/), [статья3](https://habr.com/ru/articles/731608/), [статья4](https://habr.com/ru/articles/735536/), [статья5](https://habr.com/ru/articles/728696/). Так же я нашел хорошее описание [тут](https://hostkey.com/documentation/marketplace/security/3x_ui/) и [тут](https://github.com/MHSanaei/3x-ui/wiki/Installation#install-in-one-line-recommended). Итогом будет установка LXC контейнера с ubuntu, вот конфиг LXC контейнера:

Скрытый текст

    root@pve:~# pct config 102
    arch: amd64
    cores: 2
    features: nesting=1
    hostname: xray
    memory: 2048
    nameserver: 10.0.0.1
    net0: name=eth0,bridge=services,firewall=1,hwaddr=BC:24:11:9A:65:73,ip=dhcp,type=veth
    ostype: ubuntu
    parent: VPNWorks
    rootfs: local-btrfs:102/vm-102-disk-0.raw,size=8G
    swap: 0
    unprivileged: 1

В LXC контейнере можно запускать вложенные docker контейнеры, так что внтутри я запустил xray в docker compose, так просто удобнее ничего не нужно настраивать, с таким docker-compose.yml:

Скрытый текст

    root@xray:~# cat  /home/i3draven/panel/docker-compose.yml 
    services:
      3xui:
        image: ghcr.io/mhsanaei/3x-ui:latest
        container_name: 3xui_app
        hostname: my.host.name # если надо
        volumes:
          - $PWD/db/:/etc/x-ui/
          - $PWD/cert/:/root/cert/
        environment:
          XRAY_VMESS_AEAD_FORCED: "false"
          XUI_ENABLE_FAIL2BAN: "true"
        tty: true
        network_mode: host
        restart: unless-stopped

После \`docker compose up -d\` контейнер будет запущен и готов к работе. В статьях, что я приводил выше описано как сделать соединение VLESS+Reality в админке. Тут нужно отметить, что именно VLESS+Reality наиболее продвинутый способ и я рекомендую именно его. Но в админку надо для начала попасть, для чего нам и нужен будет Caddy. Прежде чем устанавливать Caddy в OPNSense-\>System-\>Settings поменяйте порт, на котором будет работать админка OPNSense например на 8443, позже к ней мы доступ сделаем тоже через Caddy, мне так кажется удобным. Далее заходите в OPNSense и устанавливаете community plugin os-caddy.

После установки плагина нужно сделать несколько настроек в плагине. Тут важно понимать, что есть два разных механизма, первый это reverse proxy и второй L4 Proxy в Caddy. Первым мы предоставим доступ к админкам, он работает на L7 сетевом уровне, а вторым будет работать VLESS+Reality так как он нуждается в L4 уровне прокисрования. Вот такой конфиг для Caddy должен получится в итоге:

Скрытый текст

    # DO NOT EDIT THIS FILE -- OPNsense auto-generated file


    # caddy_user=root

    # Global Options
    {
        log {
            output net unixgram//var/run/caddy/log.sock {
            }
            format json {
                time_format rfc3339
            }
            level DEBUG
        }

        servers {
            protocols h1 h2

            log_credentials
            listener_wrappers {
                layer4 {
                    import /usr/local/etc/caddy/caddy.d/*.layer4listener

                    @bea500f4-0d14-431d-9bba-87ec8df540f1 tls sni google.com www.google.com

                    route @bea500f4-0d14-431d-9bba-87ec8df540f1 {
                        proxy tcp/10.0.0.199:8443 {
                        }
                    }
                }
                tls
            }
        }

        layer4 {
            import /usr/local/etc/caddy/caddy.d/*.layer4global
        }

        email твоя почта
        grace_period 10s
        import /usr/local/etc/caddy/caddy.d/*.global
    }

    # Reverse Proxy Configuration


    # Layer4 default HTTP port
    :80 {
    }
    # Layer4 default HTTPS port
    :443 {
    }

    xray.home.arpa {
        tls /usr/local/etc/caddy/certificates/690295ae86977.pem /usr/local/etc/caddy/certificates/690295ae86977.key {
        }

        handle {
            @f376fb8a-12fa-4dd2-a249-7b9b93e20e38 {
                not client_ip 192.168.1.0/24
            }
            handle @f376fb8a-12fa-4dd2-a249-7b9b93e20e38 {
                abort
            }

            reverse_proxy 10.0.0.199:2053 {
            }
        }
    }

    opnsense.home.arpa {
        tls /usr/local/etc/caddy/certificates/690295ae86977.pem /usr/local/etc/caddy/certificates/690295ae86977.key {
        }

        handle {
            @f376fb8a-12fa-4dd2-a249-7b9b93e20e38 {
                not client_ip 192.168.1.0/24
            }
            handle @f376fb8a-12fa-4dd2-a249-7b9b93e20e38 {
                abort
            }

            reverse_proxy https://10.0.0.1:8443 {
                transport http {
                    tls_insecure_skip_verify
                }
            }
        }
    }

    import /usr/local/etc/caddy/caddy.d/*.conf

Тут есть несколько моментов в конфиге Caddy. Первый 10.0.0.199 это адрес, который был выдан для xray сервиса на DHCP сервере OPNSense, для внутренних сервисов я не стал отключать DHCP, но зафиксировал lease для данного xray сервера как статический. 10.0.0.199:8443 на этом порту слушает VLESS+Reality сервер, для соединения клиентов, когда я создавал Inbound в 3x ui xray, я задал этот порт. Обратите внимание, что этот порт разрешен в firewall автоматическими правилами, так как админка на него переключена, если порт будет другой, нужно будет специально разрешить его на LAN интерфейсе. Сама же админка доступна на xray сервере в LXC контейнере на 10.0.0.199:2053. Все эти настройки делаются из OPNSense WebUI после установки плагина, конфиг руками писать не нужно. Да, не забудьте не только включить Cadyy плагин в OPNSense, но и разрешить L4 Proxy в нем. При этом обратите внимание на access \`not client_ip 192.168.1.0/24\`, это запретит доступ к данным локальным доменам из интернета, я не стал его открывать. Иначе знатоки подменят параметры запроса и получат возможность ковырять админку, но нужно домен угадать будет.

Суть проксирования в том, что Caddy опознает на L4 Proxy SNI google.com и тогда передаст сырой поток данных на xray сервер без всякого reverse proxy. После этого если xray сервер опознает VLESS коннект, то он будет работать с ним, если нет, то он прикинется google.com причем со всеми сертификатами и прочим, неотличимо. Если же Caddy не опознает SNI, то начнет работать reverse proxy и если домен будет опознан как xray.home.arpa, то xray сервер админка будет показана, а для домена opensense.home.arpa будет показана админка OPNSense. Caddy тут хороша тем, что очень просто настраивается.

Теперь Caddy работает на 443 и 80 портах на всех интерфейсах, а админку OPNSense мы перенесли на 8443 порт, так что заходить в нее можно пока на любом порту и будет работать.

Далее переходим к настройками VPS у провайдера, это наш внешний сервер, через который мы предоставим к домашним сервисам доступ из интернета. Тут все довольно просто, ставим на него ubuntu 25.04, делаем стандартные настройки безопасности вроде доступа к ssh по ключам и создания пользователя что бы не под root работать и ставим сеть.

Настройки сети такие. Первое нам нужно установить wireguard на VPS. Он нам нужен что бы обеспечить связь с домашним сервером где OPNSense. Конфиг wireguard такой:

Скрытый текст

    i3draven@gateway:~$ sudo cat /etc/wireguard/wg0.conf
    [Interface]
    Address = 10.100.0.1/30
    PrivateKey = это приватный ключ на VPS сервере сделанный
    ListenPort = 51820

    [Peer]
    PublicKey = это публичный ключ, сделанный на OPNSense
    AllowedIPs = 10.100.0.2/32, 10.0.0.0/24 

В конфиге Wireguard разрешен OPNSense, он клиент для VPS с адресом внутри тоннеля wg0 10.100.0.2/32, а так же для прохождения по тоннелю разрешены пакеты от сети 10.0.0.0/24 это пакеты от сервисов внутри моста service, который изолирует их в нашей внутренней сети. То есть наш xray сервер в итоге должен иметь возможность напрямую обратиться к VPS через этот тоннель как шлюзу 10.100.0.1.

Когда Wireguard на VPS настроен, нужно настроить nftables нам нужно делать NAT для пакетов, которые пришли от внутренних сервисов, например xray и делать NAT для пакетов, которые пришли в VPS и ушли через wg0 внутренним сервисам. То есть VPS будет маскировать как пакеты снаружи внутрь, так и пакеты изнутри наружу. Это, на мой взгляд самая простая конструкция из возможных. Вот настройки nftables:

Скрытый текст

    i3draven@gateway:~$ cat /etc/nftables.conf 
    #!/usr/sbin/nft -f

    flush ruleset

    define WAN_IF = "eth0"
    define WG_IF = "wg0"
    define WEB_SERVER_IP = 10.100.0.2

    table ip filter {
        chain LOG_AND_DROP {
            # Можно посмотреть sudo dmesg что блокировалось
            log prefix "[NFTABLES DROP] " level info counter
            drop
        }

        chain INPUT {
            type filter hook input priority filter; policy drop;
            ct state established,related accept

            # Явно разрешаем весь входящий трафик из туннеля от OPNsense
            # Это для того что бы ответные ACK пакеты просто принимались
            iifname $WG_IF accept

            iifname "lo" accept
            ip protocol icmp icmp type { echo-request, echo-reply, destination-unreachable } accept
            # Это SSH на порту таком
            tcp dport 8113 ct state new limit rate 5/minute accept
            # Это wireguard
            udp dport 51820 accept
            jump LOG_AND_DROP
        }

        chain FORWARD {
            type filter hook forward priority filter; policy drop;
            ct state established,related accept
            # Прокидываем трафик на Caddy
            iifname $WAN_IF ip daddr $WEB_SERVER_IP tcp dport { 80, 443 } accept        
            iifname $WAN_IF ip daddr $WEB_SERVER_IP udp dport { 80, 443 } accept # Добавим UDP для Xray
            # Разрешить трафику из VPN-сети выходить в интернет
            iifname $WG_IF oifname $WAN_IF accept
            
            jump LOG_AND_DROP
        }

        chain OUTPUT {
            type filter hook output priority filter; policy accept;
        }
    }

    table ip nat {
        chain PREROUTING {
            type nat hook prerouting priority dstnat; policy accept;
            # DNAT трафика из интернета на OPNsense
            iifname $WAN_IF tcp dport { 80, 443 } dnat to $WEB_SERVER_IP
            iifname $WAN_IF udp dport { 80, 443 } dnat to $WEB_SERVER_IP # И для UDP
        }

        chain POSTROUTING {
            type nat hook postrouting priority srcnat; policy accept;
            oifname $WAN_IF masquerade
            oifname $WG_IF masquerade
        }
    }

Вот это правило \`iifname "eth0" tcp dport { 80, 443 } dnat to 10.100.0.2\` пробрасывает все, что пришло из интернета на OPNSense, где слушает Caddy что бы определить какой из внутренних сервисов будет отвечать на запрос. Тем самым во внутренней сети мы можем по доменам иметь сколько угодно чего угодно. Вот эти два правила \`oifname "eth0" masquerade\`, \`oifname "wg0" masquerade\` как раз и занимаются маскировкой трафика.

Думаю это достаточно безопасная настройка nftables, если имеете советы по улучшению, было бы отлично. Но что бы не смотреть на постоянные сканирования портов я на firewall провайдера запретил все порты, кроме тех, что нужны, то есть ssh, wireguard, http, https. В противном случае ядро вашей VPS постоянно будет ловить пакеты от сканеров.

Настройка Wireguard невозможна с одной стороны, нужен как минимум публичный ключ от OPNSense, так что параллельно нужно настроить в OPNSense Wireguard. Я покажу часть конфигурации OPNSense с итогами, но настраивается все как и раньше в WebUI:

Скрытый текст

    <wireguard>
          <client version="1.0.0" persisted_at="1762127854.97">
            <clients>
              <client uuid="b0b854fc-86fc-4210-b11c-d9c0e419cb43">
                <enabled>1</enabled>
                <name>frontend_peer</name>
                <pubkey>key</pubkey>
                <psk/>
                <tunneladdress>0.0.0.0/0</tunneladdress>
                <serveraddress>addr VPS server</serveraddress>
                <serverport>51820</serverport>
                <keepalive>25</keepalive>
              </client>
            </clients>
          </client>
          <general version="0.0.1" persisted_at="1762133000.75">
            <enabled>1</enabled>
          </general>
    <server version="1.0.1" persisted_at="1762127854.97">
            <servers>
              <server uuid="99ddd6b8-baea-4856-abe3-ce3e65a3e235">
                <enabled>1</enabled>
                <name>frontend</name>
                <instance>0</instance>
                <pubkey>key</pubkey>
                <privkey>key</privkey>
                <port>51820</port>
                <mtu/>
                <dns/>
                <tunneladdress>10.100.0.2/32</tunneladdress>
                <disableroutes>1</disableroutes>
                <gateway/>
                <carp_depend_on/>
                <peers>b0b854fc-86fc-4210-b11c-d9c0e419cb43</peers>
                <debug>1</debug>
                <endpoint/>
                <peer_dns/>
              </server>
            </servers>
          </server>
        </wireguard>

Тут нужно распутаться с тем куда какие ключи поставить, но думаю разберетесь, публичный ключ OPNSense в итоге попадет в конфиг wg0.conf на VPS. Но самое главное тут в том, что при создании Wireguard Instance в OPNSense нужно обязательно указать опцию \`**Disable routes**\` это очень важно потому, что при дальнейших настройках связь с OPNSense без нее пропадет. И лишь после запрета создания routes можно сделать peer в котором в качестве allowed ips указать 0.0.0.0/0. Если маршруты создавать не запретить, то будет создан маршрут, который завернет весь трафик в Wireguard.

На вский случай, в Proxmox можно открыть консоль VM или контейнера и там все вернуть. Для OPNSense есть в консоли пункт 13 в меню, который позволяет вернуть бэкап настроек, каждый раз когда вы меняете настройки текущее состояние бэкапиться. Но я рекомендую просто делать snapshot в Proxmox и все, потом легко вернуть снимок виртуалки в состояние до опасных изменений.

Итак, в этот момент у нас есть настроенный тоннель от OPNSense до VPS, но нет маршрутов, так что он пока не работает. Нам нужно сделать еще несколько вещей. Первая это зайти в OPNSense-\>Interfaces-\>Assignments и добавить новый Wireguard VPN интерфейс в OPNSense с любым подходящим именем, я назвал WGVPS.

Так же нужно зайти в свойста OPNSense-\>Interfaces-\>{LAN, WAN, WGVPS} и снять галочку \` Block private networks\` у всех этих интерфейсов. Это уберет автоматически создаваемые правила firewall, которые заблокируют нам трафик через Wireguard тоннель. Это важный момент.

Потом нужно зайти в настройки WGVPS и поставить галочку \`Dynamic gateway policy\` это приведет к тому, что для динамического интерфейса WGVPS OPNSense создаст шлюз в OPNSense-\>System-\>Gateways-\>Configuration. Это важный момент, шлюз надо создать именно таким способом, не руками, так как VPN динамический интерфейс.

Когда шлюз создан мы можем перейти к настройкам firewall, вот конфиг:

Скрытый текст

    <filter>
    <rule uuid="631de44f-3131-4677-bc6e-7a0a0e72828f">
          <type>reject</type>
          <interface>wan</interface>
          <ipprotocol>inet</ipprotocol>
          <statetype>keep state</statetype>
          <descr>Block LAN to OpenWRT internet</descr>
          <direction>in</direction>
          <quick>1</quick>
          <source>
            <network>lan</network>
          </source>
          <destination>
            <any>1</any>
          </destination>
          <updated>
            <username>root@10.0.0.1</username>
            <time>1762217647.61</time>
            <description>/firewall_rules_edit.php made changes</description>
          </updated>
          <created>
            <username>root@10.0.0.1</username>
            <time>1762217620.69</time>
            <description>/firewall_rules_edit.php made changes</description>
          </created>
        </rule>

        <rule uuid="f6cf4478-9e1d-415e-b54f-6d42d0c0b264">
          <type>pass</type>
          <interface>wan</interface>
          <ipprotocol>inet</ipprotocol>
          <statetype>keep state</statetype>
          <descr>Acces from LAN to Firewall UI</descr>
          <direction>in</direction>
          <quick>1</quick>
          <protocol>tcp</protocol>
          <source>
            <address>192.168.1.0/24</address>
          </source>
          <destination>
            <address>(self),wanip</address>
            <port>443</port>
          </destination>
          <updated>
            <username>root@10.0.0.1</username>
            <time>1762086725.77</time>
            <description>/firewall_rules_edit.php made changes</description>
          </updated>
          <created>
            <username>root@192.168.1.132</username>
            <time>1761839287.60</time>
            <description>/firewall_rules_edit.php made changes</description>
          </created>
        </rule>
       <rule uuid="772ecf5a-9d0e-48e2-a8c5-7dcf448cf530">
          <type>pass</type>
          <interface>wan</interface>
          <ipprotocol>inet</ipprotocol>
          <statetype>keep state</statetype>
          <descr>Acces from LAN to Firewall UI</descr>
          <direction>in</direction>
          <quick>1</quick>
          <protocol>tcp</protocol>
          <source>
            <address>192.168.1.0/24</address>
          </source>
          <destination>
            <address>(self),wanip</address>
            <port>8443</port>
          </destination>
          <updated>
            <username>root@192.168.1.132</username>
            <time>1761927485.66</time>
            <description>/firewall_rules_edit.php made changes</description>
          </updated>
          <created>
            <username>root@192.168.1.132</username>
            <time>1761927485.66</time>
            <description>/firewall_rules_edit.php made changes</description>
          </created>
        </rule>
      <rule uuid="70b3f1f0-8563-4ebe-b74f-d7f806abaa40">
          <type>pass</type>
          <interface>wan</interface>
          <ipprotocol>inet</ipprotocol>
          <statetype>keep state</statetype>
          <descr>Acces from LAN to Firewall UI</descr>
          <direction>in</direction>
          <quick>1</quick>
          <protocol>tcp</protocol>
          <source>
            <address>192.168.1.0/24</address>
          </source>
          <destination>
            <address>(self),wanip</address>
            <port>22</port>
          </destination>
          <updated>
            <username>root@192.168.1.132</username>
            <time>1761848634.17</time>
            <description>/firewall_rules_edit.php made changes</description>
          </updated>
          <created>
            <username>root@192.168.1.132</username>
            <time>1761846900.64</time>
            <description>/firewall_rules_edit.php made changes</description>
          </created>
        </rule>
        <rule uuid="dc0e35d2-53bb-4987-9dfa-60455761f285">
          <type>pass</type>
          <interface>wan</interface>
          <ipprotocol>inet</ipprotocol>
          <statetype>keep state</statetype>
          <descr>Access from LAN to DNS</descr>
          <direction>out</direction>
          <quick>1</quick>
          <protocol>udp</protocol>
          <source>
            <any>1</any>
          </source>
          <destination>
            <any>1</any>
            <port>53</port>
          </destination>
          <updated>
            <username>root@192.168.1.132</username>
            <time>1761850932.61</time>
            <description>/firewall_rules_edit.php made changes</description>
          </updated>
          <created>
            <username>root@192.168.1.132</username>
            <time>1761850932.61</time>
            <description>/firewall_rules_edit.php made changes</description>
          </created>
        </rule>

    <rule uuid="e1bf2add-31eb-4b58-b824-35fbc9c0401d">
          <type>pass</type>
          <interface>lan</interface>
          <ipprotocol>inet</ipprotocol>
          <statetype>keep state</statetype>
          <descr>Allow services to reply to proxy and use DNS</descr>
          <direction>in</direction>
          <quick>1</quick>
          <source>
            <network>lan</network>
          </source>
          <destination>
            <network>(self)</network>
          </destination>
          <updated>
            <username>root@10.0.0.1</username>
            <time>1762560063.06</time>
            <description>/firewall_rules_edit.php made changes</description>
          </updated>
          <created>
            <username>root@10.0.0.1</username>
            <time>1762560063.06</time>
            <description>/firewall_rules_edit.php made changes</description>
          </created>
        </rule>

     <rule uuid="9c026114-0bcb-4419-8cbf-64f256f33bfc">
          <type>pass</type>
          <interface>lan</interface>
          <ipprotocol>inet</ipprotocol>
          <statetype>keep state</statetype>
          <descr>All LAN traffic to VPS</descr>
          <gateway>WGVPS_GW</gateway>
          <direction>in</direction>
          <quick>1</quick>
          <source>
            <network>lan</network>
          </source>
          <destination>
            <address>Local_networks</address>
            <not>1</not>
          </destination>
          <updated>
            <username>root@10.0.0.1</username>
            <time>1762198751.01</time>
            <description>/firewall_rules_edit.php made changes</description>
          </updated>
          <created>
            <username>root@10.0.0.1</username>
            <time>1762198685.98</time>
            <description>/firewall_rules_edit.php made changes</description>
          </created>
        </rule>
    <rule uuid="75537f75-d7c0-4757-99fc-df2056ef06c1">
          <type>block</type>
          <interface>lan</interface>
          <ipprotocol>inet</ipprotocol>
          <statetype>keep state</statetype>
          <descr>Block access from Services to all other local networks</descr>
          <direction>in</direction>
          <quick>1</quick>
          <source>
            <network>lan</network>
          </source>
          <destination>
            <address>Local_networks</address>
          </destination>
          <updated>
            <username>root@10.0.0.1</username>
            <time>1762560079.99</time>
            <description>/firewall_rules_edit.php made changes</description>
          </updated>
        </rule>



    <rule uuid="c404dacf-7834-4c79-b186-c20654166929">
          <type>block</type>
          <interface>opt1</interface>
          <ipprotocol>inet</ipprotocol>
          <statetype>keep state</statetype>
          <descr>Block VPS internet to WAN</descr>
          <direction>in</direction>
          <quick>1</quick>
          <source>
            <any>1</any>
          </source>
          <destination>
            <network>wan</network>
          </destination>
          <updated>
            <username>root@10.0.0.1</username>
            <time>1762218108.82</time>
            <description>/firewall_rules_edit.php made changes</description>
          </updated>
          <created>
            <username>root@10.0.0.1</username>
            <time>1762217998.73</time>
            <description>/firewall_rules_edit.php made changes</description>
          </created>
        </rule>

       <rule uuid="34289115-42de-41e4-afb9-db2835ebed67">
          <type>pass</type>
          <interface>opt1</interface>
          <ipprotocol>inet</ipprotocol>
          <statetype>keep state</statetype>
          <descr>Allow WG to Caddy HTTPS</descr>
          <direction>in</direction>
          <quick>1</quick>
          <protocol>tcp</protocol>
          <source>
            <any>1</any>
          </source>
          <destination>
            <network>(self)</network>
            <port>443</port>
          </destination>
          <updated>
            <username>root@10.0.0.1</username>
            <time>1761938713.37</time>
            <description>/firewall_rules_edit.php made changes</description>
          </updated>
          <created>
            <username>root@10.0.0.1</username>
            <time>1761937442.75</time>
            <description>/firewall_rules_edit.php made changes</description>
          </created>
        </rule>
      <rule uuid="5d4cb405-c3d4-491b-a185-6cd5121ece4b">
          <type>pass</type>
          <interface>opt1</interface>
          <ipprotocol>inet</ipprotocol>
          <statetype>keep state</statetype>
          <descr>Allow WG to Caddy HTTP</descr>
          <direction>in</direction>
          <quick>1</quick>
          <protocol>tcp</protocol>
          <source>
            <any>1</any>
          </source>
          <destination>
            <network>(self)</network>
            <port>80</port>
          </destination>
          <updated>
            <username>root@10.0.0.1</username>
            <time>1761938704.08</time>
            <description>/firewall_rules_edit.php made changes</description>
          </updated>
          <created>
            <username>root@10.0.0.1</username>
            <time>1761938704.08</time>
            <description>/firewall_rules_edit.php made changes</description>
          </created>
        </rule>
      </filter>

Приводить правила из самого pf с консоли я не стал потому, что они там не будут иметь названий и их намного больше, там много автоматических. Это те, что я настраивал из OPNsense Web UI, цитата из его конфига. Правила разрешают доступ по HTTP HTTPS к Caddy, который слушает на этих портах что бы было доступ к админке OPNSense и прочему. Возможно часть правил нужно создать в самом начале что бы доступ открыть, разберетесь.

Самое важное правило \`All LAN traffic to VPS\` именно оно заставляет все LAN сервисы ходить в интернет через VPS. Без него клиенты, подключенные к xray будут видеть ваш домашний IP так как сервисы будут ходить в интернет через домашний роутер! Там список Local networks это просто OPNSense-\>Firewall-\>Aliases и там я сделал имя для \`192.168.1.0/24\`,\`192.168.2.0/24\`,\`10.0.0.0/24\` сетей.

Наконец, последний этап, мы отказались от создания маршрутов для Wireguard через опцию \`Disable routes\`, так что нужно добавить маршрут в сеть за wg0 руками. Это делается в OPNSense-\>System-\>Routes-\>Configuration. Создаем маршрут для сети 10.100.0.0/24 (это сеть для Wireguard) и в качестве шлюза выбираем автоматически созданный для Wireguard шлюз, у меня он называется WGVPS_GW. Если бы мы шлюз руками сделали, это не сработало бы, нужен именно динамически создаваемый шлюз. Иначе маршрут сделать можно, но он только в интерфейсе будет показан, но не применится. В итоге пакеты, которые летят от xray в wg0 будут знать куда лететь.

При этом если бы мы не сделали \`Disable routes\` и 0.0.0.0/0 как разрешенные адреса для Wireguard, то у меня никаким способом не получилось завернуть трафик так что бы все внутренние сервисы из LAN ходили в интернет через VPS, а не через домашний роутер. Оставлять им выход через домашний роутер я посчитал небезопасным. Может быть кто-то в комментариях предложит лучший способ. Не безопасно это потому, в том числе, что любой клиент видит ваш домашний IP.

Что мы имеем в итоге:

1.  Доступ ко всем админкам, Proxmox, OPNSense, 3x UI xray, есть через связку OPNSense+Caddy. Доступ только из локальной сети.

2.  Есть тоннель между OPNSense и VPS за счет которого все внутренние сервисы могут ходить в интернет и быть видны в интернете за счет NAT на VPS.

3.  Локальные сервисы тем не менее могут ходить на разрешенные порты в локальной сети, например 53 порт для DNS. В целом, настройки DNS DHCP я опустил, они не обязательны, но конечно к VPS можно прикрепить доменное имя. Так же обратите внимание, что xray генерирует ссылку подключения с неверным доменом, он берет его из браузера судя по всему, а нужно будет там поставить домен VPS и порт 443, который является фронтом для наших домашних сервисов.

4.  Коннект к VPS выглядит так будто на нем стоит xray, но на деле он на домашнем сервере, с бесплатными бэкапами, относительно провайдерского безграничным диском и так далее. То есть в данную конструкцию легко добавить и другие сервисы, просто добавляя домены в Caddy и при этом сохранив xray как скрытый сервис.

Задачи со звездочкой:

1.  Настроить на OpenWrt QoS что бы регулировать ширину канала для дома и сервисов.

2.  Применить Lynis или аналоги для сканирования безопасности.

3.  Поставить среди сервисов автоматический сканер безопасности, который будет предупреждать если вы случайно накосячите.

4.  Подумать как сделать эту конструкцию с помощью snat (output nat) на OPNSense. Может будет удобнее.

5.  Специально для снижения пинга в играх может конкретно xray перенести на саму vps, изучить возможности реверс прокси у xray.

6.  Добавить на VPS второй внешний IP адрес чтобы на него перенести ssh и wireguard для большей маскировки. Тогда на адресе с xray останется только "сайт" под который вы замаскировались.

7.  Для экономных, если смириться с тем, что будет терминация https трафика и обойтись без xray vless+reality, то собственную VPS можно заменить на бесплатный Cloudflare tunnel или аналог.

8.  Для нервных: можно на уровне proxmox моста services порезать мост на VLAN-ы и сделать opnsense для них шлюзом, для полной изоляции сервисов на случай взлома.

9.  В Unbound DNS в OPNSense для LAN сети где xray включить блоклисты на DNS. Это позволит резать рекламу и шпиенское ПО, ладно бы мешало, оно сильно садит батарею телефона, очень вероятно он будет работать раза в два дольше от этого, если вы подключитесь к личному xray и правильно настроите NAT что бы все запросы от клиентов на 53 порт DNS шли через ваш Unbound DNS.

P.S. Если будут рекомендации по улучшению, буду признателен, между подобными заходами покрутить что-то у меня бывает годы проходят и я успеваю забыть детали. При этом OPNSense, xray я и вовсе настраивал впервые. Да, и возможно я найду какие-то проблемы в настройках позже, написал пока не забыл :) В общем советы приветствуются. Так же хочу напомнить о сайтах типа [такого](https://ipleak.net/), которые при работающем xray позволяют посмотреть утекают ли ваши адреса в сеть при открытии сайтов и например включить DNS в 3x-ui, а так же предпринять другие меры безопасности.

UPD1: Задача 4 и 9 сделаны. После 4 впн стал быстрее намного работать. Для 9 я так же добавил NAT rdr правило что бы все, что даже напрямую лезет куда угодно на 53 порт, попадало в мой Unbound DNS, а то есть гора скрытых хитрецов.

UPD2: Подумалось, что стоило упомянуть. Так как в статье предполагается, что внутренние сервисы находятся в одной изолированной сети services, трафик от caddy до сервисов должен быть https , что бы взлом одного из ваших сервисов не привел к перехвату данных соседних совсем уж просто. Ну, либо п.8 из задач со звездочкой становится обязатнльным. В следующей статье поговорим о безопасности побольше.

UPD3: Нужно отметить, что в современных условиях wireguard как средство связи между домашним сервером и VPS может быть не надежным. Но суть конструкции в том, что мы организуем связь домашнего сервера OPNSense как клиента к внешнему серверу VPS. Само же средство связи может быть любым. Например можно в сети services сделать VM/LXC в котором будет xray vless+reality клиент к серверу xray, установленному на VPS. Это усложнит конструкцию, но не поменяет ее суть. Просто маршрутизация внутри домашнего сервера будет не "внутренний сервис в сети services"-\>OPNSense-\>Wireguard-\>VPS-\>Internet, а "внутренний сервис в сети services"-\>OPNSense-\>VM/LXC xray клиент-\>OPNSense-\>домашний роутер-\>VPS-\>Internet, где xray клиент-\>сервер вместо wireguard-\>VPS. При этом xray в статье выбран только потому, что он позволяет продемонстрировать что такое L4 Proxy. Все остальное дело изобретательности читателя.

UPD4: Для тех кому l4 proxy не нужен, можно взглянуть на проект [https://github.com/fosrl/pangolin](https://Pangolin)

UPD5: Для ограничения доступа к админским ресурсам через caddy обратите внимание на аутентификацию через клиентский сертификат. Его можно сгенерировать прямо в OPNSense и после этого использовать в клиентском браузере и caddy. Очень удобно так как браузер о сертификате спросит только один раз и при этом у вас есть защищенная админка очередного дырявого без такой защиты, написанного черти кем и как сервиса, который тем не менее вам нужен :)
