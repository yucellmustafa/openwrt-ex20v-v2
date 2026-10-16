# TP-Link EX20v v2 OpenWrt

TP-Link EX20v v2 (EcoNet EN7528) için topluluk tarafından geliştirilen OpenWrt
portu. Proje, cihaz desteğini ve performans yamalarını tekrarlanabilir bir
derleme akışıyla sunar.

> [!WARNING]
> Bu proje resmî OpenWrt desteği değildir. Firmware yüklemek stok sistemi siler
> ve cihazı kullanılamaz hale getirme riski taşır. Cihaz size veya servis
> sağlayıcınıza ait olabilir; bütün sorumluluk kullanıcıya aittir.

## Durum

| Özellik | Durum |
| --- | --- |
| Ethernet: 1 WAN + 3 LAN | Çalışıyor |
| Kablolu PPE donanım offload | Çalışıyor |
| 2,4 GHz ve 5 GHz Wi-Fi | Çalışıyor |
| İnternet -> Wi-Fi PPE offload | Çalışıyor |
| USB, LED'ler, reset düğmesi | Çalışıyor |
| UBI overlay ve LuCI sysupgrade | Çalışıyor |
| DSL ve telefon (FXS) | Desteklenmiyor; açık kaynak sürücü yok |

Varsayılan ağ yapılandırması Türk Telekom için `eth0.35` üzerinde PPPoE'dir.
LAN arayüzü `eth0.1`, yönetim adresi `192.168.1.1` olarak gelir.

## Dahili Paketler

Ana sistem imajında aşağıdaki paketler ve paket tanımlarındaki tüm bağımlılıklar
hazır gelir:

- `luci-app-sqm`
- `luci-app-irqbalance`
- `wpad-mbedtls`

Bu seçim SQM için `kmod-sched-cake` ve `kmod-ifb` dahil gerekli çekirdek
modüllerini de imaja dahil eder. Tam WPA/EAP desteği için varsayılan
`wpad-basic-mbedtls`, `wpad-mbedtls` ile değiştirilir.

Kurtarma imajı 6 MiB kernel bölümü sınırı nedeniyle yalnızca kurtarma için
gerekli bileşenleri içerir.

> [!IMPORTANT]
> SQM kullanırken yazılım ve donanım flow offloading'i kapatın. Offload edilen
> trafik SQM kuyruğunu atladığı için bant genişliği yönetimi doğru çalışmaz.

## Kurulum

Kurulum, yedekleme, slot yönetimi ve stok firmware'e dönüş işlemleri ayrı
[TP-Link EX20v v2 OpenWrt Kurulum Aracı](https://github.com/yucellmustafa/tplink_ex20v_openwrt_tool)
ile yapılır. Bu depo kurulum scripti içermez.

Araç aşağıdaki dört imajı kullanır:

| Dosya | Amaç |
| --- | --- |
| `ex20v-rescue.bin` | Slot A kurtarma sistemi |
| `ex20v-kernel.bin` | Slot B ana kernel |
| `ex20v-ubi.bin` | Ana UBI/rootfs |
| `ex20v-v2-sysupgrade.bin` | LuCI veya `sysupgrade` güncellemesi |

Hazır imajlar GitHub Releases veya başarılı GitHub Actions derlemesinin
`ex20v-v2-images` artifact'ı üzerinden alınabilir.

## Derleme

OpenWrt kaynağı `2d1c9d1b85b8e5204420c0dd71a8421ddfe1dfd4` commit'inden
derlenir. Paketler derleme sırasında OpenWrt'nin güncel standart feed'lerinden
alınır.

Ubuntu 22.04 veya 24.04 üzerinde:

```bash
sudo apt-get update
sudo apt-get install -y build-essential clang flex bison g++ gawk gcc-multilib \
  g++-multilib gettext git libncurses-dev libssl-dev python3-setuptools rsync \
  swig unzip zlib1g-dev file wget

git clone https://github.com/openwrt/openwrt.git
cd openwrt
git checkout 2d1c9d1b85b8e5204420c0dd71a8421ddfe1dfd4
git -c user.name=builder -c user.email=builder@localhost am ../patches/*.patch
./scripts/feeds update packages luci
./scripts/feeds install luci luci-app-sqm luci-app-irqbalance

printf '%s\n' \
  CONFIG_TARGET_econet=y \
  CONFIG_TARGET_econet_en7528=y \
  CONFIG_TARGET_MULTI_PROFILE=y \
  CONFIG_TARGET_PER_DEVICE_ROOTFS=y \
  CONFIG_TARGET_DEVICE_econet_en7528_DEVICE_tplink_ex20v-v2=y \
  CONFIG_TARGET_DEVICE_econet_en7528_DEVICE_tplink_ex20v-v2-rescue=y \
  CONFIG_TARGET_ROOTFS_INITRAMFS=y > .config

make defconfig
make download -j8
make -j"$(nproc)"
```

Çıktılar `bin/targets/econet/en7528/` dizininde oluşur. GitHub Actions aynı
adımları uygular, kernel imajlarının 6 MiB sınırını denetler ve SHA256 listesini
üretir.

## Flash Yerleşimi

| Adres | Kullanım |
| --- | --- |
| `0x0000000` - `0x037ffff` | Bootloader, cihaz ayarları, MAC ve Wi-Fi EEPROM; korunur |
| `0x0380000` | Slot A kurtarma kernel'i, 6 MiB |
| `0x2980000` | Slot B ana kernel, 6 MiB |
| `0x2f80000` - `0x1bffffff` | UBI: rootfs ve kalıcı ayarlar |
| `0x1c000000` ve sonrası | BMT alanı; korunur |

Bootloader `0x240000` adresindeki ASCII bayta göre slot seçer: `0` slot A,
`1` slot B.

## Performans Notları

PPE donanım offload, NAT/VLAN/PPPoE akışlarını CPU dışına taşır. EN7528'de WED
olmadığı için internetten Wi-Fi'ye giden akışlarda stok firmware'deki ExtIf
yaklaşımına benzer CPU çıkışlı hızlı yol kullanılır.

Kontrol için:

```sh
nft list flowtables
grep -E 'repl_ok|rx_cpu' /sys/kernel/debug/econet_ppe/diag
```

`rx_cpu_nomatch` sayacının sıfırda kalması beklenir. Wi-Fi hızlı yolunu geçici
olarak kapatmak için:

```sh
echo 0 > /sys/module/econet_eth/parameters/wifi_offload
```

## Yamalar

| Yama | İçerik |
| --- | --- |
| `0001` | EX20v v2 DTS, imaj tanımları, LED/ağ ayarları ve dahili paketler |
| `0002` | PPE destekli EcoNet Ethernet sürücüsü ve port LED'leri |
| `0003` | RPS, ağ, saat dilimi ve çalışma ayarları |
| `0004` | Firewall4 VLAN aygıtlarını PPE fiziksel aygıtına eşler |
| `0005` | MT7915 DBDC 5 GHz için ayrı MAC adresi |
| `0006` | Wi-Fi hızlı yolu için `br-lan` flowtable desteği |
| `0007` | İnternet -> Wi-Fi PPE hızlı yolu |
| `0008` | Wi-Fi PPE hızlı yolunu varsayılan etkinleştirir |
| `0009` | Wi-Fi LED polaritesini ve sabit radyo durumu tetikleyicisini düzeltir |

## Lisans ve Katkılar

Yamalar ve OpenWrt değişiklikleri GPL-2.0 kapsamında sunulur. Proje;
[OpenWrt](https://openwrt.org/),
[cjdelisle/econet_eth](https://github.com/cjdelisle/econet_eth) ve
[ozgunokan/openwrt-tplink-ex20v](https://github.com/ozgunokan/openwrt-tplink-ex20v)
çalışmalarından yararlanır.

Bu proje TP-Link, Türk Telekom, EcoNet, MediaTek veya OpenWrt projesiyle resmî
olarak bağlantılı değildir ve bu kuruluşlar tarafından onaylanmamıştır.
