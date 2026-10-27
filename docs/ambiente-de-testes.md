# Ambiente de testes — DeX Station

> Gerado automaticamente por `scripts/coleta-ambiente.sh` em 2026-10-21T10:46:29-03:00
> Host: `asus`

> IPs privados mascarados para publicação (o arquivo original fica só na máquina).
## Resumo

| Campo | Valor |
|---|---|
| Modelo | ASUSTeK Computer INC. 1015BX |
| Distribuição | Void Linux |
| Kernel | 6.18.56_1 |
| Arquitetura | x86_64 |
| CPU | AMD C-60 APU with Radeon(tm) HD Graphics |
| RAM total | 1.5 GB |

---

## 1. Identificação da máquina

**Data e hora**

```
$ date -Iseconds
2026-10-21T10:46:29-03:00
```

**Modelo / fabricante (DMI)**

```
$ cat /sys/devices/virtual/dmi/id/sys_vendor /sys/devices/virtual/dmi/id/product_name /sys/devices/virtual/dmi/id/product_version 2>/dev/null
ASUSTeK Computer INC.
1015BX
x.x
```

**Modelo (dmidecode, se disponível)**

```
$ sudo -n dmidecode -s system-product-name 2>/dev/null || echo 'precisa de root — pular'
precisa de root — pular
```

**Número de série**

```
$ sudo -n dmidecode -s system-serial-number 2>/dev/null || echo 'precisa de root — pular'
precisa de root — pular
```

**Tempo ligado**

```
$ uptime -p 2>/dev/null || uptime
up 19 minutes
```

## 2. CPU e memória

**lscpu**

```
$ lscpu
Architecture:                            x86_64
CPU op-mode(s):                          32-bit, 64-bit
Address sizes:                           36 bits physical, 48 bits virtual
Byte Order:                              Little Endian
CPU(s):                                  2
On-line CPU(s) list:                     0,1
Vendor ID:                               AuthenticAMD
Model name:                              AMD C-60 APU with Radeon(tm) HD Graphics
CPU family:                              20
Model:                                   2
Thread(s) per core:                      1
Core(s) per socket:                      2
Socket(s):                               1
Stepping:                                0
Frequency boost:                         enabled
CPU(s) scaling MHz:                      91%
CPU max MHz:                             1000.0000
CPU min MHz:                             800.0000
BogoMIPS:                                1995.59
Flags:                                   fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush mmx fxsr sse sse2 ht syscall nx mmxext fxsr_opt pdpe1gb rdtscp lm constant_tsc rep_good nopl nonstop_tsc cpuid extd_apicid aperfmperf pni monitor ssse3 cx16 popcnt lahf_lm cmp_legacy svm extapic cr8_legacy abm sse4a misalignsse 3dnowprefetch ibs skinit wdt cpb hw_pstate vmmcall arat npt lbrv svm_lock nrip_save pausefilter
Virtualization:                          AMD-V
L1d cache:                               64 KiB (2 instances)
L1i cache:                               64 KiB (2 instances)
L2 cache:                                1 MiB (2 instances)
NUMA node(s):                            1
NUMA node0 CPU(s):                       0,1
Vulnerability Gather data sampling:      Not affected
Vulnerability Ghostwrite:                Not affected
Vulnerability Indirect target selection: Not affected
Vulnerability Itlb multihit:             Not affected
Vulnerability L1tf:                      Not affected
Vulnerability Mds:                       Not affected
Vulnerability Meltdown:                  Not affected
Vulnerability Mmio stale data:           Not affected
Vulnerability Old microcode:             Not affected
Vulnerability Reg file data sampling:    Not affected
Vulnerability Retbleed:                  Not affected
Vulnerability Spec rstack overflow:      Not affected
Vulnerability Spec store bypass:         Vulnerable
Vulnerability Spectre v1:                Mitigation; usercopy/swapgs barriers and __user pointer sanitization
Vulnerability Spectre v2:                Mitigation; Retpolines; STIBP disabled; RSB filling; PBRSB-eIBRS Not affected; BHI Not affected
Vulnerability Srbds:                     Not affected
Vulnerability Tsa:                       Not affected
Vulnerability Tsx async abort:           Not affected
Vulnerability Vmscape:                   Not affected
```

**Memória (free -h)**

```
$ free -h
               total        used        free      shared  buff/cache   available
Mem:           1.5Gi       470Mi       384Mi       4.2Mi       858Mi       1.1Gi
Swap:          7.8Gi          0B       7.8Gi
```

**Memória útil (MemAvailable)**

```
$ grep -E 'MemTotal|MemAvailable|SwapTotal' /proc/meminfo
MemTotal:        1612116 kB
MemAvailable:    1130156 kB
SwapTotal:       8220668 kB
```

**Swap / zram**

```
$ swapon --show 2>/dev/null || echo 'nenhum swap ativo'
NAME      TYPE      SIZE USED PRIO
/dev/sda3 partition 7.8G   0B   -2
```

**zram configurado?**

```
$ ls -1 /sys/block/ 2>/dev/null | grep -i zram || echo 'sem zram'
sem zram
```

## 3. Sistema operacional

**Distribuição (/etc/os-release)**

```
$ cat /etc/os-release
NAME="Void"
ID="void"
PRETTY_NAME="Void Linux"
HOME_URL="https://voidlinux.org/"
DOCUMENTATION_URL="https://docs.voidlinux.org/"
LOGO="void-logo"
ANSI_COLOR="0;38;2;71;128;97"

DISTRIB_ID="void"
```

**Kernel e arquitetura**

```
$ uname -a
Linux asus 6.18.56_1 #1 SMP PREEMPT_DYNAMIC Mon Oct 12 03:08:21 UTC 2026 x86_64 GNU/Linux
```

**Versão detalhada**

```
$ cat /etc/debian_version 2>/dev/null || cat /etc/lsb-release 2>/dev/null || echo 'n/d'
n/d
```

**Serviços com falha**

```
$ systemctl --failed --no-pager 2>/dev/null || echo 'sem systemd'
sem systemd
```

**Pacotes instalados (contagem)**

```
$ dpkg -l 2>/dev/null | grep -c '^ii' || echo 'n/d'
0
n/d
```

## 4. Vídeo e tela

**Resolução / saídas (xrandr)**

```
$ xrandr 2>/dev/null || echo 'xrandr indisponível (sem X ou sem pacote)'
xrandr indisponível (sem X ou sem pacote)
```

**Driver em uso**

```
$ lspci -nnk 2>/dev/null | grep -A3 -i 'vga\|display' || echo 'lspci indisponível'
00:01.0 VGA compatible controller [0300]: Advanced Micro Devices, Inc. [AMD/ATI] Wrestler [Radeon HD 6290] [1002:9807]
	Subsystem: ASUSTeK Computer Inc. Device [1043:84e4]
	Kernel driver in use: radeon
	Kernel modules: radeon, amdgpu
```

**Módulos de vídeo carregados**

```
$ lsmod 2>/dev/null | grep -E 'radeon|amdgpu|i915|drm' || echo 'nenhum'
amdgpu              15634432  0
drm_buddy              12288  1 amdgpu
amdxcp                 12288  1 amdgpu
drm_panel_backlight_quirks    12288  1 amdgpu
gpu_sched              69632  1 amdgpu
crc16                  12288  3 bluetooth,amdgpu,ext4
radeon               2187264  8
drm_ttm_helper         16384  2 amdgpu,radeon
ttm                   122880  3 amdgpu,radeon,drm_ttm_helper
agpgart                57344  2 radeon,ttm
drm_client_lib         12288  2 amdgpu,radeon
drm_exec               12288  2 amdgpu,radeon
i2c_algo_bit           24576  2 amdgpu,radeon
drm_suballoc_helper    16384  2 amdgpu,radeon
drm_display_helper    278528  2 amdgpu,radeon
drm_kms_helper        249856  5 drm_display_helper,amdgpu,radeon,drm_ttm_helper,drm_client_lib
drm                   802816  16 gpu_sched,drm_panel_backlight_quirks,drm_kms_helper,drm_exec,drm_suballoc_helper,drm_display_helper,drm_buddy,amdgpu,radeon,drm_ttm_helper,drm_client_lib,ttm,amdxcp
cec                    98304  2 drm_display_helper,amdgpu
video                  81920  3 asus_wmi,amdgpu,radeon
```

**Aceleração GL (se houver)**

```
$ glxinfo -B 2>/dev/null | head -20 || echo 'glxinfo indisponível'
```

## 5. USB

**Dispositivos USB (lsusb)**

```
$ lsusb 2>/dev/null || echo 'lsusb indisponível (apt install usbutils)'
Bus 001 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
Bus 002 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
Bus 002 Device 002: ID 13d3:5711 IMC Networks USB 2.0 UVC VGA WebCam
Bus 003 Device 001: ID 1d6b:0001 Linux Foundation 1.1 root hub
Bus 004 Device 001: ID 1d6b:0001 Linux Foundation 1.1 root hub
```

**Árvore USB (lsusb -t)**

```
$ lsusb -t 2>/dev/null || echo 'n/d'
/:  Bus 001.Port 001: Dev 001, Class=root_hub, Driver=ehci-pci/5p, 480M
/:  Bus 002.Port 001: Dev 001, Class=root_hub, Driver=ehci-pci/5p, 480M
    |__ Port 002: Dev 002, If 0, Class=Video, Driver=uvcvideo, 480M
    |__ Port 002: Dev 002, If 1, Class=Video, Driver=uvcvideo, 480M
/:  Bus 003.Port 001: Dev 001, Class=root_hub, Driver=ohci-pci/5p, 12M
/:  Bus 004.Port 001: Dev 001, Class=root_hub, Driver=ohci-pci/5p, 12M
```

**Versão do USB (lspci)**

```
$ lspci 2>/dev/null | grep -i usb || echo 'n/d'
00:12.0 USB controller: Advanced Micro Devices, Inc. [AMD/ATI] SB7x0/SB8x0/SB9x0 USB OHCI0 Controller
00:12.2 USB controller: Advanced Micro Devices, Inc. [AMD/ATI] SB7x0/SB8x0/SB9x0 USB EHCI Controller
00:13.0 USB controller: Advanced Micro Devices, Inc. [AMD/ATI] SB7x0/SB8x0/SB9x0 USB OHCI0 Controller
00:13.2 USB controller: Advanced Micro Devices, Inc. [AMD/ATI] SB7x0/SB8x0/SB9x0 USB EHCI Controller
```

## 6. Armazenamento

**Discos e partições (lsblk)**

```
$ lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINT,MODEL
NAME     SIZE TYPE FSTYPE MOUNTPOINT MODEL
sda    298.1G disk                   ST9320325AS
├─sda1   256M part vfat   /boot/efi  
├─sda2   290G part ext4   /          
└─sda3   7.8G part swap   [SWAP]     
```

**Uso de espaço (df -h)**

```
$ df -h -x tmpfs -x devtmpfs
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda2       285G  5.5G  265G   3% /
efivarfs         64K   44K   16K  74% /sys/firmware/efi/efivars
/dev/sda1       253M  142K  252M   1% /boot/efi
```

**Modelo do disco**

```
$ lsblk -dno MODEL 2>/dev/null
ST9320325AS
```

## 7. Rede

**Endereços IP**

```
$ ip -4 addr show 2>/dev/null | grep -E 'inet |^[0-9]' || ifconfig 2>/dev/null
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    inet 127.0.0.1/8 scope host lo
3: wlp1s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    inet 192.168.x.x/24 brd 192.168.x.x scope global dynamic noprefixroute wlp1s0
```

**Rota padrão**

```
$ ip route 2>/dev/null | head -5
default via 192.168.x.x dev wlp1s0 proto dhcp src 192.168.x.x metric 600 
192.168.x.x/24 dev wlp1s0 proto kernel scope link src 192.168.x.x metric 600 
```

**Wi-Fi / firmware**

```
$ lspci -nnk 2>/dev/null | grep -A3 -i 'network\|wireless' || echo 'n/d'
01:00.0 Network controller [0280]: Qualcomm Atheros AR9285 Wireless Network Adapter (PCI-Express) [168c:002b] (rev 01)
	Subsystem: Foxconn International, Inc. Device [105b:e049]
	Kernel driver in use: ath9k
	Kernel modules: ath9k
```

**Conectividade (ping curto)**

```
$ ping -c1 -W2 1.1.1.1 >/dev/null 2>&1 && echo 'internet OK' || echo 'sem internet'
internet OK
```

**SSH ativo?**

```
$ systemctl is-active ssh 2>/dev/null || systemctl is-active sshd 2>/dev/null || echo 'não'
não
```

## 8. Ferramentas do projeto

**adb**

```
$ adb version 2>/dev/null || echo 'adb não instalado'
adb não instalado
```

**scrcpy**

```
$ scrcpy --version 2>/dev/null || echo 'scrcpy não instalado'
scrcpy não instalado
```

**ffmpeg**

```
$ ffmpeg -version 2>/dev/null | head -1 || echo 'ffmpeg não instalado'
ffmpeg version 6.1.6 Copyright (c) 2000-2026 the FFmpeg developers
```

**Python**

```
$ python3 --version 2>/dev/null || echo 'n/d'
Python 3.14.8
```

**Localização dos binários**

```
$ command -v adb scrcpy ffmpeg python3 2>/dev/null || echo 'n/d'
/usr/bin/ffmpeg
/usr/bin/python3
```

## 9. Energia

**Bateria**

```
$ f=0; for b in /sys/class/power_supply/BAT*; do [ -d "$b" ] || continue; f=1; echo "$b"; cat "$b/status" "$b/energy_now" "$b/energy_full" 2>/dev/null; cat "$b/charge_now" "$b/charge_full" 2>/dev/null; done; [ $f -eq 1 ] || echo 'sem bateria detectada'
/sys/class/power_supply/BAT0
Discharging
2069000
3766000
```

**Temperatura (se disponível)**

```
$ sensors 2>/dev/null | head -20 || echo 'lm-sensors não instalado'
```

## 10. Celular conectado

> Preenchido apenas se o Galaxy estiver plugado e autorizado.

`adb` não instalado — seção deixada para preenchimento manual.

---

## 11. Fotos do setup (anexar no repositório)

Guardar em `docs/fotos/` com estes nomes:

- [ ] `01-netbook-fechado.jpg` — etiqueta do modelo embaixo
- [ ] `02-portas-laterais.jpg` — USB, HDMI, VGA, P2 e leitor de SD visíveis
- [ ] `03-bancada.jpg` — netbook + celular + cabo/hub na posição de uso
- [ ] `04-cabo-hub.jpg` — detalhe do hub/carregador usado
- [ ] `05-tela-ligada.jpg` — resolução e sistema em execução

## 12. Observações

- Hub / carregador usado:
- Cabo USB (modelo, comprimento):
- Estado da bateria do netbook (% e saúde):
- Distro de origem (antes de qualquer instalação): Void Linux
- Algo fora do padrão:

## 13. Conferência (P4)

- [ ] Todas as seções acima preenchidas
- [ ] Fotos anexadas em `docs/fotos/`
- [ ] Versões de `adb`/`scrcpy` batem com `build/VERSIONS.lock`
- [ ] Nenhum "n/d" sem justificativa
