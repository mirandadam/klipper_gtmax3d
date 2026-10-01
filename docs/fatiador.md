# Fatiador (PrusaSlicer)

O autor usa o PrusaSlicer. Os perfis estão na pasta [`prusaslicer/`](../prusaslicer/):

| Arquivo | Tipo | Conteúdo |
|---|---|---|
| [`impressora-gtmax3d-a2v2-bico0.6.ini`](../prusaslicer/impressora-gtmax3d-a2v2-bico0.6.ini) | impressora | mesa, bico 0.6, retração, G-code inicial e final |
| [`filamento-abs-gtmax3d-branco.ini`](../prusaslicer/filamento-abs-gtmax3d-branco.ini) | filamento | ABS GTMax3D branco |
| [`impressao-basico-0.2mm.ini`](../prusaslicer/impressao-basico-0.2mm.ini) | impressão | camadas de 0,2 mm, uso geral |

Para instalar, abra a pasta de configuração do PrusaSlicer (*Help > Show Configuration Folder*; em português,
*Ajuda > Mostrar pasta de config*) e copie cada arquivo para a subpasta do seu tipo: `printer/`, `filament/`
ou `print/`. Depois reinicie o PrusaSlicer. Cada perfil aparece com o nome do arquivo, sem o `.ini`. É nessas
pastas que o próprio PrusaSlicer guarda os perfis do usuário, e foi de lá que estes arquivos foram copiados;
os cabeçalhos dizem que foram gravados pelo PrusaSlicer 2.9.6.

Os perfis funcionam **em conjunto** com os arquivos `printer-gtmax3d-core-a2v2-*.cfg`: o G-code inicial
chama a macro `G29` e os comandos `M300`, `M600` e `M601` são macros definidas no cfg. Com outro cfg,
confira se essas macros existem.

Só há perfil de impressora para o bico de 0,6 mm. Com o bico de 0,4 mm, mude o diâmetro do bico
(`nozzle_diameter`) para 0.4 no perfil da impressora e salve-o com outro nome; as larguras de extrusão do
perfil de impressão, exceto a da primeira camada, são automáticas e acompanham o bico. O autor não usa estes
perfis com o bico de 0,4 mm.

## Perfil da impressora

- Tipo de G-code ("flavor"): **Klipper**.
- Mesa 220 × 220 mm, altura máxima 240 mm, bico de 0,6 mm.
- Retração: 2 mm a 40 mm/s, sem elevação do Z nos deslocamentos.
- Troca de cor: `M600`. Pausa: `M601`. Os dois pausam a impressão, afastam a cabeça de impressão e bipam.
  Troque o filamento com as macros `DESCARREGAR_FILAMENTO` e `CARREGAR_FILAMENTO` e continue com `RESUME`.
  O sensor de fim de filamento chama o mesmo `M600` quando o filamento acaba durante a impressão.
- Com a impressão já pausada, ou sem impressão em andamento, o `M600` só afasta a cabeça de impressão (se os
  eixos estiverem na origem) e bipa. Sem impressão, ele também descarta uma pausa que tenha sobrado.

### G-code inicial

```gcode
M106 S25                     ; ventoinha a 10 %
M190 S[first_layer_bed_temperature]   ; espera a mesa
M109 S[first_layer_temperature]       ; espera o bico
G29 V4 T                     ; home e malha da mesa (macro do cfg)
M400
G92 E0
G90
G0 X190 Y-5 F2000            ; vai para a área de purga, fora da mesa
G0 X40 Y-5 Z0.3 F2000
G92 E0
G1 E10.0 F200                ; purga no ar
G92 E0
G1 Y5 E2.0 F500              ; limpa na borda da mesa
G92 E0
G1 X140 E10 F1000            ; linha de purga na mesa
M400
G92 E0
G1 E-1 F2400                 ; recolhe 1 mm
G92 E0
```

A purga em Y = −5 só funciona porque o cfg permite Y até −9 (a área fora da mesa, na frente). Os parâmetros
`V4 T` do `G29` não têm efeito: a macro do Klipper os ignora.

Os comentários acima foram resumidos; o arquivo `.ini` tem o texto completo.

### G-code final

```gcode
M140 S0
M204 S4000                   ; devolve a aceleração máxima
M107
G91
G1 E-3 F2400                 ; recolhe 3 mm
G90
G1 X0
G1 Y100
G1 Z240                      ; desce a mesa até perto do fim
M104 S0
M140 S0
M84
M300 S4 P1000                ; bipe de 1 s (macro do cfg)
```

## Acelerações e velocidades

O PrusaSlicer grava comandos de aceleração (`M204`) no G-code, e o Klipper os aplica no lugar do
`max_accel` do cfg, mesmo quando passam dele. No perfil de impressão `impressao-basico-0.2mm`:

| Parâmetro | Valor |
|---|---|
| Aceleração padrão | 500 mm/s² |
| Deslocamentos (travel) | 2000 mm/s² |
| Demais (perímetros, preenchimento...) | 0, que quer dizer "usar a padrão" |

Com esse perfil a impressão roda a 500 mm/s², bem abaixo dos 4000 que o input shaper permite: é uma escolha
conservadora de qualidade. Sem acelerômetro, baixe o `M204 S4000` do G-code final (ele continua valendo
depois da impressão) e os 2000 mm/s² dos deslocamentos: ver
[Sem acelerômetro](acelerometro.md#sem-acelerômetro).

Quem limita a velocidade, na prática, é a vazão: os perfis de impressão e de filamento limitam a vazão a
5 mm³/s, o que, com o bico de 0,6 mm e camadas de 0,2 mm, dá uns 40 mm/s. As velocidades de 60 e 80 mm/s do
perfil quase não são usadas. Para imprimir mais rápido, aumente primeiro a vazão máxima, dentro do que o
hotend consegue derreter.

Os limites de máquina do perfil da impressora (1500 mm/s²) são usados só para estimar o tempo de impressão
(`machine_limits_usage = time_estimate_only`). O tempo real aparece no Fluidd durante a impressão.

## Filamento ABS GTMax3D branco

| Parâmetro | Valor |
|---|---|
| Temperatura do bico | 240 °C |
| Temperatura da mesa | 105 °C |
| Multiplicador de extrusão | 1 |
| Vazão volumétrica máxima | 5 mm³/s |
| Ventoinha | 100 %, só nas pontes e nas camadas que levam menos de 60 s; desligada nas 3 primeiras camadas |
| Camadas de menos de 10 s | a velocidade cai, até 5 mm/s no mínimo |

O cfg do bico de 0,6 mm foi calibrado com esse filamento a 228 °C (pressure advance 0,6); o perfil imprime a
240 °C, onde o pressure advance ideal pode ser um pouco diferente.

## Perfil de impressão `impressao-basico-0.2mm`

- Camadas de 0,2 mm (primeira de 0,3 mm); 2 perímetros; 5 camadas sólidas em cima e embaixo.
- Preenchimento gyroid a 15 %.
- Aba ("brim") de 5 mm, só por fora da peça.
- Velocidades: perímetros 60 mm/s (o externo a 25 %, 15 mm/s), preenchimento 80 mm/s, topo 15 mm/s, primeira
  camada 30 mm/s, deslocamentos 130 mm/s. O `max_velocity` do cfg é 100 mm/s, e o Klipper limita os
  deslocamentos a esse valor sem avisar.
- Passadas de alisamento ("ironing") nas superfícies do topo.
- Compensação de tamanho XY de −0,1 mm.

## Enviar direto para a impressora

O PrusaSlicer consegue mandar o arquivo direto para o Moonraker, sem baixar o G-code:

1. No perfil da impressora, clique no ícone de rede ("Adicionar impressora física").
2. Tipo de host: **OctoPrint**. Endereço: `http://impressora3d.local` (ou o IP fixo).
3. Deixe a chave de API em branco se o Moonraker aceitar a sua rede em `trusted_clients`. Com
   `force_logins: True`, use a chave de API do Moonraker, que o comando `~/moonraker/scripts/fetch-apikey.sh`
   mostra no Pi.

Isso depende da seção `[octoprint_compat]` no `moonraker.conf` (ver [instalacao.md](instalacao.md#4-moonraker)).
O autor não usa esse recurso e não o testou com esta impressora.
