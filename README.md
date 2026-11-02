# Orthus.AI - Tutor Técnico 

> Um tutor inteligente e assistente técnico desenvolvido para orientar estudos, esclarecer dúvidas de engenharia de IA e fundamentar conceitos avançados de **Visão Computacional**, **Agentes Inteligentes**, **LLMs** e **Machine Learning** utilizando **RAG (Retrieval-Augmented Generation)** e orquestração de fluxos com **LangGraph**.

![Uploading image.png…]()


## O que o Orthus.AI faz

* **Atua como Tutor Técnico Especializado:** Responde dúvidas sobre quatro grandes áreas da inteligência artificial (*Visão Computacional*, *Agentes Inteligentes*, *LLMs & APIs*, e *Machine Learning*).

* **Busca Semântica (RAG):** Consulta dinamicamente uma base de conhecimento própria (livros, artigos e documentações em PDF, MD e TXT) para fundamentar todas as respostas com referências precisas.

* **Rastreabilidade de Fontes:** Exibe no chat os trechos exatos, nomes de arquivos e números de páginas consultados para cada resposta gerada.

* **Análise de Documentos Anexados:** Permite enviar arquivos (PDF, DOCX, TXT, MD, PY, CSV, JSON) para resumo, extração de pontos-chave e explicação direta sem poluir o índice do RAG.

* **Respostas Estruturadas:** Padroniza as explicações em blocos didáticos (*Conceito Técnico*, *Exemplo Prático/Código*, *Detalhamento da Lógica*, *Aplicações Práticas* e *Referências/Documentação*).

* **Proteção por Design:** Conta com tratamento de prompt injection para garantir que conteúdos de documentos anexados sejam processados estritamente como dados.

## Como funciona

```
            +-------------------------------------------------------+
            |                  Ingestão (Offline)                   |
            | base_conhecimento/ -> Splitter (800) -> HuggingFace   |
            | Embeddings -> Chroma DB (chroma_db/)                  |
            +-------------------------------------------------------+

                                  Interface Streamlit (orthus.py)
┌──────────────────┐  Pergunta/Arquivo   ┌──────────────────────────────────────────┐
│ Entrada de Texto │ ──────────────────► │ Grafo LangGraph (RAGState)               │
│ & Anexo          │                     │ ┌──────────────────────────────────────┐ │
│                  │                     │ │ 1. decidir: usar_rag == True?        │ │
│ Resposta com     │ ◄────────────────── │ │    ├─ Sim ──► recuperar (Chroma k=4)  │ │
│ blocos e fontes  │  (Resposta + Fontes)│ │    └─ Não ──► responder (LLM)         │ │
│ consultadas      │                     │ └──────────────────────────────────────┘ │
└──────────────────┘                     └──────────────────────────────────────────┘
                                                             │
                                                             ▼
                                                    OpenRouter / OpenAI API
                                                    (ChatOpenAI: Gemma 4 31B, etc.)

```

### Arquitetura do RAG & Grafo

1. **Pipeline de Ingestão (`ingest.py`):**

   * **Chunking:** Processa arquivos da pasta `base_conhecimento/` utilizando `RecursiveCharacterTextSplitter` (tamanho de chunk: `800` caracteres, sobreposição: `120`).

   * **Embeddings:** Converte os trechos em vetores numéricos via `sentence-transformers/paraphrase-multilingual-MiniLM-L12-v2`.

   * **Banco Vetorial:** Armazena e indexa os vetores no **Chroma DB** (persistido na pasta `chroma_db/`).

2. **Fluxo de Execução com LangGraph:**

   * **Nó `decidir`:** Avalia se o RAG deve ser ativado (desativado apenas quando o usuário anexa arquivos sem digitar uma pergunta textual).

   * **Nó `recuperar`:** Executa a busca vetorial por similaridade e recupera os **TOP 4** trechos mais relevantes do Chroma DB.

   * **Nó `responder`:** Consolida o prompt do sistema, o histórico da conversa, o texto do anexo (se houver) e os trechos recuperados do RAG, enviando o contexto para o LLM via OpenRouter API.

## Estrutura do repositório

```
orthus-ai/
├── base_conhecimento/   # Arquivos da base de conhecimento (PDF, MD, TXT)
├── chroma_db/           # Banco vetorial persistido pelo ingest.py
├── ingest.py            # Script para criação e indexação da base vetorial
├── orthus.py            # Aplicação principal Streamlit + Grafo LangGraph
├── requirements.txt     # Dependências do projeto
├── .env.example         # Exemplo de configuração das variáveis de ambiente
└── README.md            # Documentação do projeto

```

## Configuração do ambiente

### 1. Pré-requisitos

* **Python 3.10+**
* **Git**
* Uma chave de API do **OpenRouter**

### 2. Criar uma chave de API no OpenRouter

1. Acesse o [OpenRouter](https://openrouter.ai/).
2. Crie uma conta ou faça login.
3. Acesse a seção **Models** e consulte os modelos disponíveis.
4. Escolha um modelo compatível com o projeto, como o **Google Gemma 4 31B** (ou utilize modelos gratuitos disponíveis).
5. Acesse a seção **API Keys** e clique em **Create API Key**.
6. Copie a chave gerada e guarde-a em local seguro.

> **⚠️ Importante:** Não compartilhe sua chave de API nem a envie para repositórios públicos no GitHub.

### 3. Criar e ativar o ambiente virtual

No terminal, acesse a pasta do projeto e crie o ambiente virtual:

**Windows (PowerShell):**

```powershell
python -m venv env
.\env\Scripts\Activate.ps1
```

**Linux / macOS:**

```bash
python3 -m venv env
source env/bin/activate
```

Com o ambiente ativo, o terminal exibirá `(env)` antes do prompt.

### 4. Configurar as variáveis de ambiente

Crie um arquivo chamado `.env` na raiz do projeto e adicione as seguintes variáveis:

```env
M_API_KEY=sua_chave_api_aqui
OPENROUTER_MODEL=openrouter/free
```

*Substitua `sua_chave_api_aqui` pela chave gerada no OpenRouter.*

### 5. Instalar as dependências

Com o ambiente virtual ativado, execute:

```powershell
python -m pip install --upgrade pip
pip install -r requirements.txt
```

*(Caso não possua o arquivo `requirements.txt`, instale manualmente:)*

```powershell
pip install streamlit langchain-core langchain-openai langgraph python-dotenv pypdf python-docx langchain-community langchain-text-splitters langchain-chroma langchain-huggingface sentence-transformers
```

## Executando a aplicação

### 1. Indexar a Base de Conhecimento

Se for a primeira vez executando a aplicação ou caso tenha adicionado novos documentos na pasta `base_conhecimento/`, rode o script de ingestão:

```powershell
python ingest.py
```

> **Nota:** Ao reindexar a base, recomenda-se apagar a pasta `chroma_db/` existente antes de executar o script para evitar trechos duplicados.

### 2. Iniciar a Interface Streamlit

Execute a aplicação web com o comando:

```powershell
streamlit run orthus.py
```

O Streamlit iniciará o servidor local e exibirá o endereço no terminal (geralmente `http://localhost:8501`). Abra esse endereço no navegador.

## Tecnologias Utilizadas

* **Python:** Linguagem principal do projeto.
* **Streamlit:** Interface de usuário interativa em formato de chat.
* **LangChain:** Integração de componentes de linguagem, embeddings e recuperadores.
* **LangGraph:** Orquestração do grafo de decisão e controle de estado do RAG.
* **Chroma DB (`langchain-chroma`):** Armazenamento e busca semântica em banco vetorial.
* **HuggingFace Embeddings:** Modelo `sentence-transformers/paraphrase-multilingual-MiniLM-L12-v2` para representação semântica multilíngue.
* **OpenRouter (ChatOpenAI):** Infraestrutura e API de acesso aos modelos LLM.
* **python-dotenv:** Gerenciamento de variáveis de ambiente.
* **pypdf & python-docx:** Extração de texto de documentos PDF e Word anexados no chat.

## Limitações conhecidas e Próximos Passos

* **Formatos de Ingestão:** O `ingest.py` realiza a ingestão automática de arquivos `.pdf`, `.md` e `.txt`. O suporte a arquivos `.docx` atualmente é exclusivo para anexos diretos no chat.
* **Persistência de Histórico:** O histórico de conversa é mantido na sessão ativa (`st.session_state`) e é reiniciado ao recarregar a página.
* **Estratégia de Recuperação:** O RAG utiliza busca por similaridade vetorial simples ($k=4$). A inclusão de *Reranking* e *Busca Híbrida (BM25 + Vetorial)* está planejada para melhorias futuras.
