# Acelerômetro ADXL345 e input shaper

O input shaper do Klipper compensa as vibrações da estrutura e reduz as ondulações ("ringing" ou "ghosting")
que aparecem depois dos cantos e das mudanças de direção. Com ele, a impressora do autor passou de 1500 para
4000 mm/s² de aceleração.

As frequências de ressonância podem ser medidas com um acelerômetro ADXL345 ou estimadas à mão, imprimindo
a torre de teste do [guia do Klipper](https://www.klipper3d.org/Resonance_Compensation.html). O
acelerômetro dá um resultado mais preciso e rápido.

## Ligação

Na impressora do autor, o módulo ADXL345 fica **permanentemente** preso na cabeça de impressão e é ligado
por um cabo ao **SPI0 do Raspberry Pi**, e não à RAMPS. Assim o Pi lê o sensor diretamente, por meio do
"MCU Linux" (`[mcu rpi]`, ver [firmware.md](firmware.md)).

| ADXL345 | Raspberry Pi (pino físico) |
|---|---|
| 3V3 (ou VCC) | 3,3 V (pino 1) |
| GND | GND (pino 6 ou 9) |
| CS | GPIO8 / CE0 (pino 24) |
| SDO | GPIO9 / MISO (pino 21) |
| SDA | GPIO10 / MOSI (pino 19) |
| SCL | GPIO11 / SCLK (pino 23) |

Esta é a ligação padrão do
[guia de medição de ressonâncias do Klipper](https://www.klipper3d.org/Measuring_Resonances.html). Confira os
fios do seu módulo.

Cuidados:
- Alimente o módulo com **3,3 V**.
- Prenda o módulo firme na cabeça de impressão. Se ele balançar, a medida sai errada.
- O cabo acompanha o movimento da cabeça de impressão: prenda-o junto aos outros cabos, com folga, para não
  dobrar sempre no mesmo ponto.

## Configuração

No Pi:
1. SPI habilitado: `dtparam=spi=on` em `/boot/firmware/config.txt`, e reiniciar.
2. MCU Linux instalado e o serviço `klipper_mcu` ativo ([firmware.md](firmware.md)).
3. Usuário `pi` no grupo `tty` ([instalacao.md](instalacao.md#6-ajustes-do-sistema)).
4. `numpy` no ambiente do Klipper ([instalacao.md](instalacao.md#3-klipper)) e o pacote `libopenblas0`
   ([instalacao.md](instalacao.md#2-pacotes-do-sistema)).

No `printer.cfg` (já presente nos arquivos deste repositório):

```ini
[mcu rpi]
serial: /tmp/klipper_host_mcu

[adxl345]
cs_pin: rpi:None

[resonance_tester]
accel_chip: adxl345
probe_points:
    110,110,20
```

## Sem acelerômetro

Os arquivos de configuração deste repositório vêm com o acelerômetro ativo. Sem ele:

1. Comente todas as linhas das seções `[mcu rpi]`, `[adxl345]` e `[resonance_tester]`, e não só o título:
   uma linha que fica sem a sua seção passa a fazer parte da seção de cima. A que realmente impede o Klipper de
   iniciar é a `[mcu rpi]`, quando o serviço `klipper_mcu` não está rodando; o ADXL345 em si só é lido
   durante uma medição.
2. Comente também a `[input_shaper]` inteira: as frequências nela foram medidas na impressora do autor. Se
   quiser o input shaper, meça as suas com a torre de teste do [guia do
   Klipper](https://www.klipper3d.org/Resonance_Compensation.html).
3. Use `max_accel: 1500` na seção `[printer]`, o valor que o autor usava antes do input shaper.
4. Nos perfis do PrusaSlicer deste repositório, que gravam acelerações no G-code e passam por cima do
   `max_accel` (ver [fatiador.md](fatiador.md#acelerações-e-velocidades)): no G-code final do perfil da
   impressora, troque `M204 S4000` por `M204 S1500`; no perfil de impressão, baixe a aceleração dos
   deslocamentos de 2000 para 1500 mm/s² ou menos.

## Medir

No console do Fluidd, depois de um `G28`:

1. `ACCELEROMETER_QUERY`: deve responder com três valores. Se der erro, confira a ligação e o SPI.
2. `MEASURE_AXES_NOISE`: mede o ruído do sensor com a impressora parada.
3. `SHAPER_CALIBRATE`: vibra a impressora nos eixos X e Y (faz barulho, é normal) e sugere o tipo e a
   frequência de shaper para cada eixo.
4. `SAVE_CONFIG` para gravar o resultado.

Para ver os gráficos, use `TEST_RESONANCES AXIS=X` (e `Y`) e o script `~/klipper/scripts/calibrate_shaper.py`,
como no guia do Klipper.

## Resultados na impressora do autor

| Medição | Eixo X | Eixo Y |
|---|---|---|
| outubro de 2021 | `ei` 52,6 Hz | `ei` 60,6 Hz |
| 2 de junho de 2022, em uso até hoje | `3hump_ei` 80,8 Hz | `3hump_ei` 90,8 Hz |

Na medição de 2022, o script do Klipper recomendou `ei` a 54,8 Hz no X e `mzv` a 50,4 Hz no Y; o autor
escolheu o `3hump_ei`, que no mesmo relatório aparece com vibração residual de 0 % e aceleração máxima
sugerida de 4800 mm/s² no X e 6000 mm/s² no Y. É com esses valores que a
impressora do autor trabalha a 4000 mm/s².

## Problema conhecido: escala das leituras

No sensor do autor, as leituras saem cerca de **4 vezes maiores** que o esperado. Parado, o eixo vertical
deveria marcar a gravidade, ~9800 mm/s², e marca ~39000. O ruído do `MEASURE_AXES_NOISE` também vem 4 vezes
maior.

O que já foi verificado:
- O sensor responde com a identificação correta do ADXL345 (`DEVID` = 0xE5) e com os registros configurados
  como o Klipper espera (faixa ±16 g, resolução total, 3200 Hz).
- O chip tem marcação de ADXL345 genuíno, mas marcação não prova autenticidade, e existem clones que
  respondem com o mesmo ID.
- O problema é o mesmo em sistema operacional e versões do Klipper diferentes.

A causa ainda é desconhecida. Isso **não atrapalha o input shaper**, porque as frequências de ressonância
não dependem da escala absoluta. Se você tiver o mesmo sintoma e descobrir a causa, abra uma issue.
