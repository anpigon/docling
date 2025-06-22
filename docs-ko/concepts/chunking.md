## 소개

!!! note "청킹(Chunking) 접근 방식"

    `DoclingDocument`에서 시작하여, 원칙적으로 두 가지 가능한 청킹 접근 방식이 있습니다:

    1. `DoclingDocument`를 마크다운(또는 유사한 형식)으로 내보낸 후, 후처리 단계로 사용자 정의 청킹을 수행하는 방법.
    2. `DoclingDocument`에 직접 작동하는 네이티브 Docling 청커(chunker)를 사용하는 방법.

    이 페이지는 후자, 즉 네이티브 Docling 청커를 사용하는 방법에 대해 설명합니다.
    첫 번째 접근 방식의 예시는 마크다운 내보내기 모드를 살펴보는 [이 레시피](../examples/rag_langchain.ipynb)에서 확인할 수 있습니다.

*청커(chunker)*는 [`DoclingDocument`](./docling_document.md)를 입력받아 청크(chunk) 스트림을 반환하는 Docling의 추상화 개념입니다. 각 청크는 문서의 일부를 문자열로 담고 있으며, 관련된 메타데이터를 함께 가집니다.

Docling은 다운스트림 애플리케이션을 위한 유연성과 즉시 사용 가능한 유용성을 모두 지원하기 위해, 기본 타입인 `BaseChunker`와 특정 서브클래스를 제공하는 청커 클래스 계층을 정의합니다.

LlamaIndex와 같은 생성형 AI 프레임워크와의 Docling 통합은 `BaseChunker` 인터페이스를 통해 이루어집니다. 따라서 사용자는 내장, 자체 정의 또는 서드파티 `BaseChunker` 구현체를 쉽게 연결하여 사용할 수 있습니다.

## Base Chunker

`BaseChunker` 기본 클래스 API는 모든 청커가 다음을 제공해야 한다고 정의합니다:

- `def chunk(self, dl_doc: DoclingDocument, **kwargs) -> Iterator[BaseChunk]`: 제공된 문서에 대한 청크를 반환합니다.
- `def contextualize(self, chunk: BaseChunk) -> str`: 청크에 메타데이터를 더해 직렬화된 문자열을 반환합니다. 이는 주로 임베딩 모델(또는 생성 모델)에 입력으로 사용됩니다.

## 하이브리드 청커 (Hybrid Chunker)

!!! note "`HybridChunker` 사용하기"

    - `docling` 패키지를 사용하는 경우, 다음과 같이 가져올 수 있습니다:
        ```python
        from docling.chunking import HybridChunker
        ```
    - `docling-core` 패키지만 사용하는 경우, HuggingFace 토크나이저를 사용하려면 `chunking` extra를 설치해야 합니다. 예:
        ```shell
        pip install 'docling-core[chunking]'
        ```
        또는 Open AI 토크나이저(tiktoken)를 선호하는 경우 `chunking-openai` extra를 설치할 수 있습니다. 예:
        ```shell
        pip install 'docling-core[chunking-openai]'
        ```
        그런 다음 다음과 같이 가져올 수 있습니다:
        ```python
        from docling_core.transforms.chunker.hybrid_chunker import HybridChunker
        ```

`HybridChunker` 구현체는 하이브리드 접근 방식을 사용하며, 문서 기반의 [계층적](#hierarchical-chunker) 청킹 위에 토큰화를 고려한 세분화 작업을 적용합니다.

더 정확히 말하면:

- 계층적 청커의 결과에서 시작하여, 사용자가 제공한 토크나이저(일반적으로 임베딩 모델 토크나이저와 일치시킴)를 기반으로 다음을 수행합니다:
- 한 번의 패스(pass)에서는 필요한 경우(즉, 토큰 수 기준 초과 크기)에만 청크를 분할하고,
- 다른 패스에서는 가능한 경우(즉, 동일한 제목 및 캡션을 가진 연속적인 작은 크기의 청크)에만 청크를 병합합니다. 사용자는 `merge_peers` 매개변수(기본값 `True`)를 통해 이 단계를 비활성화할 수 있습니다.

👉 사용 예시:

- [Hybrid chunking](../examples/hybrid_chunking.ipynb)
- [Advanced chunking & serialization](../examples/advanced_chunking_and_serialization.ipynb)

## 계층적 청커 (Hierarchical Chunker)

`HierarchicalChunker` 구현체는 [`DoclingDocument`](./docling_document.md)의 문서 구조 정보를 사용하여 감지된 각 개별 문서 요소에 대해 하나의 청크를 생성합니다. 기본적으로 목록 항목만 함께 병합하며, 이 동작은 `merge_list_items` 매개변수를 통해 비활성화할 수 있습니다. 또한 헤더와 캡션을 포함한 모든 관련 문서 메타데이터를 첨부하는 작업도 처리합니다.

