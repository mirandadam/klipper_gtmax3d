# Firmware: RAMPS (Arduino Mega) e MCU Linux

O Klipper tem duas partes: o programa principal (`klippy`), que roda no Raspberry Pi, e um firmware pequeno
que roda em cada microcontrolador ("MCU"). Nesta impressora há dois MCUs:

| MCU | Onde roda | Para quê |
|---|---|---|
| `mcu` | Arduino Mega 2560 da RAMPS | motores, aquecedores, sensores, display |
| `mcu rpi` | o próprio Raspberry Pi (programa `klipper_mcu`) | acelerômetro ADXL345 no SPI do Pi |

**O firmware dos MCUs precisa ser da mesma versão do Klipper do Pi.** Depois de atualizar o Klipper no Pi,
recompile e grave os dois; senão o Klipper recusa a conexão ou avisa que as versões diferem.

## Antes de tudo: peça o firmware original

Peça ao suporte da GTMax3D uma cópia do firmware original da sua impressora (arquivo `.hex`) **antes** de
gravar o Klipper. É com esse arquivo que se volta ao firmware original.

Outra saída, que o autor não testou, é ler o firmware da própria placa antes de gravar o Klipper, pelo mesmo
bootloader usado na gravação (com o Klipper parado; a porta é a do passo 4 da gravação, abaixo):
```bash
avrdude -p atmega2560 -c wiring -P /dev/serial/by-id/usb-Arduino__www.arduino.cc__0042_<número de série>-if00 -b 115200 -U flash:r:original_lido.hex:i
```

## Configurações de compilação

Como há dois MCUs, este repositório guarda uma configuração de compilação para cada um em
[`firmware_configs/`](../firmware_configs/):

| Arquivo | MCU | Opções principais |
|---|---|---|
| [`avr-ramps.config`](../firmware_configs/avr-ramps.config) | RAMPS | AVR `atmega2560`, 16 MHz, serial UART0 a 250000 baud |
| [`linux-host.config`](../firmware_configs/linux-host.config) | Raspberry Pi | "Linux process" |

Copie esses arquivos da cópia do repositório no Pi (`~/klipper_gtmax3d`, ver
[instalacao.md](instalacao.md#configurar-a-impressora)) para uma pasta própria, porque a compilação os
regrava:
```bash
mkdir -p ~/klipper_build_configs
cp ~/klipper_gtmax3d/firmware_configs/*.config ~/klipper_build_configs/
```
Para conferir ou mudar as opções num menu, use `make KCONFIG_CONFIG=<arquivo copiado> menuconfig`. Um
`make menuconfig` sem o `KCONFIG_CONFIG` grava em `~/klipper/.config`, que os comandos desta página não usam.
Para a RAMPS: *Enable extra low-level configuration options* ligado, *Micro-controller Architecture* = Atmega
AVR, *Processor model* = atmega2560, *Processor speed* = 16 MHz, *Communication interface* = UART0,
*Baud rate* = 250000.

Cada MCU compila numa pasta de saída própria (`out_avr/`, `out_linux/`), para uma compilação não apagar a
outra.

## RAMPS (Arduino Mega 2560)

A gravação usa o bootloader do Arduino, pelo mesmo cabo USB que liga o Pi à impressora. Não precisa de
programador nem de abrir a impressora. É igual na primeira vez e nas atualizações.

Pré-requisitos no Pi: os pacotes `avrdude gcc-avr binutils-avr avr-libc` (ver
[instalacao.md](instalacao.md#2-pacotes-do-sistema)).

1. **Deixe a impressora ociosa**: nada imprimindo, aquecedores desligados.

2. **Compile:**
   ```bash
   cd ~/klipper
   C=$HOME/klipper_build_configs/avr-ramps.config
   make KCONFIG_CONFIG=$C OUT=out_avr/ olddefconfig
   make KCONFIG_CONFIG=$C OUT=out_avr/ -j4
   ```
   O `KCONFIG_CONFIG` precisa ir como **argumento do `make`**, como acima. Como variável de ambiente, o
   Makefile do Klipper o ignora e usa o `~/klipper/.config`.

   O `olddefconfig` completa o arquivo com os padrões da versão do Klipper em uso, sem perguntar nada, e
   grava o resultado no próprio arquivo. No Raspberry Pi 2 a compilação leva alguns minutos.

3. **Pare o Klipper**, para ele soltar a porta serial:
   ```bash
   sudo systemctl stop klipper
   ```

4. **Descubra a porta serial** do Arduino:
   ```bash
   ls /dev/serial/by-id/
   ```
   Aparece algo como `usb-Arduino__www.arduino.cc__0042_<número de série>-if00`. O caminho completo,
   `/dev/serial/by-id/usb-Arduino__www.arduino.cc__0042_<número de série>-if00`, é o mesmo do `serial:` da
   seção `[mcu]` do `printer.cfg`.

5. **Grave**, no mesmo terminal do passo 2 (o comando usa o `$C`), com o caminho completo do passo 4:
   ```bash
   make KCONFIG_CONFIG=$C OUT=out_avr/ flash FLASH_DEVICE=/dev/serial/by-id/usb-Arduino__www.arduino.cc__0042_<número de série>-if00
   ```
   Saída esperada do `avrdude`: `Device signature = 0x1e9801 (probably m2560)`, seguida de
   `... bytes of flash written` e `... bytes of flash verified`. Leva uns 15 s. Essa saída é da última
   gravação do autor, feita no sistema anterior (Raspbian buster, `avrdude` 6.3). No Raspberry Pi OS trixie
   (`avrdude` 7.1) a compilação foi feita, mas a gravação ainda não.

6. **Religue o Klipper e confira:**
   ```bash
   sudo systemctl start klipper
   ```
   Se o `printer.cfg` já estiver no lugar e o MCU Linux já estiver instalado (ou as seções do acelerômetro
   estiverem comentadas, como em [Sem acelerômetro](acelerometro.md#sem-acelerômetro)),
   a impressora fica pronta ("Ready") no Fluidd. Antes disso, o Fluidd mostra um
   erro de conexão com o MCU `rpi`, o que não indica problema na RAMPS. No
   `~/printer_data/logs/klippy.log` aparece `Loaded MCU 'mcu'` com a versão igual à do Klipper. Uma mensagem
   `Serial connection closed` logo depois de gravar é normal: a placa reinicia quando a serial é aberta, e o
   Klipper reconecta sozinho.

Se algo der errado, grave de novo pelo mesmo caminho: a gravação não mexe no bootloader do Arduino.

Para voltar ao firmware original, o caminho esperado é gravar o `.hex` da GTMax3D (ou o lido da placa) com o
`avrdude`, pelo mesmo bootloader (com o Klipper parado):
```bash
avrdude -p atmega2560 -c wiring -P /dev/serial/by-id/usb-Arduino__www.arduino.cc__0042_<número de série>-if00 -b 115200 -D -U flash:w:original.hex:i
```
O autor tem o `.hex` original (ver [hardware.md](hardware.md#firmware-original)), mas nunca fez essa volta.

## MCU Linux (`klipper_mcu`), para o acelerômetro

Só é necessário se houver acelerômetro ligado ao Pi. O "firmware" é um programa comum do Linux, instalado
em `/usr/local/bin/klipper_mcu` e executado pelo serviço `klipper_mcu`.

**Compilar** (primeira vez e atualizações):
```bash
cd ~/klipper
L=$HOME/klipper_build_configs/linux-host.config
make KCONFIG_CONFIG=$L OUT=out_linux/ olddefconfig
make KCONFIG_CONFIG=$L OUT=out_linux/ -j4
```

**Primeira vez:** instale o programa e o serviço.
```bash
sudo cp out_linux/klipper.elf /usr/local/bin/klipper_mcu
sudo cp ~/klipper/scripts/klipper-mcu.service /etc/systemd/system/klipper_mcu.service
sudo systemctl daemon-reload
sudo systemctl enable --now klipper_mcu
sudo systemctl restart klipper
```
O Klipper distribui o serviço como `klipper-mcu.service`, com hífen. A cópia acima o renomeia para
`klipper_mcu.service`, com sublinhado, porque é esse o nome que o Moonraker espera na lista de serviços que
pode controlar (`~/printer_data/moonraker.asvc`) e o que o resto desta documentação usa.

**Atualização:** troque o programa com os serviços parados.
```bash
sudo systemctl stop klipper klipper_mcu
sudo cp /usr/local/bin/klipper_mcu ~/klipper_mcu.bak   # cópia da versão anterior
sudo cp out_linux/klipper.elf /usr/local/bin/klipper_mcu
sudo systemctl start klipper_mcu klipper
```

## Conferir as versões

No Fluidd, em *Sistema*, ou pela API:
```bash
curl -s "localhost:7125/printer/objects/query?mcu&mcu%20rpi" | grep -o '"mcu_version": *"[^"]*"'
```
As duas versões devem ser iguais à do Klipper (`git -C ~/klipper describe --tags`).

Se a versão aparecer com o hash abreviado em tamanhos diferentes (por exemplo `-g7bc4d09` e `-g7bc4d094`), o
Klipper pode acusar diferença mesmo sendo o mesmo commit. Isso acontece quando o firmware foi compilado num
clone do Klipper diferente do que roda no Pi (por exemplo, no cartão antigo), e cada clone abrevia o hash com
um tamanho. Para fixar: `git -C ~/klipper config core.abbrev 8` e reinicie o serviço com
`sudo systemctl restart klipper` (o `RESTART` do console não relê a versão). Se as versões ainda diferirem,
recompile e grave os MCUs.
