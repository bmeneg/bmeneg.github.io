---
layout: post
title: Uma conversa entre Bootloader e o Kernel do Linux - Parte 0/3
subtitle: Uma série a respeito do reconhecimento dos hardwares personalizados no Linux Embarcado
---

Algo que sempre me interessou foi saber como funcionava o interior de um Bootloader e de um Kernel do Linux quando se
tratava do reconhecimento do _hardware_ das mais diferentes plataformas embarcadas que tem aparecido no mercado, desde
as mais simples até as mais robustas e com dezenas de periféricos. Essas plataformas são conhecidas como _Single
Board Computers_ (SBC) e constantemente aparecem novas informações sobre uma nova que foi lançada com um novo processador ou
com suporte a um novo tipo de periférico. Isso sem dúvida nos distrai por muitos minutos e muitas vezes nos deixa em
dúvida sobre qual adiquirir.

Porém, com tanta nova plataforma sendo lançadas nos leva a pensar que seu desenvolvimento é algo próximo de "fácil",
não é mesmo? É aí que as coisas ficam interessantes, como realmente funciona o reconhecimento de todas estas partes de
_hardware_ em cada uma das plataformas embarcadas no Linux Embarcado?

Este é o primeiro artigo de uma série que quero realizar até o dia em que irei ministrar uma palestra na conferência
TDC 2016, no dia 06/07/2016 em São Paulo a respeito exatamente deste assunto. Infelizmente lá não poderei abordar o
assunto com tantos detalhes, mas farei meu melhor para expôr a vocês por aqui os detalhes que muitas vezes não são
encontrados tão facilmente pela _internet_.

As partes serão dividas de acordo o assunto que nelas serão tratados, mas devido ao meu conhecimento não tão vasto
nesta área irei abordar basicamente como as coisas se desenrolaram para as SBCs que empregaram processadores de
arquitura ARM, a qual tem crescido incontestavelmente muito nos últimos anos e que cada vez mais seu suporte cresce na
_mainline_ de desenvolvimento do Kernel.

1. Histórico do suporte a _Single Board Computers_ no Linux Embarcado
2. Fundamentos de Bootloader e Kernel e seus papeis no reconhecimento do _hardware_
3. _Flattened Device Trees_ e similares

