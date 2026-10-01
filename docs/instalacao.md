# Instalação do Raspberry Pi com Klipper, Moonraker e Fluidd

Passo a passo usado pelo autor em setembro de 2026 para montar um cartão SD novo, num Raspberry Pi 2 com
Raspberry Pi OS trixie. As versões usadas foram Klipper v0.13.0, Moonraker v0.11.0 e Fluidd v1.37.6.

A instalação é manual, para ficar claro o que cada componente faz. Alternativa mais automática: o
[KIAUH](https://github.com/dw-0/kiauh), um script que instala Klipper, Moonraker e Fluidd por menus. O
autor não testou o KIAUH nesta impressora. Se usá-lo no lugar das seções 3 a 5, confira ainda o `numpy` do
Klipper (`~/klippy-env/bin/pip install numpy`, seção 3, necessário para o acelerômetro), faça a seção
[6](#6-ajustes-do-sistema) e siga daí em diante: o firmware e o MCU Linux são instalados em
[Configurar a impressora](#configurar-a-impressora).

A imagem "FluiddPi", citada em versões antigas deste repositório, não é mais mantida.

## Visão geral

| Componente | Função | Onde fica |
|---|---|---|
| Klipper (`klippy`) | firmware, parte que roda no Pi | `~/klipper`, ambiente Python em `~/klippy-env` |
| Firmware dos MCUs | parte que roda na RAMPS e no Pi | ver [firmware.md](firmware.md) |
| Moonraker | API web entre o Klipper e as interfaces | `~/moonraker`, `~/moonraker-env` |
| Fluidd | interface no navegador (arquivos estáticos) | `~/fluidd`, servido pelo nginx |
| Dados | configuração, arquivos G-code, logs, banco de dados | `~/printer_data/{config,gcodes,logs,database,comms}` |

## 1. Gravar o cartão SD

Use o [Raspberry Pi Imager](https://www.raspberrypi.com/software/):

- Sistema: **Raspberry Pi OS Lite (32 bits)**. O Pi 2 v1.1, o do autor, não roda o de 64 bits; o Pi 2 v1.2
  e os modelos mais novos rodam.
- Nas opções de personalização: nome do host (ex.: `impressora3d`), usuário `pi` com senha, Wi-Fi com o
  **país BR**, fuso horário, e SSH habilitado com a sua chave pública.

Depois de ligar o Pi, acesse com `ssh pi@impressora3d.local`.

Os comandos abaixo assumem o usuário `pi`. Com outro usuário, ajuste os caminhos `/home/pi`, o `User=` do
serviço do Klipper e o `usermod` da seção 6.

## 2. Pacotes do sistema

```bash
sudo apt update && sudo apt full-upgrade -y
sudo apt install -y git nginx build-essential python3-dev python3-venv \
  libffi-dev libncurses-dev libusb-dev libusb-1.0-0-dev pkg-config \
  avrdude gcc-avr binutils-avr avr-libc \
  libjpeg-dev zlib1g-dev libsodium-dev liblmdb-dev libopenjp2-7 curl unzip rsync \
  python3-numpy python3-matplotlib libopenblas0
```

- `avrdude`, `gcc-avr`, `binutils-avr` e `avr-libc` compilam e gravam o firmware da RAMPS.
- `python3-numpy` e `python3-matplotlib` são para o script `calibrate_shaper.py`, que gera os gráficos do
  input shaper.
- `libopenblas0` é necessária para o `numpy` do Klipper, usado na calibração com o acelerômetro. Sem ela,
  comandos como `MEASURE_AXES_NOISE` falham com erro de importação do `numpy`.

## 3. Klipper

```bash
cd ~
git clone https://github.com/Klipper3d/klipper.git
python3 -m venv ~/klippy-env
~/klippy-env/bin/pip install -r ~/klipper/scripts/klippy-requirements.txt
~/klippy-env/bin/pip install numpy          # para o acelerômetro
mkdir -p ~/printer_data/{config,gcodes,logs,comms,database}
```

Crie o serviço `/etc/systemd/system/klipper.service`:

```ini
[Unit]
Description=Klipper 3D Printer Firmware
Documentation=https://www.klipper3d.org/
After=network-online.target klipper_mcu.service
Wants=udev.target

[Install]
Alias=klippy.service
WantedBy=multi-user.target

[Service]
Type=simple
User=pi
RemainAfterExit=yes
WorkingDirectory=/home/pi/klipper
ExecStart=/home/pi/klippy-env/bin/python /home/pi/klipper/klippy/klippy.py /home/pi/printer_data/config/printer.cfg -I /home/pi/printer_data/comms/klippy.serial -l /home/pi/printer_data/logs/klippy.log -a /home/pi/printer_data/comms/klippy.sock
Restart=always
RestartSec=10
```

O `Alias` precisa ser `klippy.service`, com o sufixo. Com só `klippy`, o `systemctl enable` falha.

```bash
sudo systemctl daemon-reload
sudo systemctl enable klipper
```

O firmware da RAMPS e o MCU Linux são instalados mais adiante, em
[Configurar a impressora](#configurar-a-impressora).

## 4. Moonraker

```bash
cd ~
git clone https://github.com/Arksine/moonraker.git
MOONRAKER_DATA_PATH=/home/pi/printer_data MOONRAKER_FORCE_SYSTEM_INSTALL=y MOONRAKER_SPEEDUPS=y \
  bash ~/moonraker/scripts/install-moonraker.sh
```

O instalador cria o ambiente `~/moonraker-env`, o serviço `moonraker`, as regras de permissão (para o
Moonraker poder reiniciar serviços e o Pi) e um `moonraker.conf` padrão. Na primeira vez que roda, o
Moonraker cria o arquivo `~/printer_data/moonraker.asvc`, que lista os serviços que ele pode controlar.
Confira se `klipper_mcu` está na lista.

Troque o conteúdo de `~/printer_data/config/moonraker.conf` pela configuração abaixo, que é a do autor sem o
`force_logins`:

```ini
[server]
host: 0.0.0.0
port: 7125
klippy_uds_address: ~/printer_data/comms/klippy.sock

[file_manager]

[data_store]
temperature_store_size: 600
gcode_store_size: 1000

[authorization]
cors_domains:
  *.local
  *.lan
  *://app.fluidd.xyz
trusted_clients:
  10.0.0.0/8
  127.0.0.0/8
  169.254.0.0/16
  172.16.0.0/12
  192.168.0.0/16
  FE80::/10
  ::1/128

# permite o envio direto do PrusaSlicer (ver fatiador.md)
[octoprint_compat]

# histórico de impressões e totais ("hodômetro")
[history]

[update_manager]
enable_auto_refresh: True

[update_manager client fluidd]
type: web
repo: fluidd-core/fluidd
path: ~/fluidd
```

Depois de salvar, reinicie o Moonraker: `sudo systemctl restart moonraker`.

`trusted_clients` libera o acesso sem senha para a rede local. Para exigir login, acrescente
`force_logins: True` à seção `[authorization]` e crie um usuário pelo Fluidd. A partir daí a rede local
também precisa de login, inclusive o próprio Pi: o envio pelo PrusaSlicer e os comandos `curl` desta
documentação passam a precisar da chave de API do Moonraker (nos `curl`, no cabeçalho `X-Api-Key`). O
comando `~/moonraker/scripts/fetch-apikey.sh` mostra a chave.

## 5. Fluidd e nginx

```bash
mkdir -p ~/fluidd && cd ~/fluidd
curl -sLO https://github.com/fluidd-core/fluidd/releases/latest/download/fluidd.zip
unzip fluidd.zip && rm fluidd.zip
chmod 711 /home/pi
```

O `chmod 711 /home/pi` é necessário porque o Raspberry Pi OS atual cria a pasta do usuário sem acesso para os
outros usuários, e o nginx não consegue ler o Fluidd (erro 403).

Crie `/etc/nginx/sites-available/fluidd` com o conteúdo abaixo, que é o arquivo em uso na impressora do
autor (baseado no exemplo do Fluidd). O `client_max_body_size 0` é o que permite enviar arquivos G-code
maiores que 1 MB, o limite padrão do nginx.

```nginx
upstream apiserver {
    ip_hash;
    server 127.0.0.1:7125;
}

map $http_upgrade $connection_upgrade {
    default upgrade;
    ""      close;
}

server {
    listen 80 default_server;
    listen [::]:80 default_server;

    access_log /home/pi/printer_data/logs/fluidd-access.log;
    error_log /home/pi/printer_data/logs/fluidd-error.log;

    gzip on;
    gzip_vary on;
    gzip_proxied any;
    gzip_comp_level 4;
    gzip_buffers 16 8k;
    gzip_http_version 1.1;
    gzip_types text/plain text/css text/xml text/javascript application/x-javascript application/json application/xml;

    root /home/pi/fluidd;
    index index.html;
    server_name _;

    client_max_body_size 0;
    proxy_request_buffering off;

    location / {
        try_files $uri $uri/ /index.html;
    }

    location = /index.html {
        add_header Cache-Control "no-store, no-cache, must-revalidate";
    }

    location /websocket {
        proxy_pass http://apiserver/websocket;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;
        proxy_set_header Host $http_host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Scheme $scheme;
        proxy_read_timeout 86400;
    }

    location ~ ^/(printer|api|access|machine|server)/ {
        proxy_pass http://apiserver$request_uri;
        proxy_set_header Host $http_host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Scheme $scheme;
    }

    location /webcam/ {
        postpone_output 0;
        proxy_buffering off;
        proxy_ignore_headers X-Accel-Buffering;
        access_log off;
        error_log off;
        proxy_pass http://127.0.0.1:8080/;
    }
}
```

Depois ative o site:

```bash
sudo ln -s /etc/nginx/sites-available/fluidd /etc/nginx/sites-enabled/
sudo rm /etc/nginx/sites-enabled/default
sudo nginx -t && sudo systemctl restart nginx
```

Acesse `http://impressora3d.local` no navegador.

## 6. Ajustes do sistema

- **Acesso ao MCU Linux:** `sudo usermod -aG tty pi`. O `klipper_mcu` roda como root e cria
  `/tmp/klipper_host_mcu`; sem o grupo `tty`, o Klipper falha com "Permission denied" no `[mcu rpi]`.
- **SPI para o acelerômetro:** acrescente `dtparam=spi=on` em `/boot/firmware/config.txt`.
- **Nome `.local` instável:** o endereço `impressora3d.local` às vezes deixa de responder, especialmente em
  roteadores que lidam mal com o IPv6 do mDNS. Duas medidas:
  - em `/etc/avahi/avahi-daemon.conf`, na seção `[server]`, use `use-ipv6=no` e reinicie o `avahi-daemon`;
  - no roteador, reserve um IP fixo (reserva de DHCP) para o endereço MAC do Wi-Fi do Pi. Assim o
    endereço numérico funciona mesmo quando o `.local` falha.

Reinicie o Pi depois desses ajustes.

## Configurar a impressora

1. Baixe este repositório no Pi:
   ```bash
   git clone https://github.com/mirandadam/klipper_gtmax3d.git ~/klipper_gtmax3d
   ```
2. Copie o arquivo do seu bico para o lugar do `printer.cfg`, por exemplo, para o bico de 0,4 mm:
   ```bash
   cp ~/klipper_gtmax3d/printer-gtmax3d-core-a2v2-bico0.4.cfg ~/printer_data/config/printer.cfg
   ```
   Para o bico de 0,6 mm, use o
   [`printer-gtmax3d-core-a2v2-bico0.6.cfg`](../printer-gtmax3d-core-a2v2-bico0.6.cfg).
3. Ligue o Pi à impressora pelo cabo USB. No `serial:` da seção `[mcu]`, mantenha o `/dev/serial/by-id/` e
   troque o nome que vem depois pelo que o comando `ls /dev/serial/by-id/` mostra.
4. Sem acelerômetro, siga [Sem acelerômetro](acelerometro.md#sem-acelerômetro).
5. Confira os `dir_pin` conforme os seus drivers e fios: ver
   [hardware.md](hardware.md#direção-dos-motores).
6. Compile e grave o firmware da RAMPS e, se tiver acelerômetro, instale o MCU Linux: ver
   [firmware.md](firmware.md). No fim, a impressora deve aparecer pronta ("Ready") no Fluidd.

## Primeiro teste de movimento

Faça com a mão no interruptor liga/desliga da impressora, pronto para cortar a energia.

1. `QUERY_ENDSTOPS` no console, com todos os fins de curso soltos: todos devem aparecer abertos
   (`open`). Aperte cada um com a mão e repita: o apertado deve aparecer acionado (`TRIGGERED`).
2. Leve a cabeça de impressão à mão para o meio da mesa e teste cada motor com `STEPPER_BUZZ
   STEPPER=stepper_x` (depois `stepper_y`, `stepper_z` e `extruder`): o motor anda 1 mm e volta, dez vezes. Os
   detalhes e o que observar estão no [Config checks do
   Klipper](https://www.klipper3d.org/Config_checks.html). Num CoreXY, cada motor sozinho move a cabeça de
   impressão na diagonal; é esperado.
3. Leve à origem (home) um eixo por vez (`G28 X`, `G28 Y`, `G28 Z`) e confira se cada um vai na direção do
   seu fim de curso. Se um eixo for para o lado errado, desligue a energia na hora e confira a
   [direção dos motores](hardware.md#direção-dos-motores).
4. Aqueça o bico e a mesa e confira se as temperaturas sobem de forma estável.
5. Com o bico quente, mande extrudar 50 mm e confira o sentido da extrusora.

Referência em vídeo (em inglês): [Klipper Initial Setup: Making sure things are all good before
printing](https://www.youtube.com/watch?v=T-knWbh1Gg8).

## Calibrações

Recomendadas na ordem abaixo. Todos os comandos vão no console do Fluidd.

1. **PID** do bico e da mesa: `PID_CALIBRATE HEATER=extruder TARGET=240`,
   `PID_CALIBRATE HEATER=heater_bed TARGET=105` e depois `SAVE_CONFIG`.
2. **z_offset** da sonda: refaça sempre que trocar o bico. Se na primeira camada o bico fica longe da mesa
   e as linhas não grudam, aumente o `z_offset`; se o bico raspa a mesa, diminua. Durante a primeira
   camada, os botões de ajuste de Z do Fluidd (`SET_GCODE_OFFSET`) sobem ou descem o bico na hora, mas esse
   ajuste se perde ao reiniciar. Depois que a impressão terminar, `Z_OFFSET_APPLY_PROBE` seguido de
   `SAVE_CONFIG` passa o ajuste para o `z_offset` (o `SAVE_CONFIG` reinicia o Klipper).

   Para medir do zero, o procedimento do Klipper é o `PROBE_CALIBRATE`, com o teste do papel. A sonda desta
   impressora só desce quando a cabeça de impressão vai a X = 43, Y = −5 (a macro `G29` vai até lá com
   Z = 20) e recolhe quando é comprimida contra a mesa. O autor não testou nenhum dos dois comandos nesta
   sonda.
3. **rotation_distance** da extrusora: com o bico quente, marque o filamento 70 mm acima da entrada da
   extrusora, mande extrudar 50 mm devagar (`M83` e depois `G1 E50 F60`; 50 mm é o máximo que o cfg aceita
   de uma vez) e meça quanto sobrou até a marca. O valor novo é o atual × (70 − sobra) / 50: se saiu menos
   de 50 mm, o valor diminui. Detalhes no
   [guia do Klipper](https://www.klipper3d.org/Rotation_Distance.html#calibrating-rotation_distance-on-extruders).
   A extrusora original deu 19,15.
4. **Input shaper**, se tiver acelerômetro: ver [acelerometro.md](acelerometro.md).
5. **Pressure advance**: pelo [procedimento do Klipper](https://www.klipper3d.org/Pressure_Advance.html).

Depois de um `SAVE_CONFIG`, o Klipper grava os valores no fim do `printer.cfg`, num bloco que começa com
`#*# <---------------------- SAVE_CONFIG ---------------------->`. Os valores desse bloco valem no lugar dos
que estiverem no começo do arquivo.
