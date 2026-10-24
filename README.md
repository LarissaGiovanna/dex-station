# DeX Station OS

Transforma um netbook antigo — o **Asus Eee PC 1015BX** — numa **estação DeX dedicada**.
O celular Samsung faz todo o processamento; o netbook serve apenas como **tela, teclado e
trackpad**, ligados por um único cabo USB.

Em outras palavras: o netbook **não vira um PC**. Ele vira um **terminal** do celular.

```
Samsung Galaxy (One UI 8 / Android 16)  ──cabo USB──►  Eee PC 1015BX
   (processa, roda o DeX de verdade)                   (só exibe imagem e envia teclado/mouse)
```

## Hardware alvo

| Item | Especificação |
|---|---|
| Modelo | Asus Eee PC 1015BX |
| Processador | AMD C-60 — 2 núcleos, 1,0 GHz (turbo até 1,33 GHz), x86-64 (64 bits) |
| Vídeo | APU AMD Radeon HD 6290 (integrada, sem memória dedicada) |
| Memória RAM | 2 GB DDR3 (parte é reservada para o vídeo; sobram ~1,7 GB para o sistema) |
| Tela | 10,1", 1024×600 (proporção 5:3), painel fosco |
| Disco | HDD de 5400 rpm (250 GB ou 320 GB, conforme o lote) |
| Portas | 2× USB 2.0, **HDMI** e **VGA** (ambas somente saída), Ethernet 10/100, leitor de cartão SD, P2 de áudio (fone/microfone) |
| Firmware | BIOS legado (sem UEFI) — o boot é feito por MBR/GRUB `i386-pc` |
| Teclado | ABNT2 nos modelos vendidos no Brasil |

Consequências práticas desse hardware:

- **2 GB de RAM é o piso, não a folga.** O sistema é montado para caber num orçamento de
  ~400 MB em repouso.
- **Não existe entrada de vídeo.** O 1015BX tem HDMI e VGA, mas as duas são **saídas**:
  servem para mandar a imagem do netbook para um monitor/TV, não para receber a imagem do
  celular. Por isso o caminho de vídeo é USB (veja [`PLAN.md`](PLAN.md) §3).
- **A CPU é de 2011.** O alvo é 15 fps em 1024×600, com decodificação por software.
- **A porta USB 2.0 entrega ~500 mA**, menos do que um Galaxy moderno consome em uso.
- **As saídas HDMI e VGA são um bônus:** o netbook pode espelhar ou estender a sessão para
  uma TV/monitor maior. A resolução do DeX continua 1024×600 — a imagem é ampliada, não
  ganha detalhe.
- **O P2 de áudio é a saída natural para som**, caso o áudio seja ligado (padrão é desligado).

## Status

| Fase | Conteúdo | Estado |
|---|---|---|
| M0 | Plano de desenvolvimento | ✅ [`PLAN.md`](PLAN.md) |
| M1 | Daemon do kiosk rodando na distro atual do netbook | ⬜ |
| M2 | Validação no aparelho real (DeX + teclado/trackpad) | ⬜ |
| M3 | Build da imagem (`dex-station.img`) | ⬜ |
| M4 | Live USB | ⬜ |
| M5 | Instalação no HD | ⬜ |
| M6 | Calibração de performance + documentação | ⬜ |
| M7 | Estabilização e release v1.0 | ⬜ |

## Por onde começar

1. Ler [`PLAN.md`](PLAN.md) — arquitetura, riscos, matriz de testes e roadmap.
2. **M1**: implementar `runtime/dex-station-session` e rodar **na distribuição que já está
   instalada no netbook**. O ciclo de teste cai de horas (build de imagem) para minutos.
3. **M2**: repetir com o celular conectado de verdade.

## Variantes de entrega

As duas variantes usam **a mesma** `dex-station.img`. Muda só o destino da gravação:

- **Live USB** — `scripts/make-live-usb.sh`: grava a imagem num pendrive. O HD do netbook
  não é tocado, nada é instalado. É o modo recomendado para testar.
- **Instalação no HD** — `scripts/install-on-disk.sh`: faz backup, grava a imagem no disco
  interno e instala o GRUB. Sistema dedicado e permanente.

Detalhes de cada procedimento em [`PLAN.md`](PLAN.md) §7 e §8.

## Requisitos do celular

1. One UI 8 / Android 16 (é a versão em que o DeX passou a rodar em *display virtual*).
2. **Depuração USB** habilitada e a chave RSA deste netbook autorizada.
3. Celular **desbloqueado** durante o uso — com a tela bloqueada, o DeX não inicia.
4. Celular com bateria suficiente ou ligado a um carregador (veja o aviso abaixo).
5. Se usar `--keyboard=uhid`: Ajustes → Sistema → Idiomas e entrada → Teclado físico →
   **Português (Brasil)**.

## Aviso importante sobre energia

A porta USB 2.0 do 1015BX oferece **~500 mA**. Um Galaxy em uso com DeX consome mais do que
isso, então **a bateria do netbook vai ser drenada** e o celular pode até descarregar
devagar. Soluções, em ordem de preferência:

1. **Hub USB alimentado** (com fonte própria) — o netbook só transporta dados.
2. Carregador ligado direto no celular + cabo de dados para o netbook.
3. Uso curto, com o netbook na tomada, aceitando a descarga.

## Limitações conhecidas

- Sem aceleração de vídeo no scrcpy: a decodificação é feita pela CPU (a APU tem decoder
  UVD3, mas o scrcpy não o utiliza). Por isso o teto de 15 fps.
- Sem áudio por padrão é a escolha mais segura para 2 GB de RAM (decisão em aberto no plano).
- O pendrive do modo Live **não guarda nada** entre boots — é descartável por projeto.
