# Hardware da Core A2V2 modificada

Este documento descreve a impressora do autor: uma GTMax3D Core A2V2 comprada em 2021 e modificada. Os
arquivos `printer-gtmax3d-core-a2v2-*.cfg` deste repositório foram feitos e testados nela, com exceção das
macros que o cabeçalho de cada arquivo aponta como testadas só em simulação.

## Resumo

| Item | Original de fábrica | Na impressora do autor |
|---|---|---|
| Firmware da placa | Marlin adaptado pela GTMax3D (código não divulgado) | Klipper |
| Placa controladora | RAMPS 1.4 + Arduino Mega 2560 | igual (original) |
| Drivers dos motores | A4988 (provável, ver abaixo) | TMC2209, com os fios de uma bobina de cada motor trocados |
| Computador | nenhum | Raspberry Pi 2 Model B, ligado à RAMPS por USB |
| Interface | display e cartão SD | Fluidd no navegador (display original continua funcionando) |
| Hotend e extrusora | originais, bico de 0,4 mm | originais, bico trocado para 0,6 mm |
| Sonda de nivelamento | original | igual |
| Sensor de fim de filamento | original | igual |
| Ventoinha frontal | original | Noctua NF-A4x20 PWM 5V |
| Acelerômetro | nenhum | ADXL345 na cabeça de impressão, ligado ao Raspberry Pi |

## Impressora

- CoreXY, área de impressão 220 × 220 × 240 mm (especificação divulgada pela revendedora 3DFila).
- Hotend all-metal até 295 °C; mesa de alumínio com vidro.
- Display: RepRapDiscount Smart Controller 2004 (LCD 20×4 HD44780, com encoder e bipe).
- Fonte, motores, correias e estrutura originais.

## Firmware original

O autor recebeu da GTMax3D duas versões do firmware original da A2V2: um `.hex` compilado em 2 de julho de
2020 e um `.bin` compilado em 28 de novembro de 2019. As duas se identificam como `Marlin Vers-20.1`, da
GTMax3D, para a "CoreA2v2". O código-fonte não é divulgado, e os arquivos não estão neste repositório. O que
segue foi lido no texto e nos valores gravados dentro deles.

O que o firmware original faz:

- Nivelamento automático da mesa pela sonda, com malha bilinear (`G29`). Tem também o teste de
  repetibilidade da sonda (`M48`) e a impressão de um padrão para validar a malha (`G26`).
- Troca de filamento (`M600`): estaciona a cabeça de impressão, descarrega, espera o filamento novo, carrega e
  purga. O display tem menus para carregar e descarregar filamento.
- Sensor de fim de filamento, que pode ser ligado e desligado pelo menu ou pelo `M412`.
- Ajustes durante a impressão pelo display: velocidade, ventoinha e Z offset.
- Calibração do PID (`M303`), proteção contra descontrole térmico ("thermal runaway") e temperatura máxima de
  310 °C no bico e 135 °C na mesa.
- Ajuste da potência máxima da mesa no menu.
- Estatísticas no display: total de impressões, tempo, filamento usado e trabalho mais longo.
- Arcos (`G2`/`G3`).
- Configurações guardadas na EEPROM e também no cartão SD; um menu de suporte protegido por senha.
- Menus em português; comunicação USB a 250000 baud.

O que ele **não** tem: linear advance (o equivalente do Marlin ao pressure advance), input shaping, controle
pela rede e retomada depois de queda de energia.

Valores padrão gravados no firmware (cada impressora pode ter valores diferentes salvos na EEPROM):

| | X | Y | Z | Extrusora |
|---|---|---|---|---|
| Passos por mm | 80 | 80 | 400 | 160 |
| Velocidade máxima (mm/s) | 200 | 200 | 40 | 70 |
| Aceleração máxima (mm/s²) | 1500 | 1500 | 100 | 3000 |

Com microstepping 1/16 (3200 passos por volta), esses passos por mm dão os `rotation_distance` 40, 40 e 8 do
cfg, e 20 na extrusora (o autor calibrou 19,15). Isso confirma que a impressora já saía de fábrica em 1/16.

## Placa controladora: RAMPS 1.4 + Arduino Mega 2560

- Microcontrolador ATmega2560 a 16 MHz; comunicação USB-serial a 250000 baud.
- É a mesma placa que a GTMax3D vende como peça de reposição para a linha Core (A1, A2, A2v2, A3v2 e
  outras). A página do fabricante não informa se o Arduino é original ou compatível.
- Com o Klipper, a placa só executa comandos de tempo real; o processamento fica no Raspberry Pi.
- O firmware é gravado pelo bootloader do Arduino, pelo próprio cabo USB. Não precisa de programador.
  Ver [firmware.md](firmware.md).

### Mapa de pinos

| Função | Pino |
|---|---|
| Motor X: step / dir / enable | PF0 / !PF1 / !PD7 |
| Motor Y: step / dir / enable | PF6 / !PF7 / !PF2 |
| Motor Z: step / dir / enable | PL3 / !PL1 / !PK0 |
| Extrusora: step / dir / enable | PA4 / PA6 / !PA2 |
| Fim de curso X / Y / Z | ^!PE5 / ^!PJ1 / ^!PD2 |
| Sonda | ^PD3 |
| Aquecedor / termistor do bico | PB4 / PK5 |
| Aquecedor / termistor da mesa | PH5 / PK6 |
| Ventoinha da peça | PH6 |
| Sensor de fim de filamento (chama o `M600` durante a impressão) | ^!PK1 |
| Bipe | EXP1_1 (PC0) |
| Display | conectores EXP1 e EXP2 |

Termistores do bico e da mesa: EPCOS 100K B57560G104F.

## Drivers dos motores

### Drivers originais

A GTMax3D vende como peça de reposição o conjunto de drivers **A4988** ("Driver Padrão A4988") e oferece
o **TMC2208** como opção silenciosa (4 unidades para a A2V2). Por isso o driver original provavelmente é o
A4988. O autor não guardou essa informação; confira nos drivers da sua impressora.

### TMC2209 na impressora do autor

- 4 drivers TMC2209 (X, Y, Z e extrusora), em **modo standalone**: o Klipper não conversa com eles por
  UART, e o `printer.cfg` não tem seções `[tmc2209]`.
- Microstepping 1/16, que precisa coincidir com o `microsteps: 16` do cfg. No TMC2209 em modo standalone,
  1/16 corresponde aos pinos MS1 e MS2 em nível alto, que é o que os jumpers da RAMPS fazem quando estão
  instalados. Sem jumpers, o TMC2209 fica em 1/8 e a impressora anda o dobro do comando.
- Corrente (Vref) no ajuste de fábrica do driver.
- Dissipadores e ventilação nos drivers.
- Os drivers foram trocados antes da instalação do Klipper (pelo que o autor lembra) e, em cada motor, os dois
  fios de uma das bobinas foram trocados de posição, para a impressora continuar funcionando com o firmware
  original.

Veja [upgrades.md](upgrades.md) para as vantagens da troca.

### Direção dos motores

O TMC2209 gira no sentido oposto ao do A4988 para o mesmo sinal de direção. Quem troca os drivers tem duas
saídas:

- trocar de posição, no conector de cada motor, os dois fios de **uma** das bobinas (por exemplo, os fios
  das posições 1A e 1B), o que inverte o sentido de giro. Foi o que o autor fez. Atenção: trocar também os
  fios da outra bobina desfaz a inversão;
- inverter a direção no `printer.cfg`, sem mexer nos fios (só vale com o Klipper).

Os `dir_pin` dos arquivos deste repositório são os da impressora do autor. Foram acertados na primeira noite
de configuração do Klipper (14 de agosto de 2021: o X e o Y, que vieram sem `!` do exemplo, ganharam o `!`) e
não mudaram mais.

| Drivers | Fios dos motores | `dir_pin` a usar | Situação |
|---|---|---|---|
| TMC2209 | fios de uma bobina trocados | os do repositório: `!PF1`, `!PF7`, `!PL1`, `PA6` | **confirmado** na impressora do autor |
| A4988 originais | originais | os do repositório (mesmos acima) | **provável**: a troca de fios devolve o sentido original |
| TMC2209 ou TMC2208 | originais | inverter os quatro: `PF1`, `PF7`, `PL1`, `!PA6` | **dedução, não testado** |
| TMC2208 | fios de uma bobina trocados | os do repositório | **provável**: gira como o TMC2209; não testado |

Para inverter a direção no cfg, basta pôr ou tirar o `!` antes do pino. Qualquer que seja o caso, **teste
antes de fazer o primeiro `G28`** (ver [instalacao.md](instalacao.md#primeiro-teste-de-movimento)). Num
CoreXY, um motor invertido faz a cabeça de impressão andar na diagonal ou no eixo errado e pode forçar a
mecânica contra o fim de curso.

## Raspberry Pi

- Raspberry Pi 2 Model B v1.1 (4 núcleos ARMv7 de 32 bits, 1 GB de RAM). Dá conta do Klipper, do Moonraker
  e do Fluidd com folga, mas é lento para compilar. Qualquer modelo mais novo também serve.
- O Pi 2 v1.1 só roda sistemas de **32 bits (armhf)**. O Pi 2 v1.2 e os modelos mais novos rodam também os de
  64 bits.
- Cartão SD de 16 GB é suficiente (o sistema com tudo instalado ocupa cerca de 8 GB, fora os arquivos
  G-code).
- Sem Wi-Fi embutido: usa um adaptador USB TP-Link Archer T2U PLUS (chip RTL8821AU). No Raspberry Pi OS
  trixie ele funciona com o driver do próprio kernel (`rtw88_8821au`), sem compilar nada. Em sistemas
  antigos (buster) era preciso compilar um driver à parte.
- Ligado à RAMPS pelo cabo USB, que fica conectado o tempo todo.

## Acelerômetro ADXL345

Fica permanentemente na cabeça de impressão e é ligado ao **SPI0 do Raspberry Pi** (não à RAMPS). Ver
[acelerometro.md](acelerometro.md).

## Limites físicos e cinemática

| Eixo | Mínimo | Máximo | Fim de curso | `rotation_distance` |
|---|---|---|---|---|
| X | −18 mm | 240 mm | −18 mm | 40 (correia de passo 2 mm, polia de 20 dentes) |
| Y | −9 mm | 230 mm | −9 mm | 40 (correia de passo 2 mm, polia de 20 dentes) |
| Z | −2 mm | 244 mm | 243 mm (mesa no ponto mais baixo) | 8 (fuso de 8 mm por volta) |
| Extrusora | | | | 19,15 (calibrado; extrusora original) |

- A mesa termina em X = 227 e Y = 229. Os limites negativos existem porque os fins de curso ficam fora da
  mesa; a purga do G-code inicial usa Y = −5.
- Velocidade máxima usada: 100 mm/s (300 é possível, mas gera ruído; 150 às vezes ressoa).
- Aceleração: 4000 mm/s² com input shaper; 1500 é um valor seguro sem ele.

## Mesa aquecida

**Não configure `max_power` acima de 0.2 na seção `[heater_bed]`.** Com 1.0 a mesa esquenta rápido demais,
com risco de queimar a resistência ou quebrar o vidro. O calor precisa de tempo para se espalhar.

O cfg limita a mesa a 130 °C (a revendedora 3DFila indica 135 °C).

## Sonda de nivelamento

- Sonda original, no pino `^PD3`, deslocada de X = −13 mm e Y = +35 mm em relação ao bico.
- Ela desce quando a cabeça de impressão vai a X = 43, Y = −5 e recolhe quando comprimida contra a mesa. A
  macro `G29` do cfg faz esses movimentos.
- O `z_offset` depende do comprimento do bico: recalibre ao trocar o bico.
