# ROE (Return On Experiment) of failed Langtrace SDK usage

This is a quick report on failing to use Langtrace Python SDK in order to display the trace of llm calls ([spans](https://docs.langtrace.ai/concepts#span) in Langtrace lingo) on the console (as opposed to sending them to some Langtrace remote server).

## What was done

Following lines added to `graph_util.py`:

```python
from langtrace_python_sdk import langtrace
# Following line should be added before any other langchain related initilization code
langtrace.init(write_spans_to_console=True, api_key="dummy_api_key")
```

and following lines added to `requirements.txt`

```bash
langtrace_python_sdk
importlib_metadata  # Required by langtrace_python_sdk (at runtime)
```

## What you get

Consider the following execution (python version 3.10.19) that runs smoothly without using `langtrace`:

```bash
 python extracting_graph.py --input_directory jj_doc_Four_Noble_Truths.git/original_data/ --load_markdown_document 250_BCE_-_Dhammacakkappavattana_Sutta_Four_Noble_Truths_Wikipedia_translation.md
```

When trying to execute the exact same call after the above described `langtrace_python_sdk` instrumentation, then one first gets the following warnings

```bash
<some_dir>/langtrace_python_sdk/utils/__init__.py:94: UserWarning: pkg_resources is deprecated as an API. See https://setuptools.pypa.io/en/latest/pkg_resources.html. The pkg_resources package is slated for removal as early as 2025-11-30. Refrain from using this package or pin to Setuptools<81.
  import pkg_resources
Skipping langchain due to error while instrumenting: No module named 'langchain.agents.agent'
Skipping langchain-community due to error while instrumenting: No module named 'langchain_community.vectorstores.pinecone'
```

Then some [spans](https://langcrew.ai/concepts/trace/#problem-no-traces-are-uploaded-after-execution), as expected, but eventually the following **unresolved** error is encountered

```bash
 File "[...]/jejune_extract_knowledge_graph/venv/lib/python3.10/site-packages/pydantic/main.py", line 263, in __init__
    validated_self = self.__pydantic_validator__.validate_python(data, self_instance=self)
pydantic_core._pydantic_core.ValidationError: 1 validation error for LLMSpanAttributes
`gen_ai.request.response_format`
  Input should be a valid string 
```

