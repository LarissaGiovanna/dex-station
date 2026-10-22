# DeX Station OS

Transforma um netbook antigo (Asus Eee PC 1000/1000H) numa **estação DeX dedicada**: o
Samsung Galaxy faz o processamento, o netbook vira monitor + teclado + trackpad via USB.

```
Samsung Galaxy (One UI 8 / Android 16)  ──cabo USB──►  Netbook
        (processa, DeX real)                          (exibe + entrada)
```

## Status

| Fase | Conteúdo | Estado |
|---|---|---|
| M0 | Plano de desenvolvimento | ✅ [`PLAN.md`](PLAN.md) |
| M1 | Daemon do kiosk na distro atual | ⬜ |
| M2 | Validação no aparelho real (DeX + entrada) | ⬜ |
| M3 | Build da imagem (`dex-station.img`) | ⬜ |
| M4 | Live USB | ⬜ |
| M5 | Instalação no HD | ⬜ |
| M6 | Performance + documentação | ⬜ |

## Por onde começar

1. Ler [`PLAN.md`](PLAN.md) — arquitetura, riscos, matriz de testes, roadmap.
2. **M1**: implementar `runtime/dex-station-session` e rodar na distro atual do netbook
   (feedback em minutos, sem construir imagem).
3. **M2**: validar com o celular real plugged.

## Variantes de entrega

- **Live USB** — `scripts/make-live-usb.sh`: mesma imagem gravada em pendrive; o HD do
  netbook não é tocado. Ideal para testar.
- **Instalação no HD** — `scripts/install-on-disk.sh`: backup + gravação da imagem +
  GRUB. Sistema dedicado permanente.

Ambas saem da **mesma** `dex-station.img`; a diferença é apenas o alvo do `dd`.

## Requisitos do celular

1. One UI 8 / Android 16 (DeX novo em display virtual).
2. Depuração USB habilitada e RSA autorizado nesta máquina.
3. Celular desbloqueado durante o uso.
4. Ajustes → Sistema → Idiomas e entrada → Teclado físico → **Português (Brasil)**
   (necessário se usar `--keyboard=uhid`).

## Aviso importante sobre energia

A porta USB do Eee PC 1000 entrega ~500 mA. O celular **vai drenar a bateria do
netbook**. Use um **hub USB alimentado** ou alimente o celular no carregador e use só
o cabo de dados.
