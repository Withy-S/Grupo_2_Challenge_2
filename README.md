**Documentos trabalhistas em linguagem simples, com fontes que você pode conferir.**

Aplicação de IA para ajudar trabalhadores a compreender documentos de processos trabalhistas, identificar a situação descrita e explorar possíveis próximos passos, com explicações acessíveis e referências às fontes.

Projeto acadêmico desenvolvido no **Desafio 2 da Residência em Inteligência Artificial**, com o tema **Qualidade de vida** e foco inicial na Justiça do Trabalho, tendo o TRT-10 como piloto.

> **Em planejamento.** Este repositório contém a documentação do projeto. A aplicação, os modelos e o ambiente de execução ainda serão implementados.

[Manual de MLOps e arquitetura](<docs/Manual de MLOps e arquitetura — Desafio 2 (direitos trabalhistas).docx>) · [Desafio original](<docs/Challenge 2.docx.pdf>)

## Por que este projeto existe?

Uma decisão judicial pode mudar a vida de alguém e, ainda assim, ser difícil de entender. Termos técnicos, documentos extensos e dúvidas sobre o andamento do processo tornam essa experiência ainda mais complicada.

Queremos ajudar a pessoa a compreender **o que está escrito, onde essa informação aparece e o que ainda não é possível concluir**.

A ferramenta terá caráter informativo. Orientações sobre decisões concretas do caso continuam dependendo de assistência jurídica profissional.

## A experiência que queremos construir

1. **Envie um documento:** PDF ou imagem, com escolha explícita sobre a guarda do arquivo.
2. **Acompanhe a leitura:** veja o progresso e eventuais problemas de legibilidade.
3. **Entenda o conteúdo:** receba uma explicação simples, com o trecho original ao lado.
4. **Confira as fontes:** localize o documento, a página ou a fonte jurídica que sustenta cada afirmação.
5. **Faça perguntas:** explore termos e informações presentes no material autorizado.
6. **Controle seus dados:** consulte as opções de retenção e exclusão.

A situação apresentada será a **última identificada nos documentos enviados**, com data de referência e limitações. O MVP não prevê consulta automática ao PJe ou ao DataJud.

## Funcionalidades planejadas

| Frente | Entrega prevista |
| --- | --- |
| Leitura de documentos | Extração de texto de PDFs e OCR para imagens, preservando página e posição |
| Organização do conteúdo | Separação de relatório, fundamentação e dispositivo, com divisão em trechos |
| Classificação e extração | Identificação de tipo documental, fase, campos e resultado por pedido |
| Explicação acessível | Resumos em linguagem simples, com o dispositivo literal para conferência |
| Perguntas com fontes | Recuperação de evidências dos documentos do usuário, das leis e da jurisprudência |
| Próximos passos | Possibilidades condicionais e checklists aprovados pela Curadoria jurídica |
| Validação | Verificação das frases antes de exibi-las, com sinalização de incertezas |
| Privacidade | Isolamento por usuário, guarda opcional e exclusão de dados e derivados |

## Arquitetura planejada

Cinco serviços principais — frontend, backend, Redis, worker e PostgreSQL — com armazenamento de originais e MLflow como apoio. O worker concentra o processamento com modelos e organiza o trabalho em três pipelines.

```mermaid
flowchart TB
    USER([Usuário]) --> WEB["Frontend — React<br/>Login, envio, progresso, fontes e exclusão"]
    WEB -->|Requisições autenticadas| API["Backend — FastAPI<br/>Autenticação, permissões, consentimento e filas<br/>Não carrega modelos"]
    API -->|Progresso e respostas por SSE| WEB
    API -->|Guarda só com opt-in| FILES[("Originais criptografados<br/>MinIO · prazo de exclusão")]
    SOURCES["Fontes jurídicas<br/>CLT, súmulas do TST e decisões do TRT-10"] --> CURATION{"Curadoria<br/>aprova a fonte?"}
    MLFLOW["MLflow<br/>Registro dos modelos · versões em uso"]

    subgraph REDIS["Redis — filas com RQ e streaming"]
        direction LR
        QDOC["Fila documentos"]
        QASK["Fila perguntas<br/>Prioritária"]
        PUB["Pub/Sub<br/>Frases aprovadas e progresso"]
        QLEGAL["Fila ingestao_juridica"]
    end

    API --> QDOC
    API --> QASK
    CURATION -->|Sim · lote enfileirado pelo backend| QLEGAL
    PUB -->|Resposta em streaming| API

    subgraph WORKER["Worker — único processo com modelos · contratos Pydantic entre módulos"]
        direction LR

        subgraph PREP["1. Preparação do documento"]
            direction TB
            READER["Leitor<br/>PDF: PyMuPDF · foto: qualidade e PaddleOCR<br/>Limpeza com página e posição preservadas"]
            SEGMENT["Segmentador<br/>Relatório, fundamentação e dispositivo<br/>Divisão em trechos"]
            CLASSIFIER["Classificador de ML<br/>BERTimbau + LoRA · baseline TF-IDF<br/>Tipo, fase, campos e resultado por pedido<br/>Confiança calibrada e abstenção"]
            INDEX["Indexação do documento<br/>Embeddings dos trechos<br/>Exclusão ao fim da sessão sem opt-in"]
            READER --> SEGMENT --> CLASSIFIER --> INDEX
        end

        subgraph EVIDENCE["2. Evidência, geração e validação"]
            direction TB
            INPUT["Guarda de escopo — entrada<br/>Classificador de intenção<br/>Desvia pedidos de aconselhamento"]
            RETRIEVER["Recuperador híbrido<br/>Vetor + palavra-chave nos três índices<br/>Filtros de usuário e vigência"]
            RERANK["Reranker — cross-encoder<br/>Avalia pares: pergunta e trecho<br/>Reordena fontes jurídicas e corta notas baixas"]
            EXPLAIN["Explicador — LLM via Ollama<br/>Campos extraídos + trechos recuperados<br/>Cada frase aponta sua evidência"]
            VERIFY["Verificador — cross-encoder NLI<br/>Avalia pares: frase e trecho citado<br/>Sustenta, contradiz ou não trata"]
            OUTPUT["Guarda de escopo — saída<br/>Barra frases de aconselhamento<br/>Publica frases aprovadas e grava resultados"]
            INPUT --> RETRIEVER --> RERANK --> EXPLAIN --> VERIFY -->|Aprovada| OUTPUT
            VERIFY -->|Reprovada: refaz uma vez ou remove| EXPLAIN
        end

        subgraph LEGAL["3. Ingestão jurídica"]
            direction TB
            SPLIT["Divisão pela estrutura jurídica<br/>Lei por artigo<br/>Jurisprudência por súmula, OJ ou ementa"]
            EMBED["Embeddings + metadados<br/>Fonte, versão e vigência<br/>Índice candidato"]
            REGRESSION["Regressão da suíte jurídica<br/>Perguntas-teste no índice candidato"]
            APPROVE{"Curadoria aprova<br/>a versão candidata?"}
            PUBLISH["Publicação da versão do índice<br/>Trecho revogado sai da busca<br/>Histórico preservado"]
            SPLIT --> EMBED --> REGRESSION --> APPROVE -->|Sim| PUBLISH
        end
    end

    QDOC --> READER
    FILES -.->|Leitura autorizada| READER
    QASK --> INPUT
    QLEGAL --> SPLIT
    MLFLOW -.->|Carrega versões em uso| WORKER
    CLASSIFIER -->|Campos, tipo e fase| RETRIEVER
    OUTPUT --> PUB

    subgraph POSTGRES["PostgreSQL + pgvector — busca vetorial e full-text no mesmo banco"]
        direction LR
        USERCHUNKS[("trechos_usuario<br/>Filtro por usuário")]
        RESULTS[("Contas, tarefas e resultados")]
        LAWS[("leis<br/>Versão e vigência")]
        CASELAW[("jurisprudencia<br/>Versão e vigência")]
    end

    INDEX --> USERCHUNKS
    RETRIEVER -.->|Consulta autorizada| USERCHUNKS
    RETRIEVER -.-> LAWS
    RETRIEVER -.-> CASELAW
    PUBLISH --> LAWS
    PUBLISH --> CASELAW
    OUTPUT -->|Grava resposta validada| RESULTS
    API <-->|Registra tarefas e consulta resultados| RESULTS

    classDef component fill:#112d42,stroke:#3e7293,color:#e6f1ff;
    classDef pairreview fill:#3b280e,stroke:#efa744,color:#ffe0a6,stroke-width:2px;
    classDef storage fill:#112d42,stroke:#3e7293,color:#e6f1ff;
    class USER,WEB,API,MLFLOW,QDOC,QASK,PUB,QLEGAL,READER,SEGMENT,CLASSIFIER,INDEX,INPUT,RETRIEVER,EXPLAIN,OUTPUT,SOURCES,CURATION,SPLIT,EMBED,REGRESSION,APPROVE,PUBLISH component;
    class RERANK,VERIFY pairreview;
    class FILES,USERCHUNKS,RESULTS,LAWS,CASELAW storage;
```


O backend autentica, verifica permissões e encaminha tarefas. **Todo processamento com modelos acontece no worker**, em três pipelines: documentos, perguntas e ingestão jurídica. Perguntas têm prioridade na fila.

O banco mantém o estado das tarefas e os resultados. O Pub/Sub entrega eventos ao backend, que os repassa à interface por SSE. A publicação será por frase, depois da verificação.

### Stack definida no manual

| Camada | Tecnologias previstas |
| --- | --- |
| Interface | React e TypeScript |
| API e contratos | Python, FastAPI e Pydantic |
| Filas e eventos | Redis, RQ e Pub/Sub |
| Dados e busca | PostgreSQL, pgvector e busca textual em português |
| Leitura | PyMuPDF e PaddleOCR |
| IA generativa | Modelos locais via Ollama, embeddings e RAG |
| Modelos avaliáveis | Baselines com regras e TF-IDF; BERTimbau e cross-encoders a avaliar |
| MLOps | MLflow, DVC, Label Studio e Evidently |
| Arquivos e ambiente | MinIO, uv e Docker Compose |

Versões, licenças e desempenho serão confirmados durante a implementação. No ambiente macOS previsto no manual, worker e Ollama executarão nativamente; PostgreSQL, Redis, MLflow e MinIO serão organizados pelo Docker Compose.

## Dados, evidências e qualidade

O projeto terá três conjuntos de busca: **`trechos_usuario`**, **`leis`** e **`jurisprudencia`**. Os documentos privados serão filtrados pelo usuário autorizado; as fontes jurídicas terão origem, versão e informações de vigência.

As práticas previstas incluem:

- Preservar a origem de cada trecho, incluindo página e posição.
- Anonimizar o acervo de desenvolvimento e versionar dados e rótulos com DVC.
- Separar treino, validação e teste por processo, evitando vazamento entre conjuntos.
- Validar os contratos entre módulos e os esquemas dos lotes.
- Comparar modelos com baselines e registrar experimentos e versões no MLflow.
- Avaliar extração, segmentação, classificação, recuperação e fidelidade das explicações.
- Submeter mudanças de modelos, prompts e índices a testes e revisão antes de promover uma versão.
- Bloquear versões com erros graves no conjunto de avaliação.

As metas quantitativas serão estabelecidas após medir os primeiros baselines. Os controles descritos aqui são compromissos de implementação; ainda não representam resultados alcançados.

## Privacidade desde o início

O manual prevê processamento local dos modelos e uso de dados sintéticos ou anonimizados no desenvolvimento. Documentos pessoais e segredos não devem entrar no Git.

A guarda de originais dependerá de escolha explícita do usuário. Sem essa escolha, documento, trechos e resultados deverão ser removidos ao fim da sessão; com guarda, serão aplicados controles de acesso, criptografia e prazo de exclusão. Logs deverão registrar apenas informações operacionais, sem reproduzir o conteúdo dos documentos.

## Quem faz o quê

| Papel | Responsabilidade principal | Responsável |
| --- | --- | --- |
| Engenharia de dados | Acervo, anonimização, leitor, segmentador por regras, divisão estrutural das fontes e contratos de dados | Davi |
| Ciência de dados | Taxonomia, anotação, conjuntos de referência, métricas, análise de erros e testes com pessoas | Laisa |
| Engenharia de ML e MLOps | Classificadores, reranker, verificador, experimentos, registro de modelos e monitoramento | Felipe |
| Engenharia de IA | Índices, recuperação, embeddings, LLM, prompts e integração da validação das respostas | Mayara |
| Engenharia de software | Frontend, backend, banco, migrações, filas, autenticação, infraestrutura e integração | João |
| Curadoria jurídica | Aprovação das fontes, rótulos jurídicos, limites de uso e revisão das explicações | Laisa |

## Roteiro de desenvolvimento

| Semana | Foco |
| --- | --- |
| 1 | Escopo, fontes, anonimização, contratos, métricas e ambiente |
| 2 | Primeiro lote de dados e recuperação de fontes jurídicas |
| 3 | Comparação entre LLM, RAG, reranker e verificador |
| 4 | Integração dos módulos e primeiro fluxo completo |
| 5 | Leitura de imagens, OCR e avaliação de fotos |
| 6 | Integração completa, monitoramento e preparação da demonstração |
| 7 | Testes com pessoas, refinamento e apresentação final |

O roteiro segue o manual e depende dos portões de qualidade de cada etapa.

