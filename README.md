# klipper_gtmax3d

Notas pessoais para instalação do firmware [Klipper](https://www.klipper3d.org/) e outros upgrades para a impressora Core A2V2 da GTMax3D.

*Não tenho qualquer relação comercial, patrocínio, vínculo, etc. com a empresa GTMax3D, exceto pelo fato de ter comprado uma impressora Core A2V2 no início de 2021. A empresa não divulga código fonte do firmware nem documentação específica para fazer esse tipo de alteração. Se você não quer perder a garantia, faz uso profissional da sua impressora ou não pode arriscar que ela fique sem condições de operação por algum tempo, não tente estas modificações.*

O suporte deles me apoiou com informações para fazer este upgrade e me respondeu super rápido sempre que precisei. 

O @zenaro147 enviou uma configuração exemplo para a Core A1V1 em 2023-01-03.

***Se você tiver alguma dúvida ou estiver faltando alguma informação importante, por favor abra um "issue" neste repositório.***


# Modalidade de upgrade

## Funcionalidade

A instalação do firmware Klipper é o upgrade que mais vale a pena, e pode ser feito sem abrir a impressora e sem trocar componentes.

O arquivo [printer-gtmax3d-core-a2v2.cfg](printer-gtmax3d-core-a2v2.cfg) neste repositório contém as informações necessárias, inclusive os pinos para detecção de filamento, sensores de fim de curso ("endstops"), configuração de geometria da impressora, modelo de LCD, etc. A direção dos motores configuradas nele respeita a direção original de fábrica. Se você trocar os controladores para os TMC2209, ou você precisa trocar a direção dos motores no printer.cfg (inclusive do extrusor) ou precisa trocar um par de fios de cada controlador. Ainda não fiz as macros de troca de filamento quando o filamento acaba no meio da impressão.

Existe uma versão do arquivo feito pelo @zenaro147 para a Core A1V1 - verifique a versão correta para a sua impressora, revise o que for necessário e copie o conteúdo para o arquivo printer.cfg do seu raspberry pi.

* Peça para a GTMax3D uma cópia do firmware original da impressora no formato ".hex" ANTES de você tentar mexer em qualquer coisa. Esse arquivo vai ser necessário se houver algum problema para carregar o firmware.
* Instale o [FluiddPi](https://docs.fluidd.xyz/installation/fluiddpi) em um minicomputador Raspberry PI 2B ou superior e faça as configurações como no tutorial do site.
* Instale um controlador wifi se o seu raspberry pi não tiver um embutido (no meu caso eu já tinha um Raspberry PI 2B antigo).
* Conecte raspberry pi pelo cabo USB à sua impressora.
* Acesse o Fluidd pela interface de rede (http://fluiddpi.local se você não mudou) e coloque o arquivo [printer.cfg](printer.cfg) na pasta de configuração.
* Não lembro se o Fluidd faz essa etapa sozinho, mas eu fiz manualmente o procedimento de [instalação do Klipper](https://www.klipper3d.org/Installation.html) para carregar o firmware na impressora.
* Faça os procedimentos em [Klipper Initial Setup : Making sure things are all good before printing](https://www.youtube.com/watch?v=T-knWbh1Gg8) para testar o movimento da impressora. Tudo já deve estar ok se você não trocou nenhum componente na impressora (ou se trocou e inverteu corretamente a direção dos motores), mas é importante conferir.
* Calibre a sua impressora. Normalmente só o "rotation_distance" já vai ser suficiente para você começar a imprimir, e o seu valor não deve ser muito diferente do que eu achei para a minha impressora.
* ***Atenção: NÃO CONFIGURE O max_power DA SESSÃO heater_bed PARA UM VALOR MAIOR QUE 0.2*** O arquivo printer.cfg já está com o valor correto, mexa por sua conta e risco! Eu comecei a configuração com 1.0 e a temperatura da mesa subiu super rápido. Tive sorte de não queimar a resistência nem quebrar o vidro. Esse é um componente que tem que esquentar devagar, para dar tempo do calor se espalhar e minimizar deformações.


Vantagens:
* Melhor qualidade de impressão ou maior velocidade por causa do melhor controle de temperatura e funcionalidades como avanço de pressão e "input shaping", para eliminar ondulações.
* Controlar a impressora pela rede.
* Configurar macros para facilitar algumas tarefas como carga e descarga de filamento, calibração de temperatura da mesa e do bico quando trocar o material, etc.
* Acompanhamento da impressão com estimativa realista de tempo para término.
* Alterar parâmetros de impressão enquanto ela acontece (velocidade, temperatura, extrusão, etc.).
* Mandar trabalhos para a impressora sem precisar de cartão de memória.
* Suporte para comandos G2 e G3 (Cura ArcWelder), que diminuem o tamanho do arquivo .gcode e fazem as curvas nos objetos ficarem mais suaves com menos "degraus". O firmware original suporta esses comandos mas a impressora fica super lenta na hora de executá-los.
* Se você não fizer nenhuma troca de componentes da impressora (upgrades abaixo), é possível voltar a impressora para a configuração de fábrica original.
* Tem bastante material no youtube ensinando a mexer com o Klipper.

Desvantagens:
* Exige conhecimentos básicos de informática para instalar o fluidpi em um minicomputador raspberry pi.
* A maioria da documentação está em inglês.
* Se você fizer alguma confusão com a troca de direção dos motores, a sua impressora pode tentar forçar um motor para além do fim de curso dele, podendo provocar todo tipo de problemas mecânicos. Teste com cuidado, com o dedo no interruptor liga/desliga, para poder interromper a energia instantaneamente se alguma coisa der errado.
* É bom dar uma olhada nos vídeos no youtube ensinando a instalar o klipper pela primeira vez.
* O cabo usb tem que ficar ligado o tempo todo e você tem que arrumar um lugar para colocar o raspberry pi que controla a impressora.
* Vários comandos de gcode, como o M300 que gera um bipe, param de funcionar pois não são suportados diretamente no Klipper. Você pode reimplementá-los como macros no arquivo printer.cfg, como o g29 que eu escrevi para nivelar a mesa tendo em vista as características da sonda.


## Ruído

Esse upgrade é caro mas vale muito a pena. A impressora fica muito mais silenciosa e a qualidade da impressão melhora um pouco pois os controladores são melhores.

* Mais importante: trocar os controladores de motor por 4x TMC2209 (não testei os TMC2208 mas devem funcionar bem também). Se você não mexer em mais nada, é necessário inverter dois fios de cada um dos controladores para inverter a direção do movimento. Custo: 250-300 reais no Brasil. Comprei os meus na China.
* Secundário: trocar a ventoinha da parte da frente por uma Noctua NF-A4x20 PWM 5V. Custo: ~R$130. Não acho que vale a pena comprar no exterior.

Vantagens:
* Mal dá para perceber se a impressora está imprimindo ou parada. A ventoinha externa ainda faz bastante barulho mas não incomoda tanto quanto os motores.
* Se trocar a ventoinha, melhora mais ainda. Dá até para trabalhar no mesmo ambiente em que a impressora está imprimindo.

Desvantagens:
* É necessário abrir a impressora, trocar componentes e mexer em alguns fios. Se você fizer este upgrade junto com a instalação do Klipper, a direção pode ser invertida no arquivo printer.cfg, sem a necessidade de trocar fios.
* O espaço é um pouco apertado e você tem oportunidade de queimar componentes se ligar alguma coisa no lugar errado ou colocar um controlador de cabeça para baixo.
* Risco de choque elétrico. DESLIGUE DA TOMADA antes de mexer.


## Eletrônica

Comprei o kit para fazer o upgrade da placa interna (uma RAMPS 1.4 com arduíno) para uma BigTreeTech Octopus mas acabei não fazendo a troca. Daria muito trabalho e avaliei que não vale a pena, pois, no meu caso, a única vantagem seria que a placa seria um pouco mais rápida, o que ajudaria em caso de projetos que exigissem impressão veloz com um gcode muito grande. Consegui fazer tudo com os componentes originais da impressora. Vou guardar a minha Octopus para um outro projeto.

Se você não quiser usar o RaspberryPi com o Klipper, trocar a placa interna por uma Octopus ou outra placa mais potente pode permitir que você use uma versão do Marlin 2.0 mais recente que suporte avanço de pressão. Na minha opinião dá muito mais trabalho e não deve ficar tão bom como usar o Klipper.
