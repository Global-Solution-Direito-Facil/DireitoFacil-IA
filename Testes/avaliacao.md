# Testes e avaliação da IA

Os testes foram feitos em **19/11/2026 pelo app Mobile** (emulador Android), com a conta de demonstração da Ana e arquivos **fictícios** de [`dados/`](dados/), passando pela solução publicada: API .NET (Azure) → RabbitMQ → API Java (Render) → Gemini. Cada caso tem um print tirado direto do emulador, em [`prints/`](prints/).

Este arquivo tem três partes: os [casos de teste](#1-casos-de-teste), os [resultados](#2-resultados) e os [critérios e a análise](#3-critérios-e-análise).

## 1. Casos de teste

**8 casos**: 2 entradas representativas e 6 situações de falha.

| Função | Entradas representativas | Situações de falha | Total |
|---|---|---|---|
| Leitura da receita | 1 (R01) | 2 (R02, R05) | 3 |
| Leitura do laudo | — | 4 (N02, N03, N04, L11) | 4 |
| Passo a passo com RAG | 1 (G03) | — | 1 |
| **Total** | **2** | **6** | **8** |

| Id | Função | Tipo | Entrada | O que testa | Esperado |
|---|---|---|---|---|---|
| R01 | receita | representativa | [`R01_genericos_png.png`](dados/receitas/R01_genericos_png.png) | receita impressa com três genéricos | Losartana potássica 50 mg, Metformina 850 mg e Sinvastatina 20 mg, com o princípio ativo igual ao nome |
| R02 | receita | falha | [`R02_comerciais_png.png`](dados/receitas/R02_comerciais_png.png) | nomes comerciais **sem** o princípio ativo escrito | Glifage XR 500 mg, Puran T4 50 mcg e Selozok 25 mg, com o princípio ativo **vazio** (a IA não pode completar de memória) |
| R05 | receita | falha | [`R05_laudo_como_receita.png`](dados/receitas/R05_laudo_como_receita.png) | um laudo enviado no lugar da receita | recusa: nenhum remédio lido |
| N02 | laudo | falha | [`N02_conta_de_luz.png`](dados/nao-documentos/N02_conta_de_luz.png) | conta de luz enviada como laudo médico | recusa: nada gravado, nenhum direito aberto |
| N03 | laudo | falha | [`N03_pagina_em_branco.png`](dados/nao-documentos/N03_pagina_em_branco.png) | página em branco enviada como laudo | recusa |
| N04 | laudo | falha | [`N04_ruido.png`](dados/nao-documentos/N04_ruido.png) | imagem só com ruído enviada como laudo | recusa |
| L11 | laudo | falha | [`L11_tea_sem_data.png`](dados/laudos/L11_tea_sem_data.png) | laudo de TEA legível, mas **sem data de emissão** | recusa: a IA transcreve, e a conferência do Java não grava laudo sem data |
| G03 | RAG | representativa | direito Passe Livre Interestadual – PcD do Lucas | passo a passo gerado com a base de conhecimento | de 3 a 7 passos curtos, começando com verbo, com a fonte do trecho usado (Lei 8.899/1994) |

### Como os testes foram feitos

1. **Ambiente:** o app Mobile rodando no emulador Android (Expo Go), configurado para as APIs publicadas (`EXPO_PUBLIC_DATA_SOURCE=api`). Login com a conta de demonstração da Ana, responsável pelo Lucas.
2. **Arquivos:** os arquivos de [`dados/`](dados/) foram copiados para a galeria do emulador (`adb push`), sem nenhuma alteração. Os arquivos enviados são idênticos aos do repositório (conferido por MD5).
3. **Receita (R01, R02, R05):** tela **"Minha receita"** → beneficiário Lucas → **Galeria** → foto do caso → marcar o consentimento → **"Ler receita"**. A leitura é na hora, e o app mostra a tela "Confira a leitura" (ou a recusa). No R01 o teste seguiu até **"Consultar Farmácia Popular"**.
4. **Laudo (N02, N03, N04, L11):** aba **Documentos** → **"Enviar documento"** → tipo **"Laudo médico"** → beneficiário Lucas → **Galeria** → foto do caso → consentimento → **"Enviar documento"**. A leitura é assíncrona: o documento aparece como "Recebido" e, depois de alguns minutos, muda para o resultado. No app, cada envio vira um documento numerado, e o número de cada caso está na tabela de resultados.
5. **Passo a passo (G03):** aba **Direitos** → "Passe Livre Interestadual – PcD" do Lucas → seção "Passo a passo". O mesmo caso também foi gerado no Swagger da API publicada, em 18/11/2026.
6. **Prints:** tirados direto do emulador (`adb exec-out screencap`), sem edição.

## 2. Resultados

**8 de 8 casos com o resultado esperado.**

| Id | Resultado no app | OK? | Print |
|---|---|---|---|
| R01 | 3 de 3 remédios lidos, com nome, princípio ativo e dosagem certos; os 3 aparecem como "De graça" no Farmácia Popular | ✅ | [`receita-r01-leitura-1.png`](prints/receita-r01-leitura-1.png), [`receita-r01-leitura-2.png`](prints/receita-r01-leitura-2.png), [`receita-r01-farmacia-popular.png`](prints/receita-r01-farmacia-popular.png) |
| R02 | 3 de 3 nomes e dosagens lidos; princípio ativo **vazio** nos 3 | ✅ | [`receita-r02-nomes-comerciais-1.png`](prints/receita-r02-nomes-comerciais-1.png), [`receita-r02-nomes-comerciais-2.png`](prints/receita-r02-nomes-comerciais-2.png) |
| R05 | "Nenhum remédio foi encontrado na receita." | ✅ | [`receita-r05-recusa.png`](prints/receita-r05-recusa.png) |
| N02 | status "Não deu para ler" | ✅ | [`laudo-n02-conta-de-luz-recusa.png`](prints/laudo-n02-conta-de-luz-recusa.png) (47.png) |
| N03 | status "Não deu para ler" | ✅ | [`laudos-n03-n04-l11-recusa.png`](prints/laudos-n03-n04-l11-recusa.png) (51.png) |
| N04 | status "Não deu para ler" | ✅ | [`laudos-n03-n04-l11-recusa.png`](prints/laudos-n03-n04-l11-recusa.png) (52.png) |
| L11 | status "Não deu para ler" | ✅ | [`laudos-n03-n04-l11-recusa.png`](prints/laudos-n03-n04-l11-recusa.png) (53.png) |
| G03 | 5 passos, todos começando com verbo, com a fonte "Lei 8.899/1994; Ministério dos Transportes – Programa Passe Livre" e o aviso "Texto explicado por IA" | ✅ | [`rag-g03-passe-livre.png`](prints/rag-g03-passe-livre.png), [`swagger-orientacao-gerar-caso2.png`](prints/swagger-orientacao-gerar-caso2.png) |

No print [`laudos-n03-n04-l11-recusa.png`](prints/laudos-n03-n04-l11-recusa.png) aparece também o 54.png, ainda como "Recebido". É outro envio, que não terminou, e não faz parte destes 8 casos.

### Métricas

| Função | Métrica | Resultado |
|---|---|---|
| Receita | remédios encontrados / prescritos (R01 e R02) | 6 / 6 |
| Receita | dosagens como escritas | 6 / 6 |
| Receita | **princípio ativo inventado** quando não estava escrito (R02) | **0 de 3** |
| Receita | documento errado recusado (R05) | 1 / 1 |
| Laudo | documentos inválidos recusados (N02, N03, N04, L11) | 4 / 4 |
| Laudo | **falsas aceitações** (documento inválido que seria gravado e abriria direitos) | **0** |
| RAG | passo a passo entre 3 e 7 passos, começando com verbo, com a fonte do trecho | 1 / 1 |

O passo a passo do G03 aparece no app como o cidadão vê. O mesmo caso foi gerado na API publicada pela rota `POST /api/v1/casos/2/orientacao/gerar`, com HTTP 200, a fonte vinda do trecho e o modelo `gemini-2.5-flash` ([`swagger-orientacao-gerar-caso2.png`](prints/swagger-orientacao-gerar-caso2.png)).

## 3. Critérios e análise

**Como cada caso foi julgado.** O resultado do app foi comparado com o esperado da tabela de casos de teste:

- **Receita:** cada remédio precisa aparecer com o nome e a dosagem como estão escritos. O princípio ativo só pode ser preenchido quando está escrito na receita.
- **Laudo:** um documento inválido precisa terminar como "Não deu para ler", sem gravar laudo nem abrir direitos.
- **RAG:** o passo a passo precisa ter de 3 a 7 passos, começar com verbo e citar a fonte do trecho da base, que a IA não escreve.

**Por que esses critérios.** A IA do Direito Fácil não é um chat. Ela transcreve documentos e explica direitos a partir de uma base conferida. Por isso os erros mais graves são:

- **Inventar dados.** Um princípio ativo completado de memória pode fazer o Farmácia Popular dar um veredito errado.
- **Aceitar um documento errado.** Uma conta de luz lida como laudo abriria pedidos de direitos indevidos.

As situações de falha atacam exatamente esses dois pontos.

**O que os resultados mostram:**

- **A IA não completa o que não está escrito.** No R02, ela deixou o princípio ativo vazio nos 3 remédios, mesmo sendo nomes conhecidos.
- **Documentos errados param antes do banco.** Os 4 laudos inválidos foram recusados, inclusive o L11. Nesse caso o laudo é legível e a IA consegue ler, mas a **conferência do Java** recusa por falta de data. Isso mostra a defesa em duas camadas: o modelo e a regra determinística depois dele.
- **O passo a passo cita a fonte do trecho usado**, e o app avisa que o texto foi explicado por IA e que quem decide o direito é a regra oficial.

**Limitações desta avaliação:**

- **A amostra é pequena** (8 casos) e fictícia: documentos gerados por computador, limpos. Laudos e receitas reais (manuscrito, carimbo, foto ruim, várias páginas) devem errar mais.
- **A leitura de um laudo válido** (o caminho que grava o laudo e abre os direitos) **não tem evidência nesta amostra**. Os 4 casos de laudo são situações de falha.
- **O RAG foi testado em um direito só** (G03), e nenhum caso de direito fora da base foi testado.
- **A leitura do laudo é assíncrona.** O status levou alguns minutos para mudar de "Recebido" para "Não deu para ler", porque passa pelo RabbitMQ antes de chegar à IA.
- Outras limitações da própria IA estão em [`docs/limitacoes.md`](../docs/limitacoes.md).
