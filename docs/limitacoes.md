# Limitações da IA e uso responsável

Este documento lista o que a IA do Direito Fácil **não** faz bem, o que o projeto faz a respeito e o que ficaria para uma próxima versão.

## 1. Limitações do modelo e da abordagem

| Limitação | Efeito no produto | Mitigação atual | Próximo passo |
|---|---|---|---|
| **Modelo por API externa** (Gemini). Sem internet, sem chave ou com o serviço fora do ar, a IA não responde. | Leitura do laudo e passo a passo ficam indisponíveis (503). | A API sobe sem a chave; o gestor registra o laudo à mão (`POST /laudos`) e a orientação pode ser gerada depois (`POST /casos/{id}/orientacao/gerar`). O evento fica no log para reprocessar. | Modelo local (por exemplo, um VLM pequeno via Ollama) como contingência para o CRAS sem conectividade estável. |
| **Limite de uso do plano gratuito** (~10 pedidos/min, cota diária). | Em pico (mutirão do CRAS), leituras podem falhar por 429. | `@Retryable` na API. | Plano pago ou fila com reprocessamento automático (DLQ no RabbitMQ). |
| **Dados do cidadão saem para um provedor externo** nas leituras (a foto do laudo vai ao Gemini). | Risco de privacidade; LGPD exige base legal e transparência. | Só o arquivo vai, sem nome de conta nem CPF; o passo a passo não envia dado pessoal nenhum. No plano gratuito, o Google pode usar o conteúdo enviado para melhorar seus produtos. | Plano pago (o conteúdo não é usado para treino); termo de consentimento explícito no app; avaliar modelo local para o laudo. |
| **Visão não é OCR certificado.** Manuscrito real, foto muito ruim, carimbo sobre o texto e laudos de várias páginas erram mais. | Leitura incompleta (recusada) ou, pior, campo errado. | Conferência pós-modelo recusa leitura incompleta; `tipoCondicao` desconhecido vira `OUTRA` (não abre direito); o único dado que entra na regra é o tipo. | Tela de conferência do laudo lido **antes** de gravar, como já existe para a receita; pedir foto nova quando `legivel = false`. |
| **Classificação do tipo da condição é um julgamento clínico simplificado.** Síndrome de Down sem a palavra "deficiência intelectual", TEA com comorbidade, deficiência múltipla: o modelo pode escolher outro tipo. | Pode abrir direito a menos (nunca a mais, porque `OUTRA` não conta). | A regra PL/SQL só usa o tipo; o gestor pode corrigir o laudo. | Lista de CIDs por tipo no banco (determinística) para validar ou substituir a classificação do modelo. |
| **A IA transcreve "como está escrito"**, inclusive erros do laudo (CID errado, data errada). | O sistema herda o erro do documento. | Não é defeito da IA; a conferência pega data futura e CID fora do tamanho. | Cruzar o CID com a descrição (tabela CID-10) e alertar divergência. |
| **RAG com base pequena e escrita pelo grupo.** Se a base estiver desatualizada ou errada, a orientação estará também, com a fonte certa. | Orientação correta em relação à base, mas potencialmente desatualizada em relação à lei. | Textos sem valores em reais nem prazos que mudam; fonte sempre citada; `STA_VERIFICACAO` no catálogo. | Rotina de revisão da base com data de conferência por trecho; ingestão das cartilhas oficiais (PDF) como fonte primária. |
| **Direitos parecidos** (BPC idoso × BPC PcD; credencial idoso × credencial PcD) têm embeddings próximos. | Sem o filtro, o modelo poderia misturar. | Filtro pelo código depois da busca; `encontrada = false` quando os trechos não são do direito. | |
| **Não determinismo residual.** Temperatura 0 reduz, mas não elimina, variação entre execuções e entre versões do modelo. | A mesma entrada pode gerar respostas um pouco diferentes. | Conferência determinística em Java depois de toda resposta do modelo. | Fixar a versão do modelo (`gemini-2.5-flash-<data>`). |

## 2. O que a IA nunca faz

- **Não decide elegibilidade.** Isso é da função PL/SQL `PKG_ELEGIBILIDADE`, a partir de critérios em tabela.
- **Não diz se o remédio é gratuito.** Isso é da tabela do Farmácia Popular.
- **Não escreve a fonte legal.** A fonte vem dos metadados do trecho usado.
- **Não grava nada sem a conferência do Java** (`LaudoLido.paraRequest`, `OrientacaoGerada.paraRequest`), nem sem a conferência da pessoa (receita).
- **Não recebe dado pessoal no passo a passo.**

## 3. Uso responsável

- **Credenciais**: nenhuma no repositório. A chave do Gemini fica só na variável de ambiente `GEMINI_API_KEY` da API Java; sem ela a API sobe e só as funções de IA respondem 503.
- **Dados pessoais**: nenhum dado real de cidadão no repositório; os usuários de demonstração da API são fictícios. Laudo é dado sensível de saúde (LGPD, art. 5º, II): só o arquivo vai ao modelo, sem nome nem CPF; o passo a passo não envia dado pessoal nenhum.
- **Transparência**: as limitações estão explicadas aqui e no README, e o uso de IA generativa no desenvolvimento está declarado no README.

