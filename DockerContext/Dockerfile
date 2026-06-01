FROM python:3.10-buster

RUN set -ex \
    && git clone https://github.com/EricBoix/jj_build_knowledge_graph.git \
    # Rename archive for clear context when debugging/fixing in container
    && mv jj_build_knowledge_graph/ jj_build_knowledge_graph.git

WORKDIR jj_build_knowledge_graph.git
RUN pip install --no-cache-dir -r requirements.txt

ENTRYPOINT ["python3"]
