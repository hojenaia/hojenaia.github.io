---
title: "Qwen3.8 Flash-Next numa RTX PRO 6000: os testes que faltavam"
date: 2026-11-04T12:00:00-03:00
description: "Medi Qwen3.8 Flash-Next Q2_0, IQ3_S e UD-Q4_K_XL no Basalt com uma RTX PRO 6000 Blackwell: velocidade, VRAM e resultados em testes de programação."
summary: "Na RTX PRO 6000 de 96 GB, comparei três quantizações do Qwen3.8 Flash-Next no Basalt. Q2_0 foi a mais rápida e todas passaram pelo hard canary; Q4 coube com folga, mas levou mais tempo nos testes de programação."
tags: [IA, benchmarks, hardware]
draft: false
---

## Antes dos números: modelos locais, Qwen e Basalt

Rodar uma IA local ficou bem mais interessante, mas ainda exige escolher entre modelos, quantizações, engines e hardware. Um serviço na nuvem costuma esconder quase tudo isso: você manda o prompt e recebe a resposta. Em casa, dá para controlar os dados, usar o modelo sem depender de uma API externa e experimentar sem pagar por token. Em troca, você precisa ter memória suficiente, configurar o servidor e aceitar que velocidade e qualidade dependem muito da combinação escolhida.

O modelo desta rodada é o **Qwen3.8 Flash-Next**, uma família aberta que se mostrou capaz de trabalhar como agente de programação nos testes anteriores do projeto. Ele usa uma arquitetura *mixture of experts* (MoE): embora tenha muitos parâmetros no total, só parte dos especialistas é ativada para cada token. Isso ajuda a reduzir o cálculo por token, mas não elimina a necessidade de armazenar ou movimentar os pesos. É por isso que quantização e memória disponível fazem tanta diferença.

Quantizar é representar os pesos com menos bits. Em geral, o arquivo fica menor e a inferência pode ficar mais rápida, mas há risco de perder precisão. Q2_0 usa uma representação mais compacta; IQ3_S gasta mais bits; UD-Q4_K_XL usa uma quantização ainda maior. Não existe uma resposta universal para “qual é melhor”: depende de quanto cabe na GPU, de como o engine distribui os dados e de a redução de precisão afetar ou não a tarefa real.

Para servir esses packs usei o **Basalt**, um engine local derivado do Strata e otimizado para GPUs NVIDIA Blackwell. Ele carrega o modelo, administra o que fica na VRAM e expõe uma API compatível com OpenAI para que aplicações como o OpenCode conversem com ele. O artigo original já tinha usado Basalt na RTX 5090. Aqui mantive esse engine e mudei a peça que faltava: uma RTX PRO 6000 com 96 GB de VRAM, capaz de testar também o pack Q4.

O [benchmark de LLMs locais do Fabio Akita](https://akitaonrails.com/2026/10/27/llm-benchmark-melhores-llms-open-source-hardware-caseiro/) comparou o Qwen3.8 Flash-Next em uma RTX 5090 e num Strix Halo. A conclusão prática foi interessante: no Basalt, a versão Q2_0 chegou perto de 400 tokens por segundo na 5090 e passou pelo teste difícil de programação. Mas ficou uma pergunta aberta: o que acontece com as versões maiores numa GPU com memória suficiente?

Eu tenho uma RTX PRO 6000 Blackwell com 96 GB de VRAM. Então preparei as configurações do Basalt, peguei os mesmos testes do repositório e rodei Q2_0, IQ3_S e UD-Q4_K_XL. A ideia não era inventar outra régua: era preencher a lacuna de hardware e quantizações que o artigo original deixou em aberto.

## O que foi testado

Usei os packs Basalt do Qwen3.8 Flash-Next em três quantizações:

| Quantização | O que esperar |
|---|---|
| Q2_0 | Menor e mais agressiva; tende a favorecer velocidade e espaço para o restante do modelo. |
| IQ3_S | Um pouco maior, como alternativa com mais precisão. |
| UD-Q4_K_XL | A maior das três; ocupa mais memória e, em princípio, preserva mais informação dos pesos. |

O servidor foi o Basalt, commit `2da2d3af`, com limite de potência da GPU em 450 W. Usei contexto máximo de 131.072 tokens, KV cache em `int8` e uma reserva de 4,5 GiB. A placa reportou 97.887 MiB de memória total.

O benchmark tem duas partes independentes: um teste curto de velocidade e um teste de programação em várias etapas. O teste de velocidade conversa diretamente com o servidor do modelo. O de programação usa o OpenCode como agente. Isso importa porque a versão do OpenCode não muda as medições de velocidade, mas pode afetar o tempo dos testes de código.

## O teste de velocidade

O runner envia três solicitações, cada uma pedindo até 256 tokens:

1. um prompt novo de aproximadamente 4 mil tokens;
2. um prompt novo longo, chamado de 32k no teste, que acabou tendo 42.400 tokens após tokenização e formatação;
3. uma continuação da conversa longa, para medir o efeito do reaproveitamento do KV cache.

Em cada etapa são registrados tokens de entrada por segundo, tokens gerados por segundo e o tempo até o primeiro token. O processo também acompanha VRAM livre, uso e temperatura da GPU, potência e memória do sistema. O número de “prompt tok/s” mede a leitura do contexto; “output tok/s” mede a geração sequencial depois que o modelo começa a responder. São fases diferentes e vale olhar as duas.

### Resultados de velocidade

| Quantização | Geração no prompt curto | Leitura do prompt longo | Geração no prompt longo | Primeiro token no longo | Menor VRAM livre |
|---|---:|---:|---:|---:|---:|
| Q2_0 | 483 tok/s | 9.176 tok/s | 471 tok/s | 4,62 s | 56,3 GiB |
| IQ3_S | 489 tok/s | 8.884 tok/s | 441 tok/s | 4,77 s | 40,1 GiB |
| UD-Q4_K_XL | 319 tok/s | 8.294 tok/s | 329 tok/s | 5,11 s | 15,1 GiB |

O Q2_0 foi o mais rápido na geração do prompt longo: 471 tokens por segundo. O IQ3_S ficou em 441, e o Q4 em 329. A diferença entre Q2_0 e Q4 é bem visível enquanto o modelo escreve, mas o tempo até começar a responder ficou relativamente próximo: de 4,62 a 5,11 segundos para esse prompt de 42,4 mil tokens.

O teste curto teve um resultado curioso: IQ3_S marcou 489 tokens por segundo, um pouco acima dos 483 do Q2_0. Em cargas tão curtas, diferenças pequenas podem refletir variação normal do teste; eu não usaria esses seis tokens por segundo para declarar um vencedor. No prompt longo, o Q2_0 liderou com mais folga.

Na continuação com cache, os modelos geraram a 479 tok/s (Q2_0), 434 tok/s (IQ3_S) e 346 tok/s (Q4). O primeiro token chegou em 0,06–0,07 segundo. Esse número de leitura de prompt fica enorme porque a conversa reaproveita tokens já processados. Não é uma comparação justa com o prompt frio; é uma demonstração de como o cache reduz a espera numa conversa em andamento.

O custo de memória muda bastante. O Q2_0 manteve 56,3 GiB livres no pior momento, IQ3_S 40,1 GiB e Q4 15,1 GiB. O Q4 coube e funcionou sem consumir toda a memória da placa, mas deixou muito menos margem para modelos adicionais, sessões paralelas ou outros processos na GPU.

### Mais VRAM não significa três vezes mais velocidade

A PRO 6000 tem três vezes a capacidade de memória da 5090, mas capacidade e velocidade são recursos diferentes. O ganho mais óbvio aqui foi poder carregar a quantização Q4 e ainda manter bastante VRAM livre. Isso também deixa espaço para manter mais especialistas e contexto na GPU, sem recorrer tanto à RAM do sistema.

Mas não dá para transformar esse resultado numa regra de “placa três vezes maior, modelo três vezes mais rápido”. A leitura do prompt depende de cálculo paralelo; a geração, token a token, costuma depender mais de quanto dado o modelo precisa movimentar a cada passo e da largura de banda disponível. A quantização também muda esse equilíbrio. Por isso eu medi o prefill e o decode em separado.

O Q2_0 leu o prompt longo a 9.176 tokens por segundo e o Q4 a 8.294. A diferença no prefill foi menor do que a diferença no decode, onde o Q2_0 gerou 471 tok/s contra 329 do Q4. E o tempo até o primeiro token ficou na mesma ordem de grandeza. Se o uso é uma conversa com respostas longas, decode pesa bastante. Se o agente recebe um repositório ou histórico extenso a cada etapa, tempo de leitura e cache podem ser igualmente importantes.

Também é bom separar “capacidade de contexto” de “velocidade em contexto”. O servidor foi configurado para até 131 mil tokens, mas isso não quer dizer que qualquer prompt desse tamanho vá ser processado rapidamente, nem que o modelo vá manter a mesma qualidade nesse limite. O teste frio que usamos tinha 42.400 tokens efetivos. É uma carga longa o bastante para mostrar a diferença de prefill, mas não valida sessões de 100 mil tokens.

### Como isso se compara à 5090

O artigo original mediu Q2_0 e IQ3_S no Basalt da RTX 5090. Lá, o Q2_0 ficou em 377–400 tok/s de geração e 8.573 tok/s de leitura do prompt longo; o IQ3_S ficou em 223–231 tok/s e 8.196 tok/s, respectivamente. Na PRO 6000, registramos 471/9.176 para Q2_0 e 441/8.884 para IQ3_S.

| Quantização | RTX 5090 no Basalt, artigo original | RTX PRO 6000 no Basalt, esta rodada |
|---|---:|---:|
| Q2_0, geração longa | 377–400 tok/s | 471 tok/s |
| Q2_0, leitura longa | 8.573 tok/s | 9.176 tok/s |
| IQ3_S, geração longa | 223–231 tok/s | 441 tok/s |
| IQ3_S, leitura longa | 8.196 tok/s | 8.884 tok/s |

Esses números dão contexto, mas não são um A/B perfeitamente controlado. São placas diferentes e as rodadas usaram commits e configurações de pack distintos. O que dá para afirmar com segurança é o resultado medido em cada máquina; não dá para atribuir toda a diferença somente à GPU, nem projetar que ela se repetirá em qualquer versão do Basalt. O caso do IQ3_S, em especial, tem um salto grande que merece ser repetido em condições exatamente iguais antes de virar uma conclusão geral.

O Q4 é o dado complementar mais direto: no artigo original, ele não chegou a completar os testes de qualidade na 5090 e no Strix. Aqui, na PRO 6000, rodou os três speed canaries e as três rodadas do hard canary. Isso responde à pergunta de cabimento e funcionamento nesta máquina. Não responde se o Q4 é melhor para todos os projetos.

### O que não medi

Os números são de uma única solicitação por vez. Não fiz teste de concorrência com múltiplas conversas, então não sei qual seria a vazão agregada do servidor nem como a latência por usuário mudaria com várias sessões. Também não medi consumo da workstation inteira na tomada ou custo por token; o runner coleta a potência reportada pela GPU, mas esta rodada não pretende ser uma análise de eficiência energética.

O teste de geração pede uma resposta curta e padronizada. Isso melhora a comparação entre as quantizações, mas não reproduz todas as características de um agente real: chamadas de ferramenta, respostas longas, anexos diferentes, prompts específicos de cada projeto ou períodos de ociosidade. O hard canary cobre parte desse uso de agente, mas usa uma única tarefa de programação e uma base pequena.

## O teste de programação

O hard canary não é só pedir ao modelo para escrever uma função. Ele recebe um projeto Python pequeno de conciliação de pagamentos e precisa desenvolvê-lo numa sessão contínua de quatro turnos do OpenCode. O projeto usa a biblioteca padrão do Python, `Decimal` para dinheiro e `unittest` para os testes; não há framework web ou dependências externas.

No primeiro turno, o modelo implementa uma especificação com cobranças, estornos, tarifas, moedas, datas e janelas de tempo. Parte das regras está escondida num histórico longo, com sugestões rejeitadas e uma decisão corrigida mais adiante. Entre as regras: IDs de conta não diferenciam maiúsculas de minúsculas; eventos de teste são ignorados, mas reservam seus IDs; tarifas só podem ser em USD; e contas internas `house_` não entram no total geral.

Depois, os pedidos mudam e se acumulam:

- estornos precisam encontrar a cobrança mesmo quando aparecem antes dela no arquivo; limites são avaliados pela ordem temporal;
- entram eventos de chargeback, com regras próprias e interação com os estornos;
- a CLI ganha uma opção para apresentar os saldos em outra moeda;
- a conversão de moeda é extraída para um módulo separado;
- o modelo precisa adicionar testes de regressão, preservar os testes fornecidos e revisar o código contra todos os requisitos anteriores.

Ao final, 36 testes ocultos verificam o resultado. O modelo não tem acesso a eles enquanto trabalha: o código roda num sandbox sem rede e com acesso limitado ao projeto. Os testes ocultos são montados somente na etapa de avaliação.

### Resultados do hard canary

Cada quantização foi rodada três vezes. Todas as nove execuções passaram nos 36 testes ocultos.

| Quantização | Pontuações | Tempo médio para os quatro turnos |
|---|---|---:|
| Q2_0 | 100, 100, 100 | 5,2 min |
| IQ3_S | 100, 96, 100 | 5,8 min |
| UD-Q4_K_XL | 100, 100, 100 | 12,5 min |

O 96 do segundo IQ3_S não veio de falha funcional: os 36 testes ocultos passaram. O auditor de dependências confundiu imports do pacote local `ledger` com uma dependência externa e descontou quatro pontos. As outras duas execuções IQ3_S não tiveram esse alerta.

O Q4 levou mais que o dobro do tempo médio do Q2_0 para concluir a tarefa, apesar de passar nos mesmos testes. Isso não significa que ele tenha necessariamente “pensado” por mais tempo: o wall time inclui geração, chamadas de ferramenta, execução de testes e as decisões do agente. O modelo também produz quantidades diferentes de texto entre execuções. Ainda assim, o resultado é prático: nesta tarefa, o Q4 não mostrou uma vantagem de qualidade que compensasse a velocidade menor.

Há uma ressalva de comparação. O artigo original usou OpenCode 2.1.4; para estas execuções na PRO 6000, usei OpenCode 2.2.0 porque era a versão disponível. Portanto, os tempos de programação não devem ser comparados diretamente com os do artigo. As medições de velocidade são independentes do OpenCode.

## O que esses números dizem — e o que não dizem

Para este modelo, esta máquina, o Basalt e estes prompts, a Q2_0 foi a combinação mais rápida no prompt longo e concluiu o hard canary mais depressa. A IQ3_S ficou perto em velocidade e também passou em todos os testes. O Q4 funcionou bem na PRO 6000, onde coube com folga operacional, mas foi mais lento e não demonstrou ganho de qualidade neste teste.

Isso não prova que quantização Q2 sempre seja suficiente, nem que Q4 seja inútil. O teste cobre um projeto Python pequeno, com contexto longo e requisitos que mudam; não cobre um repositório grande, contexto acima de 100 mil tokens, outros tipos de tarefa ou várias pessoas usando o servidor ao mesmo tempo. Também são apenas três repetições por quantização. Para uma decisão de compra ou para um fluxo de trabalho específico, eu ainda rodaria tarefas representativas do meu próprio uso.

O que a PRO 6000 acrescenta à discussão é mais simples: a Q4 que não cabia nos testes do artigo original em hardware doméstico pôde ser medida numa placa de 96 GB. Ela entrega 329 tokens por segundo no prompt longo, mas a Q2_0 chega a 471 e passou pelo mesmo conjunto de testes de programação. Ter mais VRAM abre espaço para testar modelos maiores; não garante que a quantização maior seja a melhor escolha.

Eu começaria pela Q2_0 para um agente local nesta configuração, porque foi rápida e não perdeu nenhum teste nesta amostra. Se aparecer um erro difícil de reproduzir, ou se o trabalho envolver um contexto muito maior, IQ3_S é um próximo passo razoável: custa velocidade em relação ao Q2_0, mas ficou próxima no benchmark e também passou nos testes. Eu escolheria Q4 quando houver uma razão concreta para usar essa quantização maior — por exemplo, uma avaliação própria mostrando que ela melhora o resultado no meu código — e não apenas porque a placa consegue carregá-la.

## Em resumo

- **Q2_0:** maior velocidade no prompt longo, 471 tok/s; média de 5,2 minutos no hard canary.
- **IQ3_S:** 441 tok/s e 5,8 minutos em média; passou nos 36 testes em todas as execuções.
- **UD-Q4_K_XL:** 329 tok/s e 12,5 minutos em média; passou nos 36 testes e teve mínimo de 15,1 GiB de VRAM livre.

Os arquivos de configuração e os resultados detalhados estão no [repositório do benchmark](https://github.com/akitaonrails/llm-opensource-benchmark), e as alterações foram propostas no [PR #1](https://github.com/akitaonrails/llm-opensource-benchmark/pull/1). O artigo original continua sendo a referência para os resultados da RTX 5090 e do Strix Halo; este texto documenta a rodada complementar na RTX PRO 6000.
