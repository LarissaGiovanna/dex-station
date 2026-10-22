# DeX Station OS — Plano de Desenvolvimento

**Alvo:** Asus Eee PC 1000 / 1000H (Intel Atom N450/N455 1,5 GHz, GMA 450, 2 GB RAM, 1024×600, BIOS legado)
**Fonte:** Samsung Galaxy com One UI 8 / Android 16
**Objetivo:** transformar o netbook numa estação DeX dedicada — o celular faz o processamento, o netbook é monitor + teclado + trackpad.

---

## 1. Sumário executivo

Não existe distribuição Linux publicada que faça "netbook como estação DeX". O que será desenvolvido é uma **imagem de sistema sob medida** (Debian 13 minimal + LXQt + scrcpy ≥ 3.0 + adb + daemon de kiosk) que:

- inicia sozinha, sem login;
- fica em uma tela de espera detectando o celular via USB;
- quando o Galaxy é plugado, abre o **DeX real** em tela cheia;
- encaminha teclado e trackpad do netbook para o celular;
- ao desconectar o cabo, limpa a sessão e volta à tela de espera.

Duas variantes a partir da **mesma imagem**:

| Variante | Artefato | Uso |
|---|---|---|
| **A — Live USB** | `dex-station.img` gravado em pendrive | Testar sem tocar no HD do netbook |
| **B — Instalação no HD** | `dex-station.img` gravado no disco do netbook | Sistema dedicado permanente |

A diferença entre as variantes é **o alvo do `dd`**, não o conteúdo da imagem. Detalhes em §7 e §8.

---

## 2. Decisões de projeto

| # | Decisão | Justificativa |
|---|---|---|
| D1 | Conexão **USB** (adb/scrcpy), não HDMI | Um único cabo para vídeo, entrada e energia; dispensa placa de captura, dongle BT e gambiarra física |
| D2 | Base **Debian 13 (trixie) minimal** + **LXQt** | Baixo consumo de RAM/CPU; LXQt é leve em X11 e suporta autologin bem |
| D3 | **scrcpy ≥ 3.0** (binário estático oficial) | Único requisito para `--new-display`; build estático evita SDL2/FFmpeg no netbook antigo |
| D4 | Display virtual **1024×600/160** | 1:1 com o painel → sem escalonamento → menos CPU de rasterização |
| D5 | Sessão em **kiosk** (daemon + autologin), sem gerenciador de janelas aparente | Usuário não interage com o SO, só com o DeX |
| D6 | Mesma imagem para USB e HD | Menos superfície de manutenção; a diferença é o alvo do `dd` |
| D7 | Detecção por **poll de `adb` a cada 2 s** (v1) | Robusto a qualquer estado do udev; custo desprezível. udev como otimização futura (v2) |
| D8 | Limpeza por **cadeia de fallback** em 3 níveis | O netbook nunca fica sem imagem, mesmo se o DeX virtual falhar |

---

## 3. Por que o USB substitui o HDMI

| | Caminho HDMI (tradicional) | Caminho USB (adotado) |
|---|---|---|
| Vídeo | DP Alt Mode (USB-C) → HDMI → placa de captura → `/dev/video0` | Codec H.264/H.265 pelo adb |
| Entrada | HID via dongle USB ou Bluetooth (emulação de HID BT no Linux) | Eventos de teclado/mouse injetados pelo adb |
| Cabos | 2–3 (HDMI, USB de dados, carga) | 1 (dados + carga) |
| Hardware extra | Placa de captura (~R$60–120) | Nenhum |
| Latência | 2 saltos de encode | 1 salto |
| Bloqueio do HDMI | Netbook só tem **saída** HDMI; não existe distro que "detecte entrada HDMI" sem placa de captura | — |

**O que se perde:** o handshake do modo TV / Smart View. **O que se ganha:** o **DeX de verdade** (taskbar, janelas redimensionáveis), porque no One UI 8 a Samsung reconstruiu o DeX sobre o *Desktop Mode* nativo do Android 16, que roda em display virtual.

---

## 4. Requisitos e matriz de compatibilidade

### 4.1 Hardware

| Item | Especificação | Observação |
|---|---|---|
| Netbook | x86_64, BIOS legado, boot por USB | Eee PC 1000: F12 no POST para o boot menu |
| RAM | ≥ 1 GB (recomendado 2 GB) | 2 GB: LXQt ~200 MB + scrcpy ~80 MB |
| Disco | ≥ 2 GB para a imagem | HDD 5400 rpm do Eee 1000: usar zram, não swap em disco |
| USB | Porta USB 2.0 para o celular | ~500 mA: o celular vai drenar a bateria do netbook |
| Bluetooth | **Não necessário** | Todo o stack de entrada vai por USB |
| Câmera | Não necessária | — |

### 4.2 Software (lado celular)

| Item | Versão mínima | Status |
|---|---|---|
| Android | 15 (One UI 7) para modo desktop genérico | Parcial |
| **One UI 8 / Android 16** | DeX novo em display virtual | **Alvo** |
| Depuração USB | Habilitada + RSA autorizado | Manual (1x por máquina) |
| Layout teclado físico | Português (Brasil) no celular | Necessário se usar `--keyboard=uhid` |

### 4.3 Software (imagem)

| Item | Versão | Origem |
|---|---|---|
| Debian trixie | 13.x | `debootstrap` |
| Kernel | `linux-image-amd64` (genérico) | Debian — evitar subkernel `atom`, removido no Debian 12+ |
| scrcpy | 3.3.4 (estático, oficial) | GitHub Releases, com SHA-256 pinado |
| adb | `android-tools-adb` | Debian (inclui regras udev) |
| LXQt + SDDM | trixie | Debian |
| GRUB | `grub-pc` (BIOS, i386-pc) | Debian |

---

## 5. Arquitetura

```
┌─────────────────────── Eee PC 1000 ───────────────────────┐
│  BIOS → GRUB → kernel + initramfs                          │
│    ↓                                                       │
│  systemd (multi-user)                                      │
│    ├─ zram swap (systemd-zram-generator)                   │
│    ├─ SDDM → autologin usuário `dex` → sessão LXQt         │
│    │     └─ autostart: ~/.config/autostart/dex-station.desktop │
│    │           └─ dex-station-session (daemon do kiosk)     │
│    │                 ├─ poll adb (2 s)                     │
│    │                 ├─ aplica settings globais do Android │
│    │                 ├─ método A: scrcpy --new-display     │
│    │                 │     └─ fallback B: overlay_display_devices │
│    │                 │          └─ fallback C: espelho simples│
│    │                 └─ unplug → cleanup → tela de espera  │
│    └─ ctrl-alt-f2 → TTY de manutenção                       │
│                                                             │
│  USB-A ──── cabo ────► Samsung Galaxy (USB-C)              │
│            dados (adb) + carga                             │
└─────────────────────────────────────────────────────────────┘
```

### Fluxo de boot

1. GRUB inicia o kernel; systemd sobe o sistema mínimo.
2. `systemd-zram-generator` cria swap em RAM (opcional, mas ajuda o 2 GB).
3. SDDM entra em autologin do usuário `dex` → sessão LXQt.
4. O daemon `dex-station-session` sobe e mostra a **tela de espera** (papel de parede com instruções + ícone de terminal).
5. Loop: a cada 2 s, `adb devices` → se houver aparelho autorizado, iniciar sessão DeX.

### Estados do daemon

```
IDLE ──(celular autorizado)──► PREPARING ──(ok)──► RUNNING
  ▲                                │
  │                          (falha no método A)
  │                                ▼
  │                            FALLBACK_B ──(falha)──► FALLBACK_C
  │                                │                        │
  └──────(celular desconectado)───┴────────────────────────┘
              (cleanup: encerra scrcpy, apaga overlay_display_devices)
```

---

## 6. Configuração do lado Android (procedimento manual, 1x)

1. **Depuração USB**: Ajustes → Sobre o telefone → Número da build (7 toques) → Opções do desenvolvedor → *Depuração USB* ON.
2. **Autorizar**: na primeira conexão, aceitar a impressão digital RSA no celular. (A tela de espera deve instruir isso.)
3. **Settings globais** (aplicados pelo daemon, mas documentados para recuperação manual):
   ```bash
   adb shell settings put global enable_freeform_support 1
   adb shell settings put global force_desktop_mode_on_external_displays 1
   adb shell settings put global enable_non_resizable_multi_window 1
   ```
4. **Layout de teclado físico** (Ajustes → Sistema → Idiomas e entrada → Teclado físico → Português (Brasil)) — necessário para `--keyboard=uhid`.
5. **Celular desbloqueado** durante o uso.
6. **Economia de energia**: o daemon usa `--turn-screen-off` no celular (a tela do Galaxy apaga, ele continua carregando).

---

## 7. Variante A — Live USB

**Objetivo:** testar o sistema inteiro sem instalar nada no HD do netbook.

- Artefato: a **mesma** `dex-station.img` (~1,5 GB), gravada em pendrive USB 2.0+ (≥ 4 GB).
- Boot: F12 no POST do Eee PC 1000 → selecionar a unidade USB. Se não aparecer: BIOS → *Boot* → *USB Boot* Enabled.
- O HD do netbook permanece **intocado** — é o teste de aceite principal.
- Persistência: **não há** de propósito. O pendrive é descartável; cada boot começa limpo. Refinamento opcional (fora do escopo v1): flag `--reset-on-boot` que apaga `/var/lib/dex-station` e o cache do adb a cada boot.
- Adequação alternativa: por ser persistente, a mesma imagem no HD guarda logs e ajustes; instalar no pendrive o que se chama de "persistência" é fora do escopo v1.

**Script**: `scripts/make-live-usb.sh <device> <img>` — recusa dispositivos montados, exige confirmação explícita, verifica o alvo com `lsblk` antes de gravar.

## 8. Variante B — Instalação no HD

**Objetivo:** sistema dedicado permanente no netbook.

**Script**: `scripts/install-on-disk.sh <device> <img>`.

Sequência obrigatória:

1. **Verificação**: target é disco (não partição), não está montado, tabela de partição confere.
2. **Backup** antes de apagar:
   - `sfdisk --dump` (tabela de partições) e cópia do MBR (`dd if=/dev/sdX of=mbr.bin bs=512 count=1`);
   - `tar` de `/home` e `/etc` do sistema atual;
   - manifesto dos pacotes instalados (`dpkg --get-selections`);
   - opcionalmente imagem completa (`dd`) se houver espaço — avisar que o HD é lento (≈ 1,5 GB/min no Eee 1000).
3. **Particionamento** (BIOS legado, MBR para máxima compatibilidade):
   - p1: 1 MiB, tipo `bios_grub` não é necessário em MBR; layout MBR simples:
     - p1 `boot` 512 MiB, `ext4`, flag `boot`;
     - p2 `root` resto do disco, `ext4`.
   - Alternativa GPT + `bios_grub` de 1 MiB, se o BIOS reclamar do MBR.
4. `mkfs.ext4` (com `-m 0` — sem espaço reservado), `tune2fs` ajustes.
5. Cópia do rootfs via `rsync -aHAX` + `fstab` com `UUID` + `noatime`.
6. `update-initramfs`, `grub-install --target=i386-pc /dev/sdX`, `grub-mkconfig`.
7. Ordem de boot: HD em primeiro (o USB continua bootável como plano B).

---

## 9. Componentes a desenvolver

### 9.1 `runtime/dex-station-session` (daemon do kiosk)

- **Linguagem**: Bash + `set -euo pipefail` (portabilidade; sem dependência de Python).
- **Entrada**: `/etc/dex-station/config` (KEY=VALUE) + flags de linha de comando.
- **Responsabilidades**:
  1. Poll `adb devices` (2 s) com timeout de espera por autorização.
  2. Validar que o aparelho é Android ≥ 15 e reportar a versão.
  3. Aplicar os 3 settings globais.
  4. Resolver a resolução virtual a partir da tela real (`xrandr` → 1024×600), com fallback fixo.
  5. Lançar o scrcpy com o perfil de flags; vigiar a janela (blackout) por N segundos.
  6. Cadeia de fallback A → B → C.
  7. Cleanup no unplug/desligamento: `settings delete global overlay_display_devices`, matar scrcpy, restaurar tela de espera.
  8. Log no journal (`systemd-cat`) + arquivo rotativo em `/var/log/dex-station/`.
- **Configuração** (`/etc/dex-station/config`):
  ```
  METHOD=auto            # auto | new-display | overlay | mirror
  RESOLUTION=auto        # auto | 1024x600
  DPI=160
  FPS=15
  BITRATE=4M
  VIDEO_CODEC=h264
  AUDIO=on               # on | off
  RENDER_DRIVER=software # software | opengl
  KEYBOARD=sdk           # sdk | uhid
  MOUSE=sdk              # sdk | uhid
  WINDOW_MODE=fullscreen # fullscreen | window
  BLACKOUT_TIMEOUT=15    # segundos até considerar falha e cair no fallback
  ```
- **Ponto de falha crítico**: `adb` server tentando conectar enquanto a tela do celular está bloqueada → timeout de 5 s por tentativa, sem travar o loop.

### 9.2 `runtime/dex-station-launch`

- Wrapper fino com subcomandos: `start`, `stop`, `status`, `shell`, `logs`, `restart`.
- `status` mostra: aparelho conectado, resolução, método em uso, PID do scrcpy, CPU/RAM.

### 9.3 `runtime/dex-wait-screen`

- Tela de espera: papel de parede PNG gerado (instruções em PT-BR) + um ou dois atalhos de desktop (`Terminal`, `Reiniciar sessão`).
- Ícone/atalho global `Alt+T` → terminal (via LXQt Global Shortcuts ou `xbindkeys`).

### 9.4 `build/build-dex-station.sh`

- Gera `dex-station.img` a partir de um host x86_64 Debian/Ubuntu (o próprio netbook serve).
- Etapas: `debootstrap` trixie → `chroot` → `post-install.sh` → `mkswap`/zram → `update-initramfs` → `grub-install` na imagem (com `grub-mkrescue`? não: instalar GRUB diretamente no arquivo de imagem via loop device) → `dd` da imagem final.
- **Determinismo**: `SHA256SUM` do artefato gerado + registro das versões fixadas (scrcpy, adb, kernel) em `build/VERSIONS.lock`.

### 9.5 `build/post-install.sh` (executa dentro do chroot)

- Pacotes: `linux-image-amd64 systemd-sysv lxqt sddm adb (ou android-tools-adb) xdg-utils xbindkeys x11-xserver-utils zram-generator (ou systemd-zram-generator) wget ca-certificates`.
- **scrcpy**: baixa o tarball estático `scrcpy-linux-x86_64-v<VER>.tar.gz`, confere SHA-256, instala em `/opt/scrcpy`, cria symlink em `/usr/local/bin`.
- **Usuário** `dex` (uid 1000), grupos `audio video plugdev input netdev`.
- **SDDM**: `/etc/sddm.conf.d/10-autologin.conf` → `[Autologin] User=dex, Session=lxqt.desktop, Relogin=false`.
- **Autostart**: `~dex/.config/autostart/dex-station.desktop` → `Exec=dex-station-session`.
- **systemd**: `dex-station.target`? Não necessário; mas criar `dex-station-cleanup.service` (oneshot, `ExecStopPost`) para garantir limpeza no desligamento.
- **Higiene**: desabilitar `snapd`, `cups`, `avahi-daemon`, `modemmanager`, `packagekit`, `whoopsie`, `e2scrub_all`; `apt-daily*` timers off; `fstrim.timer` off (SSD não é o caso, mas HDD); desabilitar automount do leitor SD (seek lento); `systemd-timesyncd` mantido.
- **LXQt**: sem compositor (`~/.config/lxqt/lxqt.conf` → `[Compositor] enabled=false`), sem animações, fundo sóbrio.
- **zram**: `/etc/systemd/zram-generator.conf` → `zram-size=min(ram/2, 1G)`.
- **Locale**: `pt_BR.UTF-8` + `LANG=pt_BR.UTF-8` (mensagens do sistema em português).
- **Hostname**: `dex-station`.

### 9.6 `build/overlay/`

- `fstab` (UUID + noatime + `errors=remount-ro`).
- `99-dex-station.rules` (udev: vendor Samsung `04e8`; permissões de acesso sem root — otimização v2, dispensável na v1).
- `10-autologin.conf` (SDDM).
- `journald.conf` (persistência mínima, `Storage=volatile`).
- `sysctl`/`modules` (`snd-hda-intel`, `i915`?).

### 9.7 `scripts/`

| Script | Função |
|---|---|
| `make-live-usb.sh` | grava a imagem num pendrive com verificações |
| `install-on-disk.sh` | backup + particionamento + cópia + GRUB no HD |
| `backup-disk.sh` | backup do estado atual do netbook (parcial ou total) |
| `restore-disk.sh` | restaura o backup |
| `flash-netbook.sh` | orquestrador: detecta o dispositivo certo e pergunta: USB ou HD? |

### 9.8 `docs/`

| Documento | Conteúdo |
|---|---|
| `OPERATOR.md` | Manual do usuário: como usar, atalhos, o que fazer se a tela ficar preta, como sair do modo kiosk |
| `BUILD.md` | Como gerar as imagens em qualquer host x86_64, versões pinadas, verificação de SHA-256 |
| `TROUBLESHOOTING.md` | Matriz de sintomas → causa → solução |

### 9.9 `tests/`

| Script | Testa |
|---|---|
| `test-config.sh` | parsing do config, defaults, valores inválidos |
| `test-state-machine.sh` | simulação de plug/unplug com adb mockado (fake `adb` no PATH) |
| `test-fallback.sh` | força falha do método A e verifica a queda para B/C |
| `smoke-live-usb.sh` | checklist automatizado pós-boot |

---

## 10. Calibração de performance (perfil Eee PC 1000H)

| Flag | Valor | Motivo |
|---|---|---|
| `--new-display` | `1024x600/160` | 1:1 com o painel; sem escalonamento |
| `--video-codec` | `h264` | menor custo de decodificação (GMA 450 não decodifica H.264 em hardware) |
| `--video-bit-rate` | `4M` | UI de desktop é majoritariamente estática |
| `--max-fps` | `15` | o DeX é UI, não jogo |
| `--render-driver` | `software` | evita GL instável/llvmpipe na GMA 450 |
| `--turn-screen-off` | ativo | economiza bateria do celular |
| `--keep-active` | ativo | impede o celular de dormir |
| `--no-audio` | opcional | opus é barato, mas dá para desligar |
| `--mouse` / `--keyboard` | `sdk` | modo absoluto funciona melhor com trackpad (sem cursor preso no canto) |
| Compositor | desligado | GMA 450 não compensa |

**Presets**: `PROFILE=eee1000` (padrão), `PROFILE=modern` (1080p/60fps/16M) para testar em hardware melhor.

---

## 11. Matriz de testes

| ID | Teste | Critério de aceite |
|---|---|---|
| T0 | Build reproduzível | `sha256sum` do artefato registrado; build 2x sem erro |
| T1 | Chroot coerente | sem serviço falhando; `systemd-analyze` sem critical path |
| T2 | Boot completo | autologin → sessão LXQt → tela de espera em < 60 s |
| T3 | Detecção | celular plugado é detectado; mensagem de "aceite a autorização" aparece |
| T4 | DeX virtual | DeX aparece no display virtual (taskbar + janela redimensionável) |
| T5 | Entrada | clique, arrasto, rolagem do trackpad funcionam; digitação ABNT2 correta (acentos, ç) |
| T6 | Hotplug | unplug → cleanup (sem display virtual órfão); replug → sessão nova |
| T7 | Fallback | método A forçado a falhar → B → C, cada um exibindo imagem |
| T8 | Live USB | boot pelo pendrive; **HD intocado** (tabela de partições e checksum inalterados) |
| T9 | Live USB → rede | celular e netbook trocam dados via USB; internet via tethering opcional |
| T10 | Instalação HD | backup gerado; imagem gravada; GRUB instalado; boot pelo HD |
| T11 | Performance | uso de CPU < 60% ocioso durante 30 min; RAM < 400 MB; sem travamento |
| T12 | Energia | celular carregando pelo USB com hub alimentado; temperatura do netbook estável |
| T13 | Recuperação | `Ctrl+Alt+F2` → TTY; `dex-station restart`; `emergency.target` |
| T14 | Autonomia | sem celular: sistema não trava, não reinicia, tela de espera persistente |

---

## 12. Riscos e mitigações

| Risco | Prob. | Impacto | Mitigação |
|---|---|---|---|
| DeX novo não aparece em display virtual em algum modelo/One UI 8.x | Média | Médio | Cadeia de fallback; testes com o aparelho real cedo (Fase M2) |
| Consumo da porta USB (500 mA) drena o netbook | Alta | Médio | Documentar hub USB alimentado; `--turn-screen-off` |
| GMA 450 engasga na decodificação | Média | Médio | Preset 1024×600/15fps; se ainda falhar, reduzir para 800×480 |
| Rolo/touchpad com movimento "absoluto" estranho | Média | Baixo | `--mouse=sdk` como padrão; testar `uhid` como alternativa |
| Layout de teclado divergente (ABNT2 vs. US intl) | Alta | Baixo | Configurar teclado físico no celular; alternativa `--keyboard=sdk` |
| Sem internet no celular | Média | Alto | Tethering do celular (o próprio adb já passa pelo USB) |
| Partição/boot em disco com tabela antiga | Média | Alto | `install-on-disk.sh` zera o disco; backup obrigatório antes |
| BIOS não permite boot USB | Baixa | Alto | Habilitar *USB Boot* na setup; fallback: instalar via USB mesmo assim |
| Ralentização do HDD (5400 rpm) | Alta | Médio | zram em vez de swap; desabilitar automount SD; trim off |
| scrcpy estático "experimental" no Linux | Baixa | Médio | Fallback: pacote Debian trixie-backports (3.3.4) ou compilar do fonte |

---

## 13. Roadmap

| Fase | Entregável | Depende de |
|---|---|---|
| **M0** | Este plano + esqueleto do repositório | — |
| **M1** | `runtime/dex-station-session` + `dex-station-launch` funcionando **na distro atual** do netbook (loop de feedback rápido, sem build de imagem) | M0 |
| **M2** | Validação no aparelho real: DeX virtual + entrada (T3–T7) | M1 + celular |
| **M3** | `build/build-dex-station.sh` + `post-install.sh` + overlay → `dex-station.img` | M2 |
| **M4** | `scripts/make-live-usb.sh` → boot pelo pendrive (T8, T9, T14) | M3 |
| **M5** | `scripts/install-on-disk.sh` + backup/restore → instalação no HD (T10) | M4 |
| **M6** | Calibração fina de performance (T11, T12), matriz de fallback, documentação | M5 |
| **M7** | Estabilização: testes automatizados, changelog, release `v1.0` | M6 |

**Ordem deliberada:** o núcleo do problema (DeX + entrada + hotplug) é validado em M1–M2 na distro que o usuário já tem, em minutos. A construção da imagem (M3) é mecânica e só aparece depois que a parte difícil está provada.

---

## 14. Estrutura do repositório

```
dex-station/
├── PLAN.md                     # este documento
├── README.md                   # quickstart
├── build/
│   ├── build-dex-station.sh    # debootstrap → imagem
│   ├── post-install.sh         # configuração dentro do chroot
│   ├── VERSIONS.lock           # versões pinadas + SHA-256
│   └── overlay/                # arquivos injetados na raiz da imagem
├── runtime/
│   ├── dex-station-session     # daemon do kiosk
│   ├── dex-station-launch      # CLI de controle
│   └── dex-wait-screen/        # tela de espera
├── scripts/
│   ├── make-live-usb.sh
│   ├── install-on-disk.sh
│   ├── backup-disk.sh
│   └── restore-disk.sh
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

- scrcpy — releases e documentação de display virtual: <https://github.com/Genymobile/scrcpy/releases> e `doc/virtual-display.md`
- DeX no One UI 8 em PC via display virtual (Android Authority, 07/2025): <https://www.androidauthority.com/samsung-dex-on-pc-one-ui-8-3582695>
- Samsung — uso de teclado e mouse no DeX (Bluetooth/USB suportados): <https://www.samsung.com/us/support/answer/ANS10003477/>
- Samsung DeX — como funciona (dock/HDMI/wireless): <https://developer.samsung.com/samsung-dex/how-it-works.html>
- Script de referência para One UI 8 (`overlay_display_devices` + 3 settings globais): <https://github.com/fqazzazee/dex-manager-linux>
- Cliente DeX para Linux via scrcpy (abordagem clássica): <https://github.com/gabdevele/DexOnLinux>
- Emulação de HID Bluetooth no Linux (referência para o plano B de entrada): `btkbdd`, `btvirt`, <https://github.com/tonytallman/easyhid>

---

## 16. Decisões em aberto

1. **Persistência no pendrive** — v1 sem persistência (pendrive descartável). Persistência real exigiria squashfs+overlayfs (mais complexo). Confirmar se é requisito.
2. **Áudio** — ligado por padrão ou desligado no preset fraco? (Eee 1000 tem alto-falantes fracos.)
3. **Idiomas** — sistema em pt_BR (padrão) ou en_US com mensagens em português?
4. **Nível de "dedicação"** — remover completamente o apt (imagem imutável) ou manter para manutenção? Recomendo manter, com `apt` disponível no terminal de manutenção.
5. **Dois perfis de uso** — apenas DeX, ou também um segundo uso (navegador local no netbook)? Altera a decisão sobre manter o LXQt completo.
