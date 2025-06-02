orag - run offline rag queries with ollama backend
==================================================
The `orag` command line tool combines content retrieved from local text documents with 
the [LLMs](https://ollama.com/library) supported by the [ollama](https://github.com/ollama/ollama) backend.
Like ollama (offline llama) it keeps the document data required for
retrieval-augmented generation ([RAG](https://en.wikipedia.org/wiki/Retrieval-augmented_generation)) offline,
so that no private information is exposed to the internet.

The tool is implemented in python using the [langchain](https://python.langchain.com/docs/introduction/) framework,
which supports all kinds of backends for LLMs and [vector databases](https://en.wikipedia.org/wiki/Vector_database)
so that they are easily composable with the the pre- and post-processing of document data.

The [FAISS](https://github.com/facebookresearch/faiss) vector store is used to store and retrieve
text chunks of the local documents. The most significant matches are included in the prompt for a LLM query.

Vector stores require a separate embedding model to convert text sequences
to [embedding](https://en.wikipedia.org/wiki/Embedding_(machine_learning)) vectors.
The embedding models differ from LLMs in that they encode the representation for
the current state, while LLMs encode the weights used to predict the next word
for text generation.

Installation
------------
A prerequisite is the [installation of the ollama backend](https://github.com/ollama/ollama/blob/main/README.md).
Internally ollama uses the [llama.cpp](https://github.com/ggml-org/llama.cpp) implementation to work with
quantized LLMs in the [GGUF](https://github.com/ggml-org/ggml/blob/master/docs/gguf.md) file format. Ollama
runs as a service in the background. When a supported GPU is available it will automatically load models
to the GPU, when they are required by a client, and drop them after an idle timeout.

The models required by `orag` have to be downloaded once. The default configuration uses
the 'llama3.2' LLM (2 GB), and the 'nomic-embed-text' embedding model (274 MB). Other models
can be configured by command line options or a '.oragcfg' file.
```
> ollama pull llama3.2
> ollama pull nomic-embed-text
```
The `orag` script should be installed in a directory in the search-path like `~/.local/bin/`,
and the required python modules can be installed with `pip` (Note: the python3 on recent
Ubuntu & Debian versions requires either to pass the `--break-system-packages` option to pip or
to use a separate python [venv](https://pipenv.pypa.io/en/latest/installation.html) to work
around [PEP 668](https://peps.python.org/pep-0668/)).
```
> python3 -m pip install -r requirements.txt
> ./orag --version
or with venv:
> python3 -m venv .venv
> source .venv/bin/activate
(.venv) > python3 -m pip install -r requirements.txt
(.venv) > cp orag .venv/bin/
(.venv) > orag --version
(.venv) > deactivate
```
The [FAISS](https://github.com/facebookresearch/faiss) vector store is also
installed with `pip`. Since there are the two options to install a version
that runs on the CPU and another version with GPU support, it is not part of
the requirements.txt and installed separately (Note: the version running on the
CPU should be sufficient until a db grows very large).
```
> python3 -m pip install faiss-cpu
or for a version supporting CUDA-12.x for nvidia GPUs:
> python3 -m pip install faiss-gpu-cu12
```
Usage
-----
All documents that are supposed to be queried together, have first to be
added to a local db directory where the weights for the embedding model
are stored (also called 'collation' in some vector stores). [FAISS](https://github.com/facebookresearch/faiss)
is an in-memory vector store. For persistent storage it just supports 
simple serialization via load() and store() operations. The
complete db is loaded for queries and rewritten on updates.
The default is to use a '.faiss/<dbname>/' subdir in the current directory. 
```
> orag mydocs add ~/my_docs/*.txt
> orag -v test add README.md
  1: README.md                        id:8a67110b.. off:     0 len:  715 orag - run offline rag queries with ollama backe..
  2: README.md                        id:11dcc563.. off:   530 len:  508 The [FAISS](https://github.com/facebookresearch/..
  3: README.md                        id:cefc25c6.. off:  1040 len:  531 Installation.------------.A prerequisite is the ..
  [...]
> du -ab .faiss
  6036	.faiss/test/index.pkl
  21549	.faiss/test/index.faiss
> ollama ps
  NAME                       ID              SIZE      PROCESSOR    UNTIL              
  nomic-embed-text:latest    0a109f422b47    849 MB    100% GPU     4 minutes from now    
```
The default is to split the texts in chunks of up to 1000 bytes (can be changed with the `--size` option),
and a possible overlapping of 200 chars in paragraphs (can be changed with the `--overlap` option).
Each of these chunks is converted to a set of weights of the embedding model, which encodes
the content to the learned attributes of the model.


A query to the LLM first converts the query text to an embedding that is used 
in a similarity search (cosine similarity) to retrieve the 4 highest ranking
chunks stored in the FAISS db. The text of these chunks is added to the LLM
prompt to make the local knowledge available for the query (the count of chunks
to retrieve can be changed with the `--top_k` option). The embedding model used
for a query has always to be the same as used for the creation of the db.
```
> orag -v test query where is the faiss db stored?
  Chunks retrieved: 4
  1: README.md                        id:11dcc563.. score:0.73 off:   530 The [FAISS](https://github.com/facebookresearch/..
  2: README.md                        id:4e3f3acb.. score:0.73 off:  2316 > python3 -m venv .venv.> source .venv/bin/activ..
  [...]
  The FAISS database is stored in a subdirectory named '.faiss/<dbname>' in the current directory, by default.
$ ollama ps
  NAME                       ID              SIZE      PROCESSOR    UNTIL              
  llama3.2:latest            a80c4f17acd5    4.0 GB    100% GPU     4 minutes from now    
  nomic-embed-text:latest    0a109f422b47    849 MB    100% GPU     4 minutes from now    
```
The `orag` tool has to load the complete db for each query call. This is
acceptable for a local operation. The LLM query runs longer than the load
time, when not too many chunks are stored in a single db directory.

To avoid this overhead the `orag` tool also supports a `chat` command that allows
to pass multiple consecutive queries to the same FAISS db.
```
> orag test chat what is the purpose of the chat command?
```
Performance
-----------
Even if small LLMs are already usable when ollama executes them per CPU,
a local GPU like the RTX 5070 Ti with 16GB runs most queries in less than
a second (Note: the RTX 5070 Ti delivers in average twice the token rate of
a RTX 4060 Ti):
```
$ orag -v test query summarize the FAISS usage
  Here is a summary of the FAISS usage:..

  prompt tokens  867 duration 189.61 msec ( 4572.50 tok/sec)
  eval   tokens  142 duration 834.76 msec (  170.11 tok/sec)
```
The creation of embeddings with the smaller embedding model is less expensive,
and allows to index new documents quickly. Most Documents need some preprocessing
to convert them to useful input texts. The langchain framework supports additional 
document splitters for formats like pdf, but in many cases it is simple enough to
convert texts with a custom tooling. This example creates a small embedding db
for recent science news from an external source:
```
$ (mkdir scn01 && cd scn01 && time lynx -dump -listonly https://text.npr.org/1007 | head -n -7 | grep -o 'https://text.npr.org/..*' | xargs -I {} -n 1 bash -c 'echo "{} .."; lynx -dump -nolist "{}" | tail -n +5 | sed -n "/_________/q;p" > "$(basename {}).txt"')
  -> 16 seconds, 20 files, 30967 byte
$ time orag -v science add ./scn01/*.txt
  1: ./scn01/nx-s1-5411771.txt        id:d3ce0c43.. off:     0 len:  757 The European Space Agency will beam the famous '..
  2: ./scn01/nx-s1-5413025.txt        id:ebd9e61b.. off:     0 len:  460 Swimmer circumnavigates Martha's Vineyard ahead ..
  [...]
  -> 2 seconds
$ du -ab .faiss/science/
  39367       .faiss/science/index.pkl
  141357      .faiss/science/index.faiss
```
