# klipper_gtmax3d

Configuração e documentação para usar o firmware [Klipper](https://www.klipper3d.org/) em impressoras 3D
GTMax3D da linha Core, com a placa original (RAMPS 1.4 + Arduino Mega 2560) e um Raspberry Pi.

O foco é a **Core A2V2** do autor, uma impressora comprada em 2021 e modificada: drivers TMC2209, Klipper com
Raspberry Pi, acelerômetro ADXL345 e bico de 0,6 mm. O repositório serve de referência para quem tem esse
hardware e de registro da configuração do próprio autor.

*O autor não tem relação comercial, patrocínio ou vínculo com a GTMax3D, além de ter comprado uma impressora.
A empresa não divulga o código do firmware original nem documentação para este tipo de modificação. Se você
não quer perder a garantia, usa a impressora profissionalmente ou não pode ficar um tempo sem ela, não faça
estas modificações.*

O suporte da GTMax3D ajudou com informações para este upgrade e sempre respondeu rápido.

***Se tiver dúvidas ou faltar alguma informação importante, abra uma
[issue](https://github.com/mirandadam/klipper_gtmax3d/issues).***

## Arquivos de configuração

| Arquivo | Impressora | Autor |
|---|---|---|
| [`printer-gtmax3d-core-a2v2-bico0.4.cfg`](printer-gtmax3d-core-a2v2-bico0.4.cfg) | Core A2V2 com o bico original, de 0,4 mm | @mirandadam |
| [`printer-gtmax3d-core-a2v2-bico0.6.cfg`](printer-gtmax3d-core-a2v2-bico0.6.cfg) | Core A2V2 com bico de 0,6 mm (em uso pelo autor) | @mirandadam |
| [`printer-gtmax3d-core-a1v1.cfg`](printer-gtmax3d-core-a1v1.cfg) | Core A1V1 | @zenaro147 (2023) |
| [`printer-gtmax-core-a3v2.cfg`](printer-gtmax-core-a3v2.cfg) | Core A3V2 | @J-Pozenato (2023) |

Os arquivos da A2V2 partem destas premissas (detalhes em [docs/hardware.md](docs/hardware.md)):

- **Drivers TMC2209, com os dois fios de uma das bobinas de cada motor trocados de posição.** Com os
  drivers originais e a ligação original, os mesmos `dir_pin` devem funcionar. Com TMC2209 e a ligação
  original, é preciso inverter as direções no cfg. Veja a
  [tabela de direção dos motores](docs/hardware.md#direção-dos-motores) antes de ligar os motores.
- **Acelerômetro ADXL345 no Raspberry Pi.** Sem ele, siga
  [Sem acelerômetro](docs/acelerometro.md#sem-acelerômetro).
- **Valores calibrados na impressora do autor** (PID, `z_offset`, pressure advance, input shaper): recalibre na
  sua.

Os arquivos da A1V1 e da A3V2 foram enviados por outras pessoas e não foram testados pelo autor.

> ⚠️ **Não configure o `max_power` da seção `[heater_bed]` com valor maior que 0.2.** Com 1.0, a mesa do autor
> esquentou rápido demais; por sorte, a resistência não queimou e o vidro não quebrou. A mesa precisa
> esquentar devagar para o calor se espalhar e minimizar deformações.

## Documentação

| Documento | Conteúdo |
|---|---|
| [docs/hardware.md](docs/hardware.md) | hardware original e modificado, firmware original, pinos, limites físicos, direção dos motores |
| [docs/instalacao.md](docs/instalacao.md) | Raspberry Pi OS, Klipper, Moonraker e Fluidd; configuração, primeiro teste e calibrações |
| [docs/firmware.md](docs/firmware.md) | compilar e gravar o Klipper na RAMPS e no Raspberry Pi |
| [docs/acelerometro.md](docs/acelerometro.md) | ADXL345, input shaper e o problema conhecido de escala |
| [docs/fatiador.md](docs/fatiador.md) | perfis do PrusaSlicer e como eles se integram ao cfg |
| [docs/backup_e_migracao.md](docs/backup_e_migracao.md) | backup e troca de cartão SD, mantendo o histórico de impressões |
| [docs/upgrades.md](docs/upgrades.md) | Klipper com Raspberry Pi, TMC2209, ventoinha Noctua, bico de 0,6 mm, ADXL345, placa Octopus |

Outras pastas:

- [`firmware_configs/`](firmware_configs/): configurações de compilação do firmware da RAMPS e do MCU Linux.
- [`prusaslicer/`](prusaslicer/): perfis de impressora, filamento e impressão.

## Roteiro resumido

1. Peça à GTMax3D o firmware original da sua impressora (arquivo `.hex`) **antes de mexer em qualquer coisa**.
   É com ele que se volta ao estado original.
2. Siga o [docs/instalacao.md](docs/instalacao.md) na ordem: Raspberry Pi (2B ou mais novo) com Klipper,
   Moonraker e Fluidd; cópia e ajuste do cfg (porta serial, acelerômetro, direção dos motores); gravação do
   firmware; primeiro teste de movimento; calibrações. Se o Pi não tiver Wi-Fi embutido, use um adaptador USB.
3. Importe os perfis do fatiador: [docs/fatiador.md](docs/fatiador.md).

## Vantagens do Klipper nesta impressora

O firmware original já é bem completo (ver [docs/hardware.md](docs/hardware.md#firmware-original)): é um
Marlin adaptado pela GTMax3D, com nivelamento automático da mesa pela sonda, troca de filamento (`M600`),
pausa pelo sensor de fim de filamento, ajuste de velocidade durante a impressão e estatísticas no display. O
que o Klipper acrescenta:

- Pressure advance e input shaper, que o firmware original não tem: melhor qualidade ou mais velocidade, sem
  as ondulações.
- Controle de temperatura muito melhor. Com o firmware original, quando a temperatura do bico baixava
  durante a impressão (por exemplo, depois de uma primeira camada mais quente), ela caía além do alvo antes
  de estabilizar, e às vezes a impressão falhava. Com o Klipper isso não acontece.
- Controle pela rede, pelo navegador, sem cartão de memória.
- Estimativa realista do tempo restante.
- Ajuste de velocidade, temperatura e fluxo durante a impressão, também pelo navegador.
- Arcos (G2/G3, como os do ArcWelder), que deixam o G-code menor e as curvas mais suaves. O firmware original
  também aceita esses comandos, mas fica muito lento ao executá-los.
- Histórico de cada impressão no navegador. O firmware original só mostra os totais no display.
- Macros editáveis no cfg, sem recompilar o firmware.
- Sem troca de componentes, dá para voltar ao firmware original gravando o `.hex` da GTMax3D. O autor nunca
  fez essa volta (ver [docs/firmware.md](docs/firmware.md)).
- Há muito material sobre o Klipper no YouTube.

## Desvantagens

- Exige conhecimentos básicos de Linux para instalar e manter o Raspberry Pi.
- A maior parte da documentação está em inglês.
- Uma direção de motor errada pode forçar a mecânica contra o fim de curso. Teste com cuidado, com a mão no
  interruptor liga/desliga.
- O cabo USB fica ligado o tempo todo, e o Raspberry Pi precisa de um lugar perto da impressora.
- Alguns comandos do firmware original não existem no Klipper e precisam ser recriados como macros. Os cfg
  deste repositório já trazem `G29` (nivelamento com a sonda), `M300` (bipe), `M600` (troca de filamento) e
  `M601` (pausa). O `M300`, o `M600`, o `M601` e o `CANCEL_PRINT` do cfg foram testados só em simulação,
  ainda não na impressora.
- Os menus do display mudam: passam a ser os do Klipper, em inglês.

## Créditos

- @mirandadam: configuração e documentação da Core A2V2.
- @zenaro147: configuração da Core A1V1 (janeiro de 2023).
- @J-Pozenato: configuração da Core A3V2 (julho de 2023).
