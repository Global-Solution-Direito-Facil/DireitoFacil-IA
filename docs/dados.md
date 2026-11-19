# Dados: fontes e preparação

## 1. Fontes de dados da IA em produção

| Dado | Fonte | Como entra | Quem mantém |
|---|---|---|---|
| **Catálogo de direitos** (20 direitos: nome, órgão, base legal, critérios) | Leis federais e canais oficiais, conferidos pelo grupo (`STA_VERIFICACAO = VERIFICADO` na tabela) | Tabela `T_DF_DIREITO` + `T_DF_CRITERIO_DIREITO` (Oracle), carga Flyway `V3` | repositório [DireitoFacil-DB](https://github.com/Global-Solution-Direito-Facil/DireitoFacil-DB) |
| **Base de conhecimento do RAG** (como pedir cada direito) | Texto escrito pelo grupo a partir da lei e das cartilhas oficiais de cada órgão (INSS/Meu INSS, Ministério dos Transportes, Receita Federal/SISEN, ANEEL, CRAS/CadÚnico, CONTRAN...) | `src/main/resources/conhecimento/direitos.md`: um bloco por direito, com `Código:` e `Fonte:` | API Java (um teste automatizado garante que todo direito do banco tem texto) |
| **Elenco do Farmácia Popular** (41 itens, atualizado em 31/08/2026) | Elenco oficial do programa | Tabela `T_DF_MEDICAMENTO_FP` | DireitoFacil-DB |
| **Laudo médico / receita** | Documento do próprio cidadão, enviado pelo app | Arquivo na API .NET; o Java baixa, a IA lê, o Java confere e grava só os campos estruturados | o cidadão (dado sensível de saúde, ver LGPD no README) |
| **Salário mínimo** (para as frações de renda da regra) | Decreto federal vigente | Tabela `T_DF_SALARIO_MINIMO` | DireitoFacil-DB |

Nenhum dado real de cidadão foi usado em desenvolvimento. Os usuários de demonstração da API (Ana, Carlos, Helena, Marcos, Rita) são fictícios.

## 2. Preparação dos dados

### 2.1 Base de conhecimento (RAG)

1. **Escrita**: cada direito recebeu um texto curto no padrão "o que é" + "Como pedir:" (lista numerada), em linguagem simples, com a fonte legal/oficial na linha `Fonte:`. Os textos não contêm valores em reais nem prazos que mudam todo ano, justamente para a IA não os repetir desatualizados.
2. **Divisão (chunking)**: `IaBaseConhecimento.lerTrechos()` divide o arquivo nos títulos `# `. Cada direito vira **um** `Document` do Spring AI, com texto = título + corpo e metadados `codigo`, `fonte`, `titulo`. Não há chunking fino: a base tem 20 textos curtos, cada um uma unidade de sentido completa.
3. **Embeddings**: `gemini-embedding-001` gera um vetor por trecho; os 20 vetores ficam no `SimpleVectorStore` em memória. A indexação roda na primeira consulta (não na subida da API).
4. **Consulta**: a pergunta é montada a partir do caso (`"Como pedir {nome}. Órgão: {órgão}. Base legal: {base legal}."`), busca `topK = 3` por similaridade de cosseno e, se algum trecho tiver o mesmo código do direito, só ele é usado.

### 2.2 Laudo e receita (Visão)

Não há pré-processamento de imagem no Java: o arquivo vai ao modelo como foi enviado (PDF, JPEG ou PNG, até o limite de 60 s por chamada). A "preparação" está no prompt e no contrato de saída:

- o prompt de sistema diz o que transcrever e o que é proibido (deduzir, completar, opinar);
- o formato de saída é um `record` Java (`LaudoLido`, `ReceitaLida`) e o Spring AI pede ao modelo JSON nesse esquema;
- a **conferência pós-modelo** (`paraRequest`) normaliza (CID em maiúsculas, texto `strip`, limites das colunas) e recusa leitura incompleta (sem CID, sem condição, sem data, data no futuro) ou documento que não é laudo.

## 3. Como os dados são protegidos

- O repositório não contém chave de API, token, senha nem dado pessoal. A chave do Gemini fica só na variável de ambiente `GEMINI_API_KEY` da API Java.
