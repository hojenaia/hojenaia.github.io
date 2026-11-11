---
title: "Reduzindo a degradação em tomadas longas do Viggle Remake"
date: 2026-11-11T12:00:00-03:00
description: "Um teste de 1:23 no ComfyUI mostra como keystone e capstone fixos ajudam a preservar o fundo entre passagens do Viggle Remake."
summary: "Como a proteção de contexto combina uma sobreposição de 17 frames com referências visuais fixas para reduzir mudanças acumuladas em tomadas longas."
tags: [IA, ComfyUI, Viggle Remake, vídeo]
draft: false
---

## Reduzindo a degradação em tomadas longas do Viggle Remake

Ao encadear várias passagens do Viggle Remake, cada trecho usa os últimos 17 frames do anterior como contexto. Isso mantém a ação contínua, mas pequenas mudanças visuais podem se acumular. Em um vídeo de 1:23, notei esse efeito no tecido branco ao fundo: ele ficava menos nítido conforme as passagens avançavam.

A variante **Viggle Remake Long Video (Context Protection)** mantém a sobreposição de 17 frames e acrescenta referências visuais estáveis para ajudar a conter esse desvio.

## Como funciona a proteção de contexto

O nó guarda dois latentes da primeira passagem e os reutiliza nas passagens seguintes:

- O **keystone** ancora o início de cada nova janela em uma referência visual fixa.
- O **capstone** oferece uma segunda referência, próxima ao fim da primeira janela. Ele só entra no processo nas etapas finais de denoising — a técnica *Late Lock* — para reduzir a chance de puxar a geração para uma pose antiga.

O capstone permanece fixo; não é atualizado a cada passagem. Os latentes extras são removidos antes da montagem da saída, portanto não acrescentam frames ao vídeo final. A sobreposição original de 17 frames continua sendo usada.

## O que o teste mostrou

Comparei o vídeo de origem, a versão gerada pelo nó original e a versão com proteção de contexto, nos frames inicial e final de um teste de 1:23. Na inspeção visual, o fundo pareceu manter melhor sua aparência na versão protegida. Em um teste mais curto, a junção que apresentava um glitch de um frame também ficou limpa com o capstone fixo e o *Late Lock*.

Esses resultados são observações visuais de alguns testes, não uma medição objetiva. A técnica continua experimental, e vale revisar as emendas e os detalhes do fundo em cada vídeo.

**Frame inicial**

![Comparação do primeiro frame: origem à esquerda, nó original no centro e Context Protection à direita](https://raw.githubusercontent.com/hojenaia/viggle-remake-comfyui-long-take/main/images/comparison-first-frame.png)

**Frame final**

![Comparação do último frame: origem à esquerda, nó original no centro e Context Protection à direita](https://raw.githubusercontent.com/hojenaia/viggle-remake-comfyui-long-take/main/images/comparison-last-frame.png)

## Código e workflow

A implementação e o workflow de exemplo estão no repositório [viggle-remake-comfyui-long-take](https://github.com/hojenaia/viggle-remake-comfyui-long-take). A variante Context Protection é experimental e preserva o funcionamento do nó original como alternativa.
