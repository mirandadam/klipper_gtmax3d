# Upgrades de hardware

Experiência do autor com modificações na Core A2V2. Os preços são de 2021.

## Klipper com Raspberry Pi

É o upgrade que mais vale a pena. Não exige abrir a impressora nem trocar componentes: basta um Raspberry Pi
(2B ou mais novo) ligado à placa original pelo cabo USB. Ver o [README](../README.md) e
[instalacao.md](instalacao.md).

## Ruído: drivers TMC2209 e ventoinha Noctua

Caro, mas vale muito a pena. A impressora fica muito mais silenciosa e a qualidade melhora um pouco, porque
os drivers são melhores.

- **Mais importante:** trocar os 4 drivers dos motores por TMC2209. O TMC2208 também deve funcionar (a
  própria GTMax3D o vende como opção silenciosa), mas o autor não testou. Custo: R$ 250 a 300 no Brasil; o
  autor comprou os seus na China.
- **Secundário:** trocar a ventoinha da frente por uma Noctua NF-A4x20 PWM 5V. Custo: ~R$ 130. Não vale a
  pena importar.

Depois da troca, mal dá para perceber se a impressora está imprimindo ou parada. Os motores eram o que mais
incomodava. Com os drivers novos e a Noctua, dá para trabalhar no mesmo ambiente.

### Cuidados na troca dos drivers

- **DESLIGUE DA TOMADA** antes de mexer. Há risco de choque elétrico.
- O espaço é apertado. Um driver encaixado de cabeça para baixo ou deslocado de uma fileira queima o driver
  e pode danificar a placa. Confira a posição dos pinos EN, GND e VM antes de ligar.
- O TMC2209 gira no sentido oposto ao A4988, o driver original provável. Troque de posição os dois fios de
  uma das bobinas de cada motor, como o autor fez, ou inverta a direção no `printer.cfg`: ver
  [Direção dos motores](hardware.md#direção-dos-motores).
- O autor nunca ajustou a corrente dos TMC2209 (ver [hardware.md](hardware.md#tmc2209-na-impressora-do-autor)).
  Para quem precisar, a página do TMC2208 no site da GTMax3D orienta medir o Vref entre o GND e o centro do
  potenciômetro, com a fonte ligada e os motores desconectados (desconecte-os com tudo desligado).

## Bico de 0,6 mm

O autor trocou o bico original, de 0,4 mm, por um de 0,6 mm, mantendo o hotend original. Em geral, um bico
maior imprime mais rápido e com paredes mais resistentes, em troca de menos detalhe fino. A troca exige mudar
o `nozzle_diameter`, recalibrar o `z_offset` e o pressure advance, e ajustar o fatiador. Os dois arquivos de
configuração (`-bico0.4` e `-bico0.6`) mostram as diferenças.

## Acelerômetro ADXL345

Um módulo barato que mede as ressonâncias para configurar o input shaper do Klipper, que por sua vez
permite acelerações bem maiores sem ondulações. Ver [acelerometro.md](acelerometro.md).

## Troca da placa (não feita)

O autor comprou um kit para trocar a placa interna (a RAMPS 1.4 com Arduino) por uma BigTreeTech Octopus,
mas desistiu. Daria muito trabalho, e a única vantagem seria uma placa mais rápida, que só faria diferença em
impressões muito velozes com G-code muito grande. Tudo funcionou com os componentes originais.

O firmware original é um Marlin sem linear advance (o equivalente ao pressure advance), e o código dele não
é divulgado. Sem o Raspberry Pi, a alternativa seria compilar um Marlin recente, com linear advance, para a
RAMPS ou para uma placa mais potente. Na opinião do autor, isso dá mais trabalho e o resultado não fica tão
bom quanto com o Klipper.
