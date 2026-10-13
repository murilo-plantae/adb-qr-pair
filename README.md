# adb-qr-pair

Parear e conectar um Android por **depuração sem fio** sem abrir o Android Studio.

O Android Studio sobe inteiro só pra desenhar um QR code e escutar mDNS. Este script
faz as duas coisas no terminal, em ~100 linhas de bash.

```
$ adb-qr-pair

█████████████████████████████████
██ ▄▄▄▄▄ █   █▄▄▄█▄▀▀▀██ ▄▄▄▄▄ ██
██ █   █ █ ▀▄ █▀▀▀▀▄ ▄██ █   █ ██
██ █▄▄▄█ █▀██▀▀▄▄▀▀▀▄▀ █ █▄▄▄█ ██
██▄▄▄▄▄▄▄█▄▀▄█ ▀▄█▄▀▄▀ █▄▄▄▄▄▄▄██
             ( ... )

No celular: Depuração sem fio > Parear dispositivo com código QR
Aguardando (até 120s)... Ctrl-C para abortar.
✓ pareado com 192.168.0.12:37115
✓ conectado: 192.168.0.12:40221
```

## Instalação

Precisa de `adb` (platform-tools 30+, pelo mDNS embutido) e `qrencode`.

```bash
sudo apt install qrencode          # ou: brew install qrencode

curl -o ~/.local/bin/adb-qr-pair \
  https://raw.githubusercontent.com/murilo-plantae/adb-qr-pair/main/adb-qr-pair
chmod +x ~/.local/bin/adb-qr-pair
```

## Uso

No celular: **Opções do desenvolvedor → Depuração sem fio**.

```bash
adb-qr-pair             # reconecta via mDNS; só mostra o QR se não estiver pareado
adb-qr-pair --pair      # força o QR (device novo ou pareamento revogado)
adb-qr-pair --connect   # só reconecta, nunca mostra QR
adb-qr-pair --invert    # inverte as cores, se a câmera não ler no seu tema de terminal
adb-qr-pair --timeout N # segundos esperando o device no mDNS (default 120)
```

**O pareamento persiste entre reboots do celular** — o que muda é só a porta. Então
no dia a dia você raramente precisa do QR: ligue a chave da depuração sem fio e rode
`adb-qr-pair` sem argumento, que ele reconecta direto.

## Como funciona

O QR do Android Studio não tem mágica nenhuma. É esta string:

```
WIFI:T:ADB;S:<nome>;P:<senha>;;
```

com nome e senha aleatórios. Quando o celular lê o código, ele passa a anunciar o
serviço mDNS `_adb-tls-pairing._tcp` usando aquele `<nome>`, e o host só precisa de:

```bash
adb pair <ip>:<porta> <senha>
```

O script gera o par, renderiza o QR com `qrencode -t ANSIUTF8`, descobre o endereço
com `adb mdns services` e dispara o `adb pair`. Depois do pareamento o device anuncia
`_adb-tls-connect._tcp` numa outra porta, que é onde o `adb connect` entra.

Não é preciso `avahi-browse` nem `dns-sd`: o `adb` moderno traz o próprio stack mDNS
(openscreen).

## Limitações

- Os dois precisam estar na mesma rede, e a rede não pode ter isolamento de clientes
  (AP isolation) — mDNS não atravessa.
- Em VPN ativa o mDNS costuma não enxergar a LAN.
