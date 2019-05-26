---
title: "Limitações e pitfalls de memórias NAND (PT-BR only)"
author: "Bruno Meneguele"
date: 2016-05-11
---

O suporte às memórias NAND na árvore principal do Kernel do Linux se mostram
bastante maduras para o uso do total potencial destas memórias, porém, mesmo o
suporte sendo bastante robusto e completo alguns problemas relacionados ao
_chip_/arquitetura das memórias tem trazido vários problemas às aplicações,
principalmente àquelas que rodam sobre _Single Board Computers_, como
Cubieboard/truck ou qualquer outra que possuí memória não volátil deste tipo, e
que geralmente não estão alinhadas diretamente à linha principal do Kernel
possuindo uma versão mais antigas com adaptações (_patches_) específicas para
aquela SBC.

As memórias NAND são dividas em algumas categorias, as quais surgiram em tempos
diferentes: NAND _Single-Level Cells_ (SLC), _Multi-Level Cells_ (MLC) e
_Triple-Level Cells_ (TLC). Os problemas encontrados aos chips da categoria MLC
se aplicam igualmente ou até mesmo se agravam nos TLCs, portanto qualquer
referência feita à MLC implicará à TLC também.

Estas categorias dizem respeito a quantidade de níveis (de tensão) que cada
célula de memória pode assumir, que está diretamente relacionada a quantidade
de bits que cada célula de memória armazena. Mas para entender isso melhor
vamos primeiro explicar um pouco da arquitetura da memória NAND.

## Tecnologia e arquitetura NAND

Na memória NAND os bits são codificados por níveis de tensão, os quais são
todos iniciados em **1** e a programação (operação PROGRAM) desta memória
implica na mudança desse estado para **0** e o caminho inverso, de **0** para
**1**, é feito pela operação ERASE. As operações possíveis sobre esta memória
não agem sobre cada bit, mas sim sobre uma quantidade de memória fixa e
diferente para cada operação, sendo estas quantidades:

* PAGE: unidade mínima para a operação PROGRAM;
* BLOCK: unicadade mínimo para a operação ERASE.

<center>![Arquitetura interna NAND](/imgs/nand-arch.png)</center>

Na imagem é possível observar quão lenta é a operação de ERASE em comparação
com a PROGRAM, sendo assim, é importante realizar o máximos de PROGRAMs
possível, antes de realizar um ERASE.

Cada bit (ou bits dependendo da categoria da memória) é armazenado em uma
célula da memória, sendo que a quantidade de bits presente na célular varia com
a categoria da NAND e seu estado muda de acordo a tensão aplicada. A figura
seguinte mostra este esquema que parece ser um tanto estranho quando comparado
com os tipos convencionais de memória.

<center>![Células NAND](/imgs/nand-cell.png)</center>

## Os problemas

O problema surge justamente neste ponto: quando *uma célula passa a representar
o estado de dois ou mais bits*. Isto pode parecer muito bom, já que a
quantidade de memória pode ser dobrada (MLC), triplicada (TLC), sem a mudança
do tamanho físico da memória, mas algo que não é muito levado em conta são os
problemas para estabelecer esta quantidade de subníveis de tensão. 

Considerando que as memórias NAND são alimentadas a um nível de 3.3V,
estabelecer 4 subníveis ou até mesmo 8 (no caso de TLC) passa a ser um desafio,
pois quanto maior o número de níveis, menor a distância dos limites de tensão
para cada nível. Contando que as memórias NAND possuem um desgaste natural a
cada operação feita (tanto na escrita como na leitura) a instabilidade do nível
de tensão aumenta conforme o tempo de uso passa. O problema de _bit flip_ passa
a acontecer com maior frequência quanto maior o número de níveis de tensão por
célula, pois qualquer pequena variação pode causar este _flip_, ainda mais
quando a memória esta desgastada. Outro problema é a retensão dos dados, a qual
varia gravemente de acordo a temperatura do ambiente e também pela leitura e
escrita de dados adjacentes, sendo este último chamado de _operation
disturbance_.

Infelizmente, _operation disturbance_ pode ocorrer em qualquer categoria de
memórias NAND, mas se mostra muito mais presente nas memórias de células
multinível (MLC e TLC). A figura a seguir exemplifica de maneira simples o que
é este problema. E de forma resumida pode ser dito que isso ocorre pois a
mudança de tensão em um ponto gera pequenas variações de tensão por pequeno
período de tempo nas regiões adjacentes à modificada.

<center>
![Disturbio causado em regiões vizinhas](/imgs/nand-operation-disturbance.png)
</center>

Outro problema muito sério quanto a isso é que com a forma como estas memórias
foram projetadas pode ocorrer que diferentes bits dentro de uma mesma célula
pertencer a PAGEs diferentes. Desta forma um único erro (através de _bit flip_
por exemplo) pode forçar ao ECC (_Error Correction Code_) a realizar mais
operações sobre regiões variadas a memória, desgastando ainda mais a memória.
Além disso, como é possível que dados em diferentes blocos sejam corrompidos de
uma única vez é possível também que diferentes arquivos em um sistema de
arquivo seja corrompido pela falha de uma única célula, gerando erros
totalmente não relacionados.

<center>![Páginas pareadas](/imgs/nand-paired-pages.png)</center>

Sendo assim, além do gerenciamento de escrita e leitura em excesso na memória
ser extremamente necessário, tanto pela aplicação como pelo sistema de
arquivos, deve-se levar em conta estas peculariedades da memória. No entanto os
produtores dos chips NAND desconsideram muitas questões que seriam extremamente
úteis e necessárias aos desenvolvedores de drivers para lidar com estes
problemas. Como exemplo de questões que poderiam ajudar: apresentar uma
descrição de como alterar (se possível) os intervalos de tensão que definem os
níveis da célula em caso de desgaste de memória, a descrição do que ocorre
quando a energia é rompida no meio da alguma operação e a elaboração de um
padrão entre fabricantes, a qual não existe ainda.

## Possíveis soluções

Para alguns destes problemas há soluções, porém, muitas delas são meramente
teóricas e de difícil implementação e, consequentemente, teste, estando
diretamente relacionadas aos drivers e aos sistemas de arquivos específicos
para memórias NAND, os quais devem ser utilizados **sempre** que possível, pois
o uso de um sistema de arquivo não preparado para esse tipo de memória (como o
EXT3 ou 4) diminuirá drasticamente a vida útil desta. Mas também é importante
observar que mesmo estes sistemas de arquivos, sendo mais recomendados que os
tradicionais, ainda não se mostram totalmente preparados para lidar com
memórias de células multiníveis, como é possível ver neste
[link](http://www.linux-mtd.infradead.org/doc/ubifs.html#L_ubifs_mlc) (obrigado
[Sérgio Prado](http://sergioprado.org/) por esta informação adicional).

Embora constantes estudos e desenvolvimentos estão sendo feitos sobre possíveis
soluções, ainda é preferível evitar o uso das **memórias multi-nivel** como
fonte **principal** de armazenamento, pois muitas das soluções conhecidas
limitariam o uso da memória, algumas em relação ao espaço lógico disponível e
outras em relação ao desempenho. Sendo assim, se for necessário utilizar uma
memória NAND como armazenamento principal para seu sistema, escolha o uso de
NANDs SLC se possível. 

## Comentários finais

Este problema enfrentei e ainda enfrento, principalmente por causa da SBC que
foi escolhida para a solução de onde trabalho e o requisito do sistema
operacional utilizar a memória NAND como armazenamento principal, e que
eventualmente, depois certo tempo de uso, vários problemas de _bad blocks_
surgem por todo o sistema. Na escolha da SBC não foi levada em consideração
estes possíveis problemas de memória, por isso fiquem sempre atentos nestas
questões para futuras decisões! Verifiquem como está o suporte para tal no
Kernel considerado estável para seu SBC.

Vou ficando por aqui sobre este assunto. Qualquer dúvida ou crítica só enviar
um comentário ou entrar em contato em minhas redes sociais.

Até mais.
