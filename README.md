# Layout-Aware-Document-Extraction-Chunking-and-Indexing
Prerequisite:
- [Amazon Bedrock model access](https://docs.aws.amazon.com/bedrock/latest/userguide/model-access.html)
- [Deploy Embedding and Text Generation Large Language Models with SageMaker JumpStart](https://docs.aws.amazon.com/sagemaker/latest/dg/jumpstart-foundation-models-use.html)
- [Create OpenSearch Domian](https://docs.aws.amazon.com/opensearch-service/latest/developerguide/createupdatedomains.html). This solution uses **IAM** as master user for fine-grained access control.

This example notebook guides you through the process of utilizing Amazon Textract's layout feature. This feature allows you to extract content from your document while maintaining its layout and reading format. Amazon Textract Layout feature is able to detect the following sections:
- Titles
- Headers
- Sub-headers
- Text
- Tables
- Figures
- List 
- Footers
- Page Numbers
- Key-Value pairs

Here is a snippet of Textract Layout feature on a page of Amazon Sustainability report using the Textract Console UI:
<img src="images/amazonsus2022.jpg" width="1000"/>

The [Amazon Textract Textractor Library](https://aws-samples.github.io/amazon-textract-textractor/index.html) is a library that seamlessly works with Textract features to aid in document processing. You can start by checking out the [examples in the documentation.](https://aws-samples.github.io/amazon-textract-textractor/notebooks/layout_analysis_for_text_linearization.html)
This notebook utilizes the Textractor library to interact with Amazon Textract and interpret its response. It enriches the extracted document text with XML tags to delineate sections, facilitating layout-aware chunking and document indexing into a Vector Database (DB). This process aims to enhance Retrieval Augmented Generation (RAG) performance.

## DOCUMENT PROCESSING AND INDEXING
<img src="images/txt layout-Page-2.jpg" width="800"/>

1. Upload multi-page document to Amazon S3.
2. Call Amazon Textract Start Document Analysis api call to extract Document Text including Layout and Tables. The response provides structured text aligned with the original document formatting and the pandas tables of each table detected in the document.
3. Enrich this extracted text further with XML tags indicating semantic sections, adding contextual metadata through the Textractor library.
4. The textrcat library extracts tables in plain text, maintaining their original layout. However, for improved processing and manipulation, it's advisable to convert them to CSV format. This method replaces the plain text tables with their CSV counterparts obtained from Textract's table feature.
5.  In this approach, the extracted text is segmented based on document title sections, the highest hierarchy level in a document. Each subsection within the title section is then chunked according to a maximum word threshold. Below outlines our approach to handling the chunking of subsection elements.:

    - **Tables:** Tables are chunked row by row until the maximum number of alphanumeric words is reached. For each table chunk, the column headers are added to the table along with the table header, typically the sentence or paragraph preceding the table in the document. This ensures that the information of the table is retained in each chunk.
    
    <img src="images/table chunkers.png" width="800" height=700/>
    
    - **List:** Chunking lists found in documents can be challenging. Naive chunking methods often split list items by sentence or newline characters. However, this approach presents issues as only the first list chunk typically contains the list title, which provides essential information about the list items. Consequently, subsequent list chunks become obsolete. In this notebook, lists are chunked based on their individual list items. Additionally, the header of the list is appended to each list chunk to ensure that the information of the list is preserved in each chunk.
    <img src="images/list chunker.png" width="800" height=700/>
    
    - **Section and subsection:** The structure of a document can generally be categorized into titles, sections, and paragraphs. A paragraph is typically the smallest unit of a document that conveys information independently, particularly within the context of a section or subsection header. In this method, text sections are chunked based on paragraphs, and the section header is added to each paragraph chunk (as well as tables and lists) within that section of the document.
    <img src="images/text chunks.png" width="800" height=700/>
    
6. Metadata is appended to each respective chunk during indexing, encompassing:
    - The entire CSV tables detected within the chunk.
    - The section header ID associated with the chunk.
    - The section title ID linked to the chunk.
    
    When retrieving a passage based on hybrid search (combining semantic and text matching), there's flexibility in the amount of content forwarded to the LLM. Some queries may necessitate additional information, allowing users to choose whether to send the corresponding chunk subsection or title section based on the specific use case.
    
## RETRIEVAL AUGMENTED GENERATION

<img src="images/rag-sect.jpg" width="800"/>

In the RAG process, we retrieve the top K=n similar passages from the Vector Database (Amazon OpenSearch Service). Subsequently, we generate a prompt template using any hierarchical section of the retrieved passages, indexed as metadata. This flexibility allows us to adjust the quantity of information forwarded to the LLM, facilitating the generation of contextual answers. 