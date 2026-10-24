# DeX Station OS — Plano de Desenvolvimento

**Alvo:** Asus Eee PC 1015BX — AMD C-60 (2 núcleos, 1,0 GHz, turbo 1,33 GHz, x86-64), APU
Radeon HD 6290, 2 GB DDR3, tela 10,1" 1024×600, HDD 5400 rpm, BIOS legado (sem UEFI),
3× USB 2.0, HDMI e VGA (ambas saída), P2 de áudio (fone/microfone).
**Fonte:** Samsung Galaxy com One UI 8 / Android 16.
**Objetivo:** transformar o netbook numa estação DeX dedicada — o celular faz o processamento,
o netbook é apenas monitor + teclado + trackpad.

> **Revisão 3 (24/10/2026):** portas do 1015BX corrigidas — ele **tem HDMI e VGA** (ambas
> somente saída) e **P2 de áudio**; §3 e §4.1 reescritos para explicar por que isso não muda
> a decisão pelo USB (as portas não recebem vídeo).
>
> **Revisão 2 (24/10/2026):** hardware corrigido de "Eee PC 1000 / Atom N450 / GMA 450" para
> **1015BX / AMD C-60 / Radeon HD 6290**; boot menu corrigido para **Esc**; adicionadas as
> consequências da APU (vídeo sem aceleração, RAM reservada), os pacotes de firmware que
> faltavam e ajustes de coerência entre seções.

---

## 0. Glossário

Termos usados no documento, em linguagem direta.

| Termo | O que é |
|---|---|
| **DeX** | Área de trabalho estilo PC da Samsung. No One UI 8 ela roda sobre o *Desktop Mode* do Android 16, que pode ser exibido num **display virtual** — uma tela "falsa" criada por software. |
| **Display virtual** | Tela adicional criada dentro do Android, sem monitor físico. É o que permite ver o DeX na janela do scrcpy. |
| **adb** | *Android Debug Bridge*. Canal de comando entre o computador e o celular, via USB. |
| **scrcpy** | Programa que espelha (ou abre um display virtual do) Android numa janela do Linux, enviando também os eventos de teclado e mouse. |
| **Kiosk** | Modo "quiosque": o equipamento liga direto na função desejada, sem login nem área de trabalho visível ao usuário. |
| **Daemon** | Programa que roda em segundo plano e reage a eventos (aqui: celular conectado/desconectado). |
| **Fallback** | Plano B automático: se o método principal falhar, tenta o próximo. |
| **zram** | Swap comprimido na própria RAM. Ajuda em máquinas com pouca memória e HDD lento, ao custo de CPU. |
| **debootstrap** | Ferramenta que monta um sistema Debian mínimo dentro de uma pasta, sem instalador. |
| **chroot** | Executar comandos "como se" estivesse dentro do sistema montado, para configurá-lo. |
| **GRUB** | Gerenciador de boot. Em BIOS legado usa o alvo `i386-pc` (sim, mesmo com kernel 64 bits). |
| **MBR / GPT** | Formatos de tabela de partição. MBR é o mais compatível com BIOS antigo. |
| **DPI** | Densidade lógica da tela. 160 DPI ≈ 1 pixel lógico = 1 pixel físico num painel 1024×600. |
| **Bitrate / FPS** | Taxa de dados e quadros por segundo do vídeo transmitido. Menos é mais leve. |
| **UVD3** | Bloco de decodificação de vídeo da APU. Existe no HD 6290, mas **não é usado** pela v1 (o scrcpy decodifica em software). |
| **Live USB** | Sistema que roda a partir do pendrive, sem instalar no disco interno. |

---

## 1. Sumário executivo

Não existe distribuição Linux pronta que transforme um netbook em estação DeX. O que será
desenvolvido é uma **imagem de sistema sob medida** — Debian 13 minimal + LXQt + scrcpy + adb
+ daemon de kiosk — que:

- liga sozinha, sem pedir senha;
- fica numa **tela de espera** monitorando a porta USB;
- quando o Galaxy é conectado, abre o **DeX real** em tela cheia;
- encaminha teclado e trackpad do netbook para o celular;
- ao desconectar o cabo, limpa a sessão e volta à tela de espera.

Duas variantes saem da **mesma** imagem:

| Variante | Artefato | Uso |
|---|---|---|
| **A — Live USB** | `dex-station.img` gravado num pendrive | Testar sem tocar no HD do netbook |
| **B — Instalação no HD** | `dex-station.img` gravado no disco interno | Sistema dedicado permanente |

A diferença entre as variantes é **o alvo da gravação**, não o conteúdo da imagem (§7 e §8).

---

## 2. Decisões de projeto

| # | Decisão | Justificativa |
|---|---|---|
| D1 | Conexão por **USB** (adb + scrcpy), não por vídeo | Um único cabo leva imagem, entrada e energia; dispensa placa de captura e dongle Bluetooth |
| D2 | Base **Debian 13 (trixie) minimal** + **LXQt** | Baixo consumo de RAM/CPU; LXQt é leve em X11 e tem autologin simples |
| D3 | **scrcpy ≥ 3.0** | Primeira versão com `--new-display` (display virtual), requisito do DeX no One UI 8 |
| D4 | Display virtual **1024×600 @ 160 DPI** | Casa 1:1 com o painel → nada é reescalonado → menos CPU para rasterizar |
| D5 | Sessão em **kiosk** (autologin + daemon), sem gerenciador de janelas visível | O usuário só interage com o DeX, nunca com o SO |
| D6 | Mesma imagem para USB e HD | Uma coisa só para manter; muda apenas o destino da gravação |
| D7 | Detecção por **consulta ao adb a cada 2 s** (v1) | Funciona em qualquer estado do udev e custa quase nada. udev fica como otimização (v2) |
| D8 | Limpeza por **cadeia de fallback em 3 níveis** | O netbook nunca fica sem imagem, mesmo se o display virtual falhar |
| D9 | Kernel e userland **amd64** (64 bits) | O C-60 é x86-64; o BIOS legado só afeta o boot (GRUB `i386-pc`), não o kernel |

---

## 3. Por que USB em vez de HDMI/VGA

| | Caminho de vídeo (tradicional) | Caminho USB (adotado) |
|---|---|---|
| Vídeo | Celular → adaptador USB-C/HDMI → placa de captura → `/dev/video0` | H.264/H.265 enviado pelo adb |
| Entrada | HID por dongle USB ou Bluetooth | Eventos de teclado/mouse injetados pelo adb |
| Cabos | 2–3 (vídeo, dados, carga) | 1 (dados + carga) |
| Hardware extra | Placa de captura (~R$ 60–120) e adaptador | Nenhum |
| Latência | encode no adaptador + decode no netbook | encode no celular + decode no netbook |
| Limite físico | **O 1015BX tem HDMI e VGA, mas as duas são saídas** — não há entrada de vídeo | — |

Duas observações importantes sobre as portas do 1015BX:

1. **HDMI e VGA são saídas.** Servem para levar a imagem do netbook a um monitor ou TV —
   útil, por exemplo, para apresentar a sessão numa tela maior. Elas **não recebem** sinal:
   não existe "entrada HDMI" num netbook de 2011. O vídeo do celular precisa chegar como
   **dado**, e é isso que o USB faz.
2. **Mesmo com uma placa de captura HDMI, o caminho USB continua melhor.** A captura entraria
   por USB 2.0 (largura de banda curta, ~500 mA), exigiria um adaptador USB-C→HDMI no celular
   — o que tira o DeX do modo display virtual — e custaria mais uma peça. O ganho seria nulo.
   A presença do HDMI, portanto, é conveniência para **saída**, não alternativa de entrada.

**O que se perde:** o handshake de modo TV / Smart View (o celular não "acha" que está num
dock). **O que se ganha:** o **DeX completo** (barra de tarefas, janelas redimensionáveis),
porque no One UI 8 a Samsung reconstruiu o DeX sobre o Desktop Mode nativo do Android 16,
que roda em display virtual.

---

## 4. Requisitos e matriz de compatibilidade

### 4.1 Hardware

| Item | Especificação do 1015BX | Observação |
|---|---|---|
| CPU | AMD C-60, 2 núcleos, 1,0 GHz (turbo 1,33 GHz), 9 W | Sem AVX; 64 bits; desempenho próximo de um Atom de mesma época |
| GPU | Radeon HD 6290 (integrada, até 384 MB da RAM compartilhada) | Driver `radeon`/Mesa; **não** é Intel (nada de `i915`/GMA) |
| RAM | 2 GB DDR3, canal único, 1 slot | ~1,7 GB utilizáveis após a reserva de vídeo. Oficialmente 2 GB máx.; há relatos de 4 GB funcionando |
| Tela | 10,1" 1024×600, fosca | Alvo do display virtual; sem escalonamento |
| Disco | HDD 5400 rpm, 250/320 GB | Lento: usar zram, evitar swap em disco, desligar serviços que escrevem muito |
| USB | 2 portas USB 2.0 | ~500 mA por porta — o celular vai drenar a bateria |
| Saída de vídeo | **HDMI** e **VGA (D-Sub)** — ambas só de saída | Permitem espelhar/estender a sessão numa TV ou monitor; **não** recebem sinal |
| Áudio | P2 (fone/microfone) + alto-falantes internos + microfone | Saída de fone útil se o áudio do scrcpy for ligado |
| Rede | Ethernet 10/100 + Wi-Fi 802.11 b/g/n (Broadcom na maioria dos lotes) | Wi-Fi exige `firmware-brcm80211` |
| Bluetooth | 3.0 em algumas unidades | **Não necessário** — toda a entrada vai por USB |
| Firmware | BIOS legado, sem UEFI | Boot por USB com **Esc** no POST; setup com **F2** |
| Câmera | 0,3 MP | Não usada |

### 4.2 Software (lado celular)

| Item | Versão mínima | Status |
|---|---|---|
| Android | 15 (One UI 7) para modo desktop genérico | Parcial |
| **One UI 8 / Android 16** | DeX novo em display virtual | **Alvo** |
| Depuração USB | Habilitada + chave RSA autorizada | Manual (1× por máquina) |
| Layout do teclado físico | Português (Brasil) no celular | Necessário **apenas** com `--keyboard=uhid` |

### 4.3 Software (imagem)

| Item | Versão | Origem |
|---|---|---|
| Debian trixie | 13.x | `debootstrap` (componentes `main`, `contrib`, `non-free-firmware`) |
| Kernel | `linux-image-amd64` (genérico) | Debian — não usar subkernel `686`/`atom` (descontinuado no Debian 12+) |
| Firmware | `firmware-amd-graphics` (GPU), `firmware-brcm80211` (Wi-Fi) | Debian `non-free-firmware` |
| scrcpy | 3.x (tar oficial) | GitHub Releases, com SHA-256 conferido |
| adb | pacote `adb` | Debian (já traz as regras udev) |
| Desktop | LXQt + SDDM | Debian |
| Driver de vídeo X11 | `xserver-xorg-video-radeon` + Mesa | Debian |
| GRUB | `grub-pc` (`i386-pc`) | Debian |

---

## 5. Arquitetura

```
┌─────────────────────── Eee PC 1015BX ──────────────────────┐
│  BIOS → GRUB (i386-pc) → kernel amd64 + initramfs          │
│    ↓                                                       │
│  systemd (multi-user)                                      │
│    ├─ zram swap (systemd-zram-generator)                   │
│    ├─ SDDM → autologin usuário `dex` → sessão LXQt         │
│    │     └─ autostart: ~/.config/autostart/dex-station.desktop
│    │           └─ dex-station-session (daemon do kiosk)    │
│    │                 ├─ consulta adb (2 s)                 │
│    │                 ├─ aplica settings globais do Android │
│    │                 ├─ método A: scrcpy --new-display     │
│    │                 │     └─ fallback B: overlay_display_devices
│    │                 │          └─ fallback C: espelho simples
│    │                 └─ unplug → limpeza → tela de espera  │
│    └─ Ctrl+Alt+F2 → TTY de manutenção                      │
│                                                            │
│  USB 2.0 ──── cabo ────► Samsung Galaxy (USB-C)            │
│              dados (adb) + carga (limitada a 500 mA)       │
└────────────────────────────────────────────────────────────┘
```

### Fluxo de boot

1. A BIOS carrega o GRUB; o kernel amd64 sobe com o systemd.
2. `systemd-zram-generator` cria swap comprimido na RAM (ajuda no 2 GB, ao custo de CPU).
3. SDDM faz autologin do usuário `dex` e abre a sessão LXQt.
4. O daemon `dex-station-session` sobe e mostra a **tela de espera**: papel de parede com
   instruções e um atalho de terminal.
5. Loop: a cada 2 s, `adb devices`. Se houver aparelho autorizado, inicia a sessão DeX.

### Estados do daemon

```
IDLE ──(celular autorizado)──► PREPARING ──(ok)──► RUNNING
  ▲                                 │
  │                           (falha no método A)
  │                                 ▼
  │                            FALLBACK_B ──(falha)──► FALLBACK_C
  │                                 │                      │
  └──────(celular desconectado)─────┴──────────────────────┘
        (limpeza: encerra scrcpy, apaga overlay_display_devices)
```

---

## 6. Configuração do lado Android (manual, 1×)

1. **Depuração USB**: Ajustes → Sobre o telefone → Número da build (7 toques) → Opções do
   desenvolvedor → *Depuração USB* ON.
2. **Autorizar**: na primeira conexão, aceitar a chave RSA no celular. A tela de espera
   deve instruir isso.
3. **Settings globais** (o daemon aplica; documentados aqui para recuperação manual):
   ```bash
   adb shell settings put global enable_freeform_support 1
   adb shell settings put global force_desktop_mode_on_external_displays 1
   adb shell settings put global enable_non_resizable_multi_window 1
   ```
   São inofensivos se desnecessários no seu build; servem de rede de segurança para os
   fallbacks B e C.
4. **Layout do teclado físico** (Ajustes → Sistema → Idiomas e entrada → Teclado físico →
   Português (Brasil)) — **só** necessário com `--keyboard=uhid`.
5. **Celular desbloqueado** durante o uso.
6. **Economia de energia**: o daemon usa `--turn-screen-off` (a tela do Galaxy apaga; o
   celular continua rodando o DeX).

---

## 7. Variante A — Live USB

**Objetivo:** testar o sistema inteiro sem instalar nada no HD do netbook.

- Artefato: a **mesma** `dex-station.img` (~2,5–3 GB), gravada em pendrive USB 2.0+ de
  **8 GB ou mais** (4 GB é apertado porque a imagem cresce com LXQt e firmware).
- Boot: ligar o netbook segurando **Esc** → escolher a unidade USB. Se ela não aparecer:
  **F2** na inicialização → aba *Boot* → *USB Boot* / prioridade de boot.
- O HD do netbook permanece **intocado** — este é o critério de aceite principal (T8).
- Persistência: **não há**, de propósito. Cada boot começa do zero. Se isso for necessário
  no futuro, o caminho é squashfs + overlayfs (fora do escopo da v1).

**Script**: `scripts/make-live-usb.sh <dispositivo> <img>` — recusa dispositivo montado,
exige confirmação explícita, confere o alvo com `lsblk`, grava com `dd`, roda `sync` e
compara o SHA-256 do pendrive com o da imagem.

---

## 8. Variante B — Instalação no HD

**Objetivo:** sistema dedicado permanente no netbook.

**Script**: `scripts/install-on-disk.sh <dispositivo> <img>`.

Sequência obrigatória:

1. **Verificação**: o alvo é um disco (não uma partição), não está montado, e a geometria
   confere com o `lsblk`.
2. **Backup** antes de apagar:
   - tabela de partições (`sfdisk --dump`) e cópia do MBR
     (`dd if=/dev/sdX of=mbr.bin bs=512 count=1`);
   - `tar` de `/home` e `/etc` do sistema atual;
   - manifesto dos pacotes instalados (`dpkg --get-selections`);
   - opcionalmente imagem completa (`dd`), se houver espaço — avisando que o HDD do 1015BX
     grava a ~20–25 MB/s (≈ 1,5 GB/min).
3. **Particionamento** (BIOS legado, MBR para máxima compatibilidade):
   - deixar **1 MiB livres no começo do disco** (área onde o GRUB grava o `core.img`);
   - p1: `boot`, 512 MiB, `ext4`, flag `boot`;
   - p2: `root`, restante do disco, `ext4`.
   - Alternativa: GPT com uma partição `bios_grub` de 1 MiB, caso o BIOS reclame do MBR.
4. `mkfs.ext4` com `-m 0` (sem espaço reservado a root) e ajustes de `tune2fs`.
5. Cópia do rootfs com `rsync -aHAX`; `fstab` por `UUID` com `noatime`.
6. `update-initramfs -u`, `grub-install --target=i386-pc /dev/sdX`, `grub-mkconfig -o
   /boot/grub/grub.cfg`.
7. Ordem de boot: HD primeiro; o pendrive continua bootável como plano B.

---

## 9. Componentes a desenvolver

### 9.1 `runtime/dex-station-session` (daemon do kiosk)

- **Linguagem**: Bash com `set -euo pipefail` (sem dependência de Python).
- **Entrada**: `/etc/dex-station/config` (KEY=VALUE) + flags de linha de comando.
- **Responsabilidades**:
  1. Consultar `adb devices` a cada 2 s, com `timeout` de 4 s por chamada e espera pela
     autorização RSA (mensagem na tela).
  2. Validar que o aparelho é Android ≥ 15 e registrar a versão.
  3. Aplicar os 3 settings globais.
  4. Resolver a resolução pela tela real (`xrandr` → 1024×600), com valor fixo de fallback.
  5. Lançar o scrcpy com o perfil de flags e vigiar a janela por N segundos.
  6. Cadeia de fallback A → B → C.
  7. Limpeza no unplug/desligamento: `settings delete global overlay_display_devices`,
     encerrar o scrcpy, restaurar a tela de espera.
  8. Log no journal (`systemd-cat`) + arquivo rotativo em `/var/log/dex-station/`.
- **Configuração** (`/etc/dex-station/config`):
  ```
  METHOD=auto            # auto | new-display | overlay | mirror
  RESOLUTION=auto        # auto | 1024x600
  DPI=160
  FPS=15
  BITRATE=4M
  VIDEO_CODEC=h264
  AUDIO=off              # on | off  (off por padrão: 2 GB de RAM)
  RENDER_DRIVER=software # software | opengl
  KEYBOARD=sdk           # sdk | uhid
  MOUSE=sdk              # sdk | uhid
  WINDOW_MODE=fullscreen # fullscreen | window
  BLACKOUT_TIMEOUT=15    # segundos sem imagem até considerar falha e cair no fallback
  ADB_TIMEOUT=4          # segundos por chamada ao adb
  POLL_INTERVAL=2        # segundos entre consultas
  ```
- **Ponto de falha conhecido**: com a tela do celular bloqueada, o adb conecta mas o DeX não
  sobe. O daemon deve detectar isso e mostrar a instrução "desbloqueie o celular" em vez de
  travar o loop.

### 9.2 `runtime/dex-station-launch`

- Wrapper fino com subcomandos: `start`, `stop`, `status`, `shell`, `logs`, `restart`.
- `status` mostra: aparelho conectado, resolução, método em uso, PID do scrcpy, CPU e RAM.

### 9.3 `runtime/dex-wait-screen`

- Tela de espera: papel de parede PNG gerado (instruções em PT-BR) + atalhos de desktop
  (`Terminal`, `Reiniciar sessão`).
- Atalho global `Alt+T` → terminal (via LXQt Global Shortcuts ou `xbindkeys`).

### 9.4 `build/build-dex-station.sh`

- Gera `dex-station.img` a partir de um host x86_64 Debian/Ubuntu (o próprio netbook serve,
  se tiver espaço).
- Etapas: `debootstrap` trixie → `chroot` → `post-install.sh` → `update-initramfs` →
  `grub-install --target=i386-pc` no arquivo de imagem (via *loop device*, com 1 MiB livre
  antes da primeira partição) → `sync` → fechamento da imagem.
- **Rastreabilidade**: `SHA256SUMS` do artefato + versões fixadas (scrcpy, adb, kernel) em
  `build/VERSIONS.lock`. O build é **repetível**, não bit-a-bit reproduzível (debootstrap e
  `apt` gravam timestamps e estados próprios).

### 9.5 `build/post-install.sh` (executa dentro do chroot)

- Pacotes: `linux-image-amd64 systemd-sysv lxqt sddm adb xdg-utils xbindkeys
  x11-xserver-utils xserver-xorg-video-radeon firmware-amd-graphics firmware-brcm80211
  zram-tools|systemd-zram-generator wget ca-certificates`.
- **scrcpy**: baixa o tarball oficial `scrcpy-linux-x86_64-v<VER>.tar.gz`, confere o SHA-256,
  instala em `/opt/scrcpy` e cria symlink em `/usr/local/bin`. Se o binário exigir bibliotecas
  além da glibc da trixie, usar o pacote `scrcpy` do Debian como alternativa.
- **Usuário** `dex` (uid 1000), grupos `audio video plugdev input netdev`.
- **SDDM**: `/etc/sddm.conf.d/10-autologin.conf` → `[Autologin] User=dex`,
  `Session=lxqt.desktop`, `Relogin=false`. (Alternativa mais leve, se a RAM apertar:
  `lightdm` ou `startx` via `getty@tty1`.)
- **Autostart**: `~dex/.config/autostart/dex-station.desktop` → `Exec=dex-station-session`.
- **Limpeza no desligamento**: `dex-station-cleanup.service` (`Type=oneshot`,
  `RemainAfterExit=yes`, `ExecStop=/usr/local/lib/dex-station/cleanup.sh`,
  `WantedBy=multi-user.target`) — `ExecStop` roda no `systemctl poweroff/reboot`.
- **Higiene**: desabilitar `cups`, `avahi-daemon`, `modemmanager`, `packagekit`, timers
  `apt-daily*`, `fstrim.timer` (HDD não suporta TRIM), automount do leitor de SD (busca
  lenta no cartão). Manter `systemd-timesyncd`.
  Obs.: `snapd` e `whoopsie` são pacotes de Ubuntu — não existem no Debian, não há o que
  desabilitar.
- **LXQt**: sem compositor (`~/.config/lxqt/lxqt.conf` → `[Compositor] enabled=false`), sem
  animações, fundo sóbrio.
- **zram**: `/etc/systemd/zram-generator.conf` → `zram-size=min(ram/2, 1G)`.
- **Locale**: `pt_BR.UTF-8` + `LANG=pt_BR.UTF-8`.
- **Hostname**: `dex-station`.

### 9.6 `build/overlay/`

- `fstab` (UUID + `noatime` + `errors=remount-ro`).
- `99-dex-station.rules` (udev: vendor Samsung `04e8`; permissões sem root — otimização v2).
- `10-autologin.conf` (SDDM).
- `journald.conf` (`Storage=persistent` com `SystemMaxUse=32M` — em HDD, 32 MB de log é
  barato e salva o diagnóstico depois de um travamento).
- `sysctl`/`modules`: `radeon` (GPU) e `snd-hda-intel` (áudio). **Não** usar `i915` (Intel).

### 9.7 `scripts/`

| Script | Função |
|---|---|
| `make-live-usb.sh` | grava a imagem num pendrive com verificações e conferência de SHA-256 |
| `install-on-disk.sh` | backup + particionamento + cópia + GRUB no HD |
| `backup-disk.sh` | backup parcial ou total do estado atual do netbook |
| `restore-disk.sh` | restaura o backup |
| `flash-netbook.sh` | orquestrador: detecta o dispositivo e pergunta "USB ou HD?" |

### 9.8 `docs/`

| Documento | Conteúdo |
|---|---|
| `OPERATOR.md` | Manual do usuário: atalhos, tela preta, como sair do kiosk |
| `BUILD.md` | Como gerar a imagem em qualquer host x86_64, versões fixadas, SHA-256 |
| `TROUBLESHOOTING.md` | Sintoma → causa → solução |

### 9.9 `tests/`

| Script | Testa |
|---|---|
| `test-config.sh` | parsing do config, defaults, valores inválidos |
| `test-state-machine.sh` | plug/unplug com `adb` falso no PATH |
| `test-fallback.sh` | força falha do método A e verifica a queda para B/C |
| `smoke-live-usb.sh` | checklist automatizado pós-boot |

---

## 10. Calibração de performance (perfil 1015BX / C-60)

| Flag | Valor | Motivo |
|---|---|---|
| `--new-display` | `1024x600/160` | 1:1 com o painel; sem escalonamento |
| `--video-codec` | `h264` | menor custo de decodificação em CPU (a UVD3 da APU não é usada pelo scrcpy) |
| `--video-bit-rate` | `4M` | UI de desktop é majoritariamente estática |
| `--max-fps` | `15` | o DeX é interface, não jogo |
| `--render-driver` | `software` | evita instabilidade de GL no Radeon HD 6290; testar `opengl` como otimização |
| `--turn-screen-off` | ativo | economiza bateria do celular |
| `--keep-active` | ativo | impede o celular de dormir |
| `--no-audio` | padrão | opus é barato, mas 2 GB de RAM pede economia; se ligado, usar o P2 (fone) |
| `--mouse` / `--keyboard` | `sdk` | modo absoluto funciona melhor com trackpad |
| Compositor | desligado | a APU não compensa a camada extra |

**Presets**: `PROFILE=1015bx` (padrão) e `PROFILE=modern` (1080p/60 fps/16M) para testar em
hardware melhor.

---

## 11. Matriz de testes

| ID | Teste | Critério de aceite |
|---|---|---|
| T0 | Build repetível | `sha256sum` registrado; dois builds seguidos sem erro |
| T1 | Chroot coerente | nenhum serviço falhando; `systemd-analyze` sem caminho crítico |
| T2 | Boot completo | autologin → LXQt → tela de espera em < 90 s (HDD lento) |
| T3 | Detecção | celular plugado é detectado; mensagem de autorização aparece |
| T4 | DeX virtual | DeX aparece no display virtual (barra de tarefas + janela redimensionável) |
| T5 | Entrada | clique, arrasto e rolagem funcionam; digitação ABNT2 correta (acentos, ç) |
| T6 | Hotplug | unplug → limpeza (sem display virtual órfão); replug → sessão nova |
| T7 | Fallback | método A forçado a falhar → B → C, cada um exibindo imagem |
| T8 | Live USB | boot pelo pendrive; **HD intocado** (tabela e checksums inalterados) |
| T9 | Rede | dados trafegam pelo USB; internet via tethering opcional |
| T10 | Instalação HD | backup gerado; imagem gravada; GRUB instalado; boot pelo HD |
| T11 | Performance | CPU < 60% e RAM < 400 MB em repouso por 30 min; sem travamento |
| T12 | Energia | celular carregando via hub alimentado; temperatura do netbook estável |
| T13 | Recuperação | `Ctrl+Alt+F2` → TTY; `dex-station restart`; `emergency.target` |
| T14 | Autonomia | sem celular: não trava, não reinicia, tela de espera persistente |

---

## 12. Riscos e mitigações

| Risco | Prob. | Impacto | Mitigação |
|---|---|---|---|
| DeX novo não aparece em display virtual em algum One UI 8.x | Média | Médio | Cadeia de fallback; testar no aparelho real cedo (M2) |
| Porta USB (500 mA) drena a bateria do netbook | Alta | Médio | Hub alimentado; `--turn-screen-off`; documentar no README |
| Radeon HD 6290 engasga na decodificação por CPU | Média | Médio | 1024×600/15 fps; se falhar, 800×480 |
| 2 GB de RAM (≈1,7 GB úteis) estoura com LXQt + scrcpy | Média | Alto | zram 1 GB; sem compositor; áudio off; `swap` só em zram |
| Trackpad com movimento "absoluto" estranho | Média | Baixo | `--mouse=sdk` padrão; testar `uhid` |
| Layout de teclado divergente (ABNT2 × US intl) | Alta | Baixo | Configurar teclado físico no celular; alternativa `--keyboard=sdk` |
| Sem internet no netbook (firmware Wi-Fi ausente) | Média | Médio | Incluir `firmware-brcm80211`; validar na instalação; tethering USB como plano B |
| Disco/tabela de partição antiga atrapalha o boot | Média | Alto | `install-on-disk.sh` zera o disco; backup obrigatório |
| BIOS não oferece boot por USB | Baixa | Alto | Habilitar *USB Boot* no F2; alternativa: gravar o HD por outro PC com adaptador USB-SATA |
| HDD 5400 rpm lento | Alta | Médio | zram em vez de swap; desligar automount SD; `fstrim` off |
| Build oficial do scrcpy incompatível com glibc da trixie | Baixa | Médio | Pacote `scrcpy` do Debian ou compilar do fonte |
| Superaquecimento / throttle do C-60 (9 W, cooler de netbook) | Média | Baixo | Limpar o cooler; não bloquear as saídas de ar; monitorar temperatura em T12 |

---

## 13. Roadmap

| Fase | Entregável | Depende de |
|---|---|---|
| **M0** | Este plano + esqueleto do repositório | — |
| **M1** | `dex-station-session` + `dex-station-launch` funcionando **na distro atual** do netbook | M0 |
| **M2** | Validação no aparelho real: DeX virtual + entrada (T3–T7) | M1 + celular |
| **M3** | `build-dex-station.sh` + `post-install.sh` + overlay → `dex-station.img` | M2 |
| **M4** | `make-live-usb.sh` → boot pelo pendrive (T8, T9, T14) | M3 |
| **M5** | `install-on-disk.sh` + backup/restore → instalação no HD (T10) | M4 |
| **M6** | Calibração fina (T11, T12), matriz de fallback, documentação | M5 |
| **M7** | Estabilização: testes automatizados, changelog, release `v1.0` | M6 |

**Ordem deliberada:** o núcleo do problema (DeX + entrada + hotplug) é validado em M1–M2 na
distro que já está no netbook, em minutos. A construção da imagem (M3) é mecânica e só entra
depois que a parte difícil está provada.

---

## 14. Estrutura do repositório

```
dex-station/
├── PLAN.md                     # este documento
├── README.md                   # visão geral
├── build/
│   ├── build-dex-station.sh    # debootstrap → imagem
│   ├── post-install.sh         # configuração dentro do chroot
│   ├── VERSIONS.lock           # versões fixadas + SHA-256
│   └── overlay/                # arquivos injetados na raiz da imagem
├── runtime/
│   ├── dex-station-session     # daemon do kiosk
│   ├── dex-station-launch      # CLI de controle
│   └── dex-wait-screen/        # tela de espera
├── scripts/
│   ├── make-live-usb.sh
│   ├── install-on-disk.sh
│   ├── backup-disk.sh
│   ├── restore-disk.sh
│   └── flash-netbook.sh
├── docs/
│   ├── OPERATOR.md
│   ├── BUILD.md
│   └── TROUBLESHOOTING.md
└── tests/
    ├── test-config.sh
    ├── test-state-machine.sh
    ├── test-fallback.sh
    └── smoke-live-usb.sh
```

---

## 15. Referências

- scrcpy — releases e display virtual: <https://github.com/Genymobile/scrcpy/releases> e
  `doc/virtual-display.md`
- DeX no One UI 8 via display virtual (Android Authority, 07/2025):
  <https://www.androidauthority.com/samsung-dex-on-pc-one-ui-8-3582695>
- Samsung — teclado e mouse no DeX: <https://www.samsung.com/us/support/answer/ANS10003477/>
- Samsung DeX — como funciona: <https://developer.samsung.com/samsung-dex/how-it-works.html>
- Script de referência One UI 8 (`overlay_display_devices` + 3 settings globais):
  <https://github.com/fqazzazee/dex-manager-linux>
- Cliente DeX para Linux via scrcpy: <https://github.com/gabdevele/DexOnLinux>
- Emulação de HID Bluetooth no Linux (plano B de entrada): `btkbdd`, `btvirt`,
  <https://github.com/tonytallman/easyhid>
- Especificações do AMD C-60: <https://www.techpowerup.com/cpu-specs/c-60.c796>
- Especificações do Eee PC 1015BX:
  <https://www.notebookcheck.net/Asus-Eee-PC-1015BX-hits-Germany-carries-C-60.62087.0.html>
  e manual oficial (portas, atalhos de boot):
  <https://www.asus.com/us/supportonly/eee%20pc%201015bx/helpdesk_manual/>

---

## 16. Decisões em aberto

1. **Persistência no pendrive** — v1 sem persistência. Persistência real exigiria
   squashfs + overlayfs. Confirmar se é requisito.
2. **Áudio** — padrão desligado na v1 (2 GB de RAM). O netbook tem P2 de fone, então há
   como ouvir sem depender dos alto-falantes fracos. Reavaliar depois do T11.
3. **Idiomas** — sistema em pt_BR (padrão) ou en_US com mensagens em português?
4. **Nível de "dedicação"** — remover o apt (imagem imutável) ou manter para manutenção?
   Recomendo manter, com apt disponível no TTY de manutenção.
5. **Segundo uso** — apenas DeX, ou também um navegador local no netbook? Isso muda a
   decisão de manter o LXQt completo.
6. **Gerenciador de sessão** — SDDM (padrão, ~60–80 MB) ou algo mais leve
   (`lightdm`/`startx`) para preservar RAM no 2 GB?
