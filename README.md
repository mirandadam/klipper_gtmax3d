# klipper_gtmax3d

Notas pessoais para instalação do firmware [Klipper](https://www.klipper3d.org/) e outros upgrades para a impressora Core A2V2 da GTMax3D.

*Não tenho qualquer relação comercial, patrocínio, vínculo, etc. com a empresa GTMax3D. A empresa não divulga código fonte do firmware nem documentação específica para fazer esse tipo de alteração. Se você não quer perder a garantia, faz uso profissional da sua impressora ou não pode arriscar que ela fique sem condições de operação por algum tempo, não tente estas modificações.*

O suporte deles me apoiou com informações para fazer este upgrade e me respondeu super rápido sempre que precisei. 

***Se você tiver alguma dúvida ou estiver faltando alguma informação importante, por favor abra um "issue" neste repositório.***


# Modalidade de upgrade

## Funcionalidade

Vantagens:
* 


## Ruído

Esse upgrade é caro mas vale muito a pena. A impressora fica muito mais silenciosa e a qualidade da impressão melhora um pouco pois os controladores são melhores.

* Mais importante: trocar os controladores de motor por 4x TMC2209 (não testei os TMC2208 mas devem funcionar bem também). Se você não mexer em mais nada, é necessário inverter dois fios de cada um dos controladores para inverter a direção do movimento. Custo: 250-300 reais no Brasil. Comprei os meus na China.
* Secundário: trocar a ventoinha da parte da frente por uma Noctua NF-A4x20 PWM 5V. Custo: ~R$130. Não acho que vale a pena comprar no exterior.

Vantagens:
* Não dá nem para perceber quando a impressora está imprimindo, de tão silenciosa que fica. A ventoinha externa ainda faz bastante barulho mas não incomoda tanto quanto os motores.
* Se trocar a ventoinha, melhora mais ainda. Dá até para trabalhar no mesmo ambiente em que a impressora está imprimindo.

Desvantagens:
* É necessário abrir a impressora, trocar componentes e mexer em alguns fios.
* O espaço é um pouco apertado e você tem oportunidade de queimar componentes se ligar alguma coisa no lugar errado ou colocar um controlador de cabeça para baixo.
* Risco de choque elétrico. DESLIGUE DA TOMADA antes de mexer.


## Eletrônica

Comprei o kit para fazer o upgrade da placa interna (uma RAMPS 1.4 com arduíno) para uma BigTreeTech Octopus mas acabei não fazendo. Não vale a pena. Consegui fazer tudo com os componentes originais da impressora.


