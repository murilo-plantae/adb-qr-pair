# adb-qr-pair

Depuração sem fio no Android sem abrir o Android Studio.

A IDE inteira sobe só pra desenhar um QR code e escutar mDNS. Aqui são dois scripts
bash que fazem isso no terminal:

- **`adb-qr-pair`** — pareia e conecta o aparelho
- **`adb-shot`** — captura a tela ativa e traz o PNG pra máquina

## Instalação

Precisa de `adb` (platform-tools 30+, pelo mDNS embutido) e `qrencode`.
O `adb-shot` usa Pillow para o `--scale` — opcional.

```bash
sudo apt install qrencode python3-pil     # ou: brew install qrencode

for f in adb-qr-pair adb-shot; do
  curl -o ~/.local/bin/$f \
    https://raw.githubusercontent.com/murilo-plantae/adb-qr-pair/main/$f
  chmod +x ~/.local/bin/$f
done
```

## adb-qr-pair

No celular: **Opções do desenvolvedor → Depuração sem fio**.

```bash
adb-qr-pair             # reconecta via mDNS; só mostra o QR se não estiver pareado
adb-qr-pair --pair      # força o QR (device novo ou pareamento revogado)
adb-qr-pair --connect   # só reconecta, nunca mostra QR
adb-qr-pair --invert    # inverte as cores, se a câmera não ler no seu tema de terminal
adb-qr-pair --timeout N # segundos esperando o device no mDNS (default 120)
```

```
██ ▄▄▄▄▄ █   █▄▄▄█▄▀▀▀██ ▄▄▄▄▄ ██
██ █   █ █ ▀▄ █▀▀▀▀▄ ▄██ █   █ ██
██ █▄▄▄█ █▀██▀▀▄▄▀▀▀▄▀ █ █▄▄▄█ ██
             ( ... )
✓ pareado com 192.168.0.12:37115
✓ conectado: 192.168.0.12:40221
```

O QR do Android Studio não tem mágica nenhuma. É esta string:

```
WIFI:T:ADB;S:<nome>;P:<senha>;;
```

com nome e senha aleatórios. Quando o celular lê o código, ele passa a anunciar o
serviço mDNS `_adb-tls-pairing._tcp` usando aquele `<nome>`, e o host só precisa de
`adb pair <ip>:<porta> <senha>`. Depois do pareamento o device anuncia
`_adb-tls-connect._tcp` numa outra porta, que é onde o `adb connect` entra.

Não é preciso `avahi-browse` nem `dns-sd`: o `adb` moderno traz o próprio stack
mDNS (openscreen).

## adb-shot

```bash
adb-shot                # captura a tela ativa -> ~/Pictures/adb-shots/<timestamp>.png
adb-shot -s 40          # redimensiona pra 40%, bom pra colar em issue/chat
adb-shot -o /tmp/x.png  # destino específico
adb-shot -w             # espelha: toda print tirada NO celular é puxada automaticamente
adb-shot -O             # abre no visualizador
adb-shot --no-copy      # não copia pro clipboard
```

Copia pro clipboard (`wl-copy` no Wayland, `xclip` no X11) e mantém
`~/Pictures/adb-shots/last.png` apontando pra última captura — um caminho fixo é
prático pra ferramenta externa (ou um agente de IA) pegar a tela atual.

Usa `adb exec-out screencap -p`, com fallback via `adb shell screencap` + `adb pull`:
algumas ROMs devolvem lixo no `exec-out`, então o script valida o magic number do
PNG antes de aceitar o arquivo.

## Pegadinhas da depuração sem fio

**Ela cai a cada boot, e isso é de propósito.** O Android desliga a depuração sem fio
em todo reinício. O par de chaves do pareamento sobrevive; só o toggle não. Por isso
o modo default do `adb-qr-pair` tenta reconectar antes de mostrar QR — depois de um
boot, normalmente basta religar a chave no aparelho. Ela também cai ao sair da rede
Wi-Fi em que foi ativada.

**Anúncio mDNS fantasma.** Com o adbd derrubado, o responder mDNS do Android (UDP
5353) continua anunciando `_adb-tls-connect._tcp` com a porta da sessão morta. Aí
`adb mdns services` lista o device, parece tudo certo, e o TCP responde RST. A
assinatura é ping respondendo + zero portas TCP abertas no aparelho. O `adb-qr-pair`
detecta o caso e manda religar o toggle, em vez de dar um erro genérico.

**Transport duplicada.** O auto-connect mDNS registra o mesmo aparelho duas vezes, a
segunda com sufixo `" (2)"` no nome. São transports distintas, e todo comando morre
com `more than one device` até fixar uma com `adb -t <transport_id>`. Pior: o nome do
transport contém espaço, então `awk '$2=="device"'` não casa — tem que casar no
`" device "` da linha, nunca por posição de campo.

**Rede.** Os dois precisam estar na mesma rede, sem isolamento de clientes (AP
isolation), e VPN ativa costuma impedir o mDNS de enxergar a LAN.
