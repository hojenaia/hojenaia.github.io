---
title: "Mais de 345 frames no Viggle Remake: um nó de long video para ComfyUI"
date: 2026-11-09T12:00:00-03:00
description: "Como adaptei o workflow do Viggle Remake no ComfyUI para renderizar vídeos longos em janelas sobrepostas, com um nó customizado e testes visuais."
summary: "O nó de long video divide o clipe em janelas de até 345 frames e usa 17 frames de sobreposição para continuar a geração. Este é o método, os testes e os limites que encontrei."
tags: [IA, ComfyUI, Viggle Remake, vídeo]
draft: false
---

## Mais de 345 frames no Viggle Remake: um nó de long video para ComfyUI

O Viggle Remake funciona bem em tomadas curtas, mas o fluxo normal do ComfyUI esbarra em um limite prático: até 345 frames por render. A 24 fps, isso dá pouco mais de 14 segundos. Para trabalhar com vídeos mais longos, adaptei o workflow para renderizar em passagens sobrepostas e reuni-las em uma única saída.

O resultado é um nó customizado de long video para ComfyUI. Testei o workflow com vários clipes convertidos para 24 fps. As renderizações concluídas ficaram visualmente boas, mas a continuidade ainda precisa ser conferida em cada emenda. Não medi qualidade com métricas automáticas; a avaliação foi visual.

## Como as passagens se conectam

O nó recebe o vídeo inteiro e o divide em janelas que respeitam o limite do modelo. O tamanho padrão é 294 frames, ou 12,25 segundos a 24 fps. É possível aumentar até 345 frames se houver memória suficiente na GPU.

Cada passagem seguinte começa com os últimos 17 frames gerados na passagem anterior. O nó codifica esse trecho como latentes e o mantém fixo durante a nova geração. Depois, remove os frames sobrepostos ao juntar os resultados. Assim, o limite de frames se aplica a cada render, e não à duração total do vídeo.

Com a janela padrão, a primeira passagem pode gerar até 12,25 segundos; cada passagem seguinte acrescenta até 277 frames novos, cerca de 11,54 segundos. Um vídeo maior que 345 frames, portanto, pode resultar de várias passagens encadeadas.

O áudio também é dividido pelos mesmos intervalos e reunido na saída. Se nenhum áudio for conectado, a saída fica silenciosa.

## O que os testes mostraram

Usei o nó em clipes de dança com movimentos variados e diferentes resoluções, todos preparados a 24 fps. Uma renderização de várias passagens terminou corretamente, e os resultados que revisei pareceram bons. Isso mostrou que o workflow consegue ultrapassar o limite de uma única janela sem exigir que o vídeo seja encurtado antes da geração.

A inspeção visual continua sendo parte do processo. Vale conferir os momentos em que uma passagem se conecta à seguinte e procurar mudanças na identidade da personagem, na roupa, no movimento ou no enquadramento. A sobreposição ajuda o modelo a continuar a ação, mas não garante que todos os detalhes permaneçam estáveis.

## O que o nó ainda não resolve

O nó foi feito para uma tomada contínua. Ele não detecta cortes, não reinicia a geração quando a câmera muda de plano e não reescreve prompts com timestamps para cada janela. Por isso, tenho usado prompts que descrevem a ação sem horários absolutos e tenho evitado clipes com cortes bruscos.

O vídeo precisa estar a 24 fps. O valor padrão de 294 frames é um ponto de partida; janelas maiores podem exigir mais VRAM. E, mesmo quando a renderização termina sem erro, isso não substitui a revisão das emendas.

## O pacote no GitHub

Separei o nó de long video do pacote original do Viggle Remake. O repositório [viggle-remake-comfyui-long-take](https://github.com/hojenaia/viggle-remake-comfyui-long-take) registra apenas o nó **Viggle Remake Long Video** e inclui um workflow de exemplo. Os nós originais e os helpers de condicionamento continuam vindo do pacote oficial `ComfyUI-Viggle-Remake`, que precisa estar instalado à parte.

O repositório não inclui pesos, LoRA, vídeos de origem nem imagens de referência. A ideia é acrescentar suporte a tomadas longas ao workflow existente, mantendo cada render dentro do limite do modelo.
