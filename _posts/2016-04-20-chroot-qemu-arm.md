---
layout: post
title: Chroot e QEMU para ARM em x86 
subtitle: Usando chroot e QEMU para desenvolver para ARM em outra arquitetura
---

De forma breve tentarei contextualizar a necessidade que tive para chegar a esta
técnica que irei apresentar, que sem dúvida, quando descobri, fiquei
absurdamente impressionado.

## A necessidade 

Como todos vocês devem saber (baste dar uma lida rápida [sobre
mim](/sobre)) trabalho com Linux Embarcado, principalmente sobre
plataformas ARM, e de tempos em tempos sou responsável por gerar a imagem do
sistema completo (bootloader + kernel + rootfs + aplicações da solução
proprietária) de uma forma que facilite no ambiente de produção. Cada um desses
componentes foram personalisados separadamente para a solução e no final tudo
que se espera é uma imagem **.img**, **.iso**, qualquer coisa assim. 

No momento era necessário aplicar algumas pequenas mudanças nos pacotes do
sistema (rootfs) da imagem, que influenciariam diretamente o funcionamento das
aplicações proprietárias. Sendo assim, a forma básica que trabalhávamos era
colocar esta imagem na plataforma ARM, fazer as mudanças necessárias e
recompactar o rootfs inteiro a fim de regerar a imagem final. Isto era feito
pois o Kernel modificado que estava utilizando não estava preparado (versão,
módulos, entre outras coisas) para rodar sobre uma virtualização com QEMU + KVM,
por exemplo. Com certeza melhorias poderiam ser feitas para habilitar este
suporte, mas por uma questão de tempo as coisas não correm sempre como o ideal
né? 

Este processo de gravar a imagem, modificar o que for necessário quando aos
pacotes do sistema e compactar (fazer um _backup_) do sistema inteiro levava um
bom tempo (considerando as mudanças), ainda mais contando que a plataforma
utilizada não possuí um grande poder de processamento.

Foi quando fui atrás de achar alguma técnica mais viável que me permitiria
modificar o sistema por completo mesmo não sendo da mesma arquitetura, algo como
o **chroot** faz, misturado com o suporte das diferentes arquiteturas do 
**QEMU**.

## A descoberta

Foi aí que descobri como utilizar exatamente estas duas ferramentas juntas
através de uma pitadinha de _mágica_ que as entranhas do kernel as vezes nos
enconde.

Utilizando um módulo do chamado **binfmt_misc** é possível você registrar um
interpretador específico para um determinado tipo de arquitetura quando
necessário. Em outras palavras, você consegue dizer ao kernel qual executável
ele deve usar para interpretar outro que foi compilado para uma arquitura
específica X. Mas vamos com calma.

Para começar, irei supor que quero criar uma imagem de um sistema de arquivos
ARM para depois poder utilizar na plataforma ARM.

```
$ dd of=arm-rootfs.img bs=1 seek=4G count=0
# mkfs.ext4 -F arm-rootfs.img
```

Após isso apenas monte esta imagem em qualquer local.

```
$ mkdir arm-chroot
# mount -o loop arm-rootfs arm-chroot/
```

Com um RootFS de alguma distribuição ARM em mãos (por exemplo este do [ArchLinux
ARM](br2.mirror.archlinuxarm.org/os/ArchLinuxARM-armv7-latest.tar.gz)),
descompacte dentro da pasta que foi criada e que será onde acessaremos como
chroot.

```
# tar -zxpf ArchLinuxArm-armv7-latest.tar.gz -C arm-chroot
```

Caso demorar muito (o que não deve acontecer) pode utilizar uma descompactação
paralela, porém é necessário possuir alguma aplicação com tal suporte instalado,
como o **pigz**.

```
# tar -xpf ArchLinuxArm-sun7i-latest.tar.gz -C arm-chroot \
    --use-compress-program=pigz -C arm-chroot
```

A partir de agora é onde a _mágica_ acontece: baixe a versão estática do binário
do qemu-user [aqui](https://packages.debian.org/sid/qemu-user-static) e, caso 
você não utilizar a distribuição Debian ou alguma outra derivada deste, será 
necessário converter o pacote. Para isso pode ser utilizado a aplicação "alien",
a qual provavelmente deve existir no repositório da sua distribuição.

Uma vez instalado no sistema o **qemu-arm-static**, faça uma cópia do mesmo para
o sistema ARM:

```
# cp /usr/bin/qemu-arm-static arm-chroot/usr/bin/qemu-arm-static
```

Feito isso, está na hora de definir este como interpretador de executáveis que 
foram compilados para ARM e que estão presentes sobre o seu Linux Kernel para 
x86. Para fazer isso é necessário você possuir o módulo **binfmt_misc** 
devidamente habilitado em seu kernel.

```
# zcat /proc/config.gz | grep -i binfmt_misc
CONFIG_BINFMT_MISC=y
```

Certifíque-se que o mesmo foi montado.

```
# ls /proc/sys/fs/binfmt_misc/
```

Caso não, monte-o com:

```
# mount binfmt_misc -t binfmt_misc /proc/sys/fs/binfmt_misc 
```

Como já dito anteriormente, com este suporte é possível registrar um
interpretador específico para executar/interpretar os binários de outras 
plataformas, como ARMv7. O comando para registrar é um tanto diferente:

```
# echo ':arm:M::\x7fELF\x01\x01\x01\x00\x00\x00\x00\x00\x00\x00\x00\x00\x02\x00\x28\x00:\xff\xff\xff\xff\xff\xff\xff\x00\xff\xff\xff\xff\xff\xff\xff\xff\xfe\xff\xff\xff:/usr/bin/qemu-arm-static:' > /proc/sys/fs/binfmt_misc/register
```

Antes de continuar, vamos entender o que é tudo isso. Segundo a documentação do 
módulo [binfmt_misc](https://www.kernel.org/doc/Documentation/binfmt_misc.txt) 
temos a seguinte informação:

> Binfmt_misc recognises the binary-type by matching some bytes at the beginning
  of the file with a magic byte sequence (masking out specified bits) you have 
  supplied.  

e também:

> To actually register a new binary type, you have to set up a string looking 
  like :name:type:offset:magic:mask:interpreter:flags (where you can choose the 
  ':' upon your needs) and echo it to /proc/sys/fs/binfmt_misc/register.

Ou seja, cada campo separado por ':' possui um significado próprio, sendo estes:

```
- name: nome que será dado ao registro no kernel, para nós, 'arm';
- type: o tipo de dados que será usado para identificar o executável, para nós, 
  'M' de 'magic';
- offset: distância em bytes que o valor mágico estará do início do cabeçalho do 
  executável, para nós foi omitido que significa um offset de 0;
- magic: são os valores exatos que devem existir nos cabeçalhos dos executáveis 
  a serem interpretados, no nosso caso os executáveis devem ser no formato ELF
  para plataforma ARM de 32 bits. A definição desta restrição está neste 18 
  bytes apresentados;
- mask: define os bytes que devem ser considerados (0xFF) ou ignorados (0x00) 
  na hora de ler o cabeçalho do executável;
- interpreter: caminho absoluto para o interpretador;
- flags: informações específicas que podem ser passadas para o interpretador na
  hora da execução.
```

Para maiores detalhes sobre o valor mágico passado visite este 
[link](https://en.wikipedia.org/wiki/Executable_and_Linkable_Format). Apenas para 
facilitar, os primeiros bytes **\x7fELF\x01** indica ser um arquivo com formato
ELF de 32 bits, já em **\x02\x00\x28\x00** quer dizer que é um executável para
plataforma ARM, ou seja, todos arquivos _executáveis, com formato ELF de 32 
bits e compilados para ARM_ devem ser executados pelo **interpreter**.

Sendo assim, o sistema ARM em arm-chroot/ já está pronto para ser utilizado,
agora basta fazer o processo comum do chroot.

```
# mount -t proc /proc arm-chroot/proc
# mount -o bind /dev arm-chroot/dev
# mount -o bind /dev/pts arm-chroot/dev/pts
# mount -o bind /sys arm-chroot/sys
# mount -o bind /run arm-chroot/run
# chroot arm-chroot/ /bin/bash
```

Uma vez dentro, você pode executar os comandos normais da distribuição referente
ao rootfs utilizado. No meu caso, utilizando o rootfs do ArchLinux ARM, posso 
executar o **pacman -S** para realizar instalações de pacotes do sistema, que 
irão afetar apenas os arquivos do sistema dentro de arm-chroot/.

Quando finalizado, você pode recompactar o rootfs em um .tar.gz ou então apenas 
desmontar a imagem .img (e os binds feitos) e utilizá-la em uma plataforma ARM 
real. 

Assim é como efetuo alterações de forma simples e rápida em minhas tarefas
do dia a dia.

Qualquer dúvida ou crítica, mandem um comentário!  
Valew a todos e até o próximo post! \o.
