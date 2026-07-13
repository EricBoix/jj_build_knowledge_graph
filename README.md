# Hands on lowbrow exploration of GraphRAG in Python<!-- omit in toc -->

## Table of contents<!-- omit in toc -->

- [Introduction](#introduction)
- [Running things: the simple extraction use case](#running-things-the-simple-extraction-use-case)
- [Running with Docker](#running-with-docker)
- [Further advanced document "chunckings"](#further-advanced-document-chunckings)
- [Visually explore the resulting knowledge graph (with neo4j web UI)](#visually-explore-the-resulting-knowledge-graph-with-neo4j-web-ui)
- [Use the knowledge graph programmatically](#use-the-knowledge-graph-programmatically)
- [Dump/Restore the database content for later usage](#dumprestore-the-database-content-for-later-usage)
- [LLM (calls) observability](#llm-calls-observability)
- [References](#references)
- [Next steps](#next-steps)

## Introduction

This directory explores, with a direct hands-on approach, a process of graph extraction (and exploitation) that is described in the ["Local GraphRAG with LLaMa 3.1 - LangChain, Ollama & Neo4j" youtube tutorial](https://www.youtube.com/watch?v=nkbyD4joa0A).
The original associated code, from which this work is partly derived, is available through [this Coding Crash Courses git repository](https://github.com/Coding-Crashkurse/GraphRAG-with-Llama-3.1.git).

## Running things: the simple extraction use case

### Configure and start a Neo4j database (to collect the extracted graph)

Refer to [jj_workflow_shell configuration stage](https://github.com/EricBoix/jj_workflow_shell.git/Readme.md) in order to configure the shell utilities/methods.

TLDR;

```bash
cd `git rev-parse --show-toplevel`         # Implicit from now on
git clone https://github.com/EricBoix/jj_workflow_shell.git
cp env-reference .env
# Edit and configure resulting .env file
export RESULTS_DIR=`pwd`/result_data       # Syntactic sugar
\rm -fr $RESULTS_DIR/database
```

```bash
source jj_workflow_shell/Neo4jDatabase.sh    # Implicit from now on
launch_neo4j_db $RESULTS_DIR $NEO4J_PORT $NEO4J_USERNAME/$NEO4J_PASSWORD
```

### Realize the graph extraction

```bash
# Prepare the virtual environment
python3.10 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
```

```bash
# Retrieve some input data e.g.
git clone https://github.com/EricBoix/jj_doc_Four_Noble_Truths.git jj_doc_Four_Noble_Truths.git
# Extract the graph and store it in database
python extracting_graph.py \
--input_directory jj_doc_Four_Noble_Truths.git/original_data/ \
--load_markdown_document 250_BCE_-_Dhammacakkappavattana_Sutta_Four_Noble_Truths_Wikipedia_translation.md
```

### Notes

- when the extraction is too lengthy (or running on a remote ssh server) consider using

    ```bash
    python extracting_graph.py [...] > extract.log &
    tail -f extract.log
    ```

- when a pre-existing neo4j database content exists (look a the content of the `./data` directory), and depending on your filesystem rights setup) you might get an error message on

    ```bash
    docker run --interactive --tty --rm --volume=`pwd`/data:/data neo4j /usr/bin/rm -fr /data/*
    rmdir ./data
    ```

## Running with Docker

Build the image from the repository root:

```bash
docker build -t jejuneness:jj_build_knowledge_graph https://github.com/EricBoix/jj_build_knowledge_graph.git#:DockerContext
```

Shallow testing of the image

```bash
docker run jejuneness:jj_build_knowledge_graph extracting_graph.py --help
```

Run the extraction (adjust paths and `.env` as needed):

```bash
docker run --rm \
  -v /path/to/data:/data \
  --env-file .env \
  jejuneness:jj_build_knowledge_graph \
  extracting_graph.py --input_directory /data --load_markdown_document file.md
```

## Further advanced document "chunckings"

If you wish to break down the original document in chunks that follow the sentence structure (as opposed to evenly sized chunks with some overlap) use the following script (that depends on the output of [Collecting Gold Dust conversion](https://github.com/EricBoix/jj_doc_Collecting_Gold_Dust/blob/main/Readme.md)) :

```bash
python extracting_graph_semantic_chuncker.py \
--input_directory ../../../Data/ISBN_978-0-9835844-5-2_-_Collecting_Gold_Dust/ \
--load_markdown_document result_data/2019_-_Sayadaw-U-Tejaniya-Collecting-Gold-Dust-Web-Book-1_-_local_converter.md \
--load_json_document result_data/2019_-_Sayadaw-U-Tejaniya-Collecting-Gold-Dust-Web-Book-1_-_Sentences_as_LangChain_Document.json
```

## Visually explore the resulting knowledge graph (with neo4j web UI)

Interactively explore the extracted graph through neo4j web UI

```bash
# For exact hostname/port refer to the NEO4J_URI entry of your `/.env` # configuration file):
open http://localhost:7474/
```

Run [`cypher (queries)`](https://neo4j.com/docs/cypher-manual/current/introduction/) like

```bash
# Assert the UI is connected to the proper db server (has to match what was 
# configured in you .env file)
$:server connect   
# Make sure you are connect to the right database (again this has to match 
# with what was configured in you .env file)
$:use neo4j
# Display all the nodes of the full extracted graph. Caveat emptor: 
# within the UI settings, the "Initial node display" integer parameter 
# controls the number of nodes displayed which defaults to 300
neo4j$ MATCH (n) RETURN n
# Display all nodes AND edges
neo4j$ MATCH (n) MATCH ()-[r]->() RETURN n, r

# Display the nodes with the "Person" label
neo4j$ MATCH (n) WHERE n:Person RETURN n
# Display the nodes NOT having the "Person" label
neo4j$ MATCH (n) WHERE NOT n:Person RETURN n
# Display the nodes not having "Document" as single label
neo4j$ MATCH (n) WHERE NOT(SIZE(LABELS(n)) = 1 AND n:Document) RETURN n
# A composition of the above
neo4j$ MATCH (n) WHERE NOT(SIZE(LABELS(n)) = 1 AND n:Document) and NOT n:Person RETURN n
# Nodes that are not documents together with their relations
neo4j$ MATCH (n) WHERE NOT n:Document OPTIONAL MATCH (n)-[r]-(c) WHERE NOT c:Document RETURN n,r,c
...
```

## Use the knowledge graph programmatically

### Displaying knowledge graph main characteristics

```bash
python display_neo4jdb_graph_characteristics.py
```

### Searching the database

For example search the graph database with a Natural Language query by running the provided python script

```bash
python extract_from_knowledge_graph.py
```

Or search both the knowledge graph and the embedding space structures with

```bash
python vector_and_graph_hybrid_search.py
```

## Dump/Restore the database content for later usage

Again, refer to [jj_workflow_shell configuration stage](https://github.com/EricBoix/jj_workflow_shell.git/Readme.md) in order to configure and use the `dump_database` and `restore_database` shell utilities/methods.

## LLM (calls) observability

Refer to [`Observability/README.md`](Observability/README.md) for installation, backend launch, and a guided analysis walkthrough of LLM observability.

Here are a few numerical results. The markdown and sentences columns indicate the number of [LangChain documents](https://reference.langchain.com/python/langchain-core/documents) that where respectively sent to the llm for interpretation (extracting nodes and edges).

| Book | Markdown (Chuncker) | Sentences | # llm calls |
| ---- | -------- | --------- | ----------- |
| [Four Noble Truths](https://github.com/EricBoix/jj_doc_Four_Noble_Truths) | FIXME | FIXME | FIXME |
| [Collecting Gold Dust](https://github.com/EricBoix/jj_doc_Collecting_Gold_Dust) | FIXME | FIXME | FIXME |
| [Zen flesh, zen bones](https://github.com/EricBoix/jj_doc_Zen_Flesh_Zen_Bones) | 45 (UnstructuredMarkdownLoader) | 2479 | 2524 |

## References

- [GraphRAG: The Marriage of Knowledge Graphs and RAG: Emil Eifrem](https://www.youtube.com/watch?v=knDDGYHnnSI)
- [Introduction to Neo4j](https://www.youtube.com/watch?v=YDWkPFijKQ4cdt)
- [Build a RAG agent with LangChain](https://docs.langchain.com/oss/python/langchain/rag#ollama)
- Calling [LLM through OpenwebUI examples](https://github.com/UDL-LIRIS/python-openwebui-bootstraping-examples)

## Next steps

### Improve graph extraction: define and use ontologies

- Refer to this inadequate yet inspiring, [OWL Ontology of Consciousness](https://www.researchgate.net/publication/383227961_An_OWL_Ontology_of_Consciousness)
- [Knowledge graphs organizing principles tutorial](https://medium.com/@michelleloh.tech/day-15-learn-knowledge-graphs-part-2-organizing-principles-17c76eb3686b)

### Improve graph extraction: more stuff to try

- [Read this and improve the script](https://neo4j.com/blog/developer/knowledge-graph-extraction-challenges/)

### Improve graph extraction: explore alternative chunckings

[RAG chunking strategies article](https://dev.to/sreeni5018/rag-chunking-strategies-4i3a) mentions 6 strategies (checked box indicate strategies used/explored with this code)

- Fixed-Size chunking (LangChains's `CharacterTextSplitter`)
- Recursive character chunking (LangChains's `RecursiveCharacterTextSplitter`)
- Semantic chunking
- Document structure-aware chunking for example
  - [x] Markdown aware chunking (e.g. LangChain's `MarkdownHeaderTextSplitter`)
  - [x] Grammar aware chunking: use paragraph and sentence structure.
- Hierarchical (parent/child) chunking: can be naturally combined with/deduced from document-structure-aware chunking
- LLM-based (and agentic chunking)

The [`ToolTesting/GraphRAG/extracting_graph.py` code](./Doc/ToolTesting/GraphRAG/extracting_graph.py#32) currently uses [LangChain's `RecursiveCharacterTextSplitter`](https://reference.langchain.com/python/langchain-text-splitters/character/RecursiveCharacterTextSplitter) as "chuncker". But knowledge graph focuses on semantics and using a semantic based chuncker can only improve things (although it comes at a cost) at two levels : retrieval and citation. Since the [`ConvertPdfToMarkdown` package] produces sentence (and/or paragraph, sub-section...) based outputs we have the natural opportunity to use the available semantic chunckers starting with [langchain_experimental's  `SemanticChunker`](https://github.com/langchain-ai/langchain-experimental/blob/main/libs/experimental/langchain_experimental/text_splitter.py#L99).

References:

- [`SemanticChunker` class](https://github.com/langchain-ai/langchain-experimental/blob/main/libs/experimental/langchain_experimental/text_splitter.py#L99) as offered by [langchain_experimental (python package)](https://github.com/langchain-ai/langchain-experimental/tree/main)
- [langchain_experimental "SemanticChunker" tutorial](https://colab.research.google.com/github/LangChain-OpenTutorial/LangChain-OpenTutorial/blob/main/07-TextSplitter/04-SemanticChunker.ipynb#scrollTo=312e3aae)
- ["A Visual Exploration of Semantic Text Chunking" article](https://towardsdatascience.com/a-visual-exploration-of-semantic-text-chunking-6bb46f728e30/):
  - :warning: This article mentions that it is key to "use a model that has been trained to generate meaningful embeddings" and forwards to [`SentenceTransformers` library](https://sbert.net/)
- [Langchain's tutorial: Build a semantic search engine with LangChain](https://docs.langchain.com/oss/python/langchain/knowledge-base)

### Ingesting a Markdown file

- Use [LangChain's `UnstructuredMarkdownLoader`](https://docs.langchain.com/oss/python/integrations/document_loaders/unstructured_markdown)
