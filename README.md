# PropSegmEnt: A Large-Scale Corpus for ***Prop***osition-Level ***Segm***entation and ***Ent***ailment Recognition

PropSegmEnt is an annotated dataset for segmenting English text into propositions, and recognizing proposition-level entailment relations --- whether a different, related document entails each proposition, contradicts it, or neither.

The dataset annotates samples of document clusters from two datasets with topically clustered documents:
[NewSHead](https://github.com/google-research-datasets/NewSHead), from the news domain, and [Wikipedia Translated Clusters](https://github.com/google-research-datasets/wiki-translated-clusters-nli), from Wikipedia.

The construction of the dataset is described in detail in our [PropSegmEnt paper preprint](https://github.com/google-research-datasets/PropSegmEnt/propsegment-preprint.pdf).

## File structure

We provide a 3-way dev/train/test split for data annotating each of the two underlying corpora as separate JSON files.

## Document cluster data

Each document cluster contains up to three documents on related topics. The document clusters come from two sources:
* The Wikipedia Translated Clusters dataset, with Wikipedia articles from the same topic in different
  languages machine-translated into English (tagged as `SOURCE_WIKI_CLUSTERS` in the data). These documents are available under CC BY-SA 3.0 from Wikipedia.
* The NewSHead dataset of clustered news articles about the same event (tagged as `SOURCE_NEWSHEAD`). These documents are not explicitly reproduced in our corpus, and must be re-crawled from the provided URLs.

Each document we used is stripped down to the main article body (using the news-please tool for NewSHead), and truncated to at most the first 10 sentences.

In this dataset, a cluster of documents is made available as a `DocumentCluster` JSON entry (for SOURCE_WIKI_CLUSTERS), or as  a `DocumentClusterRetrievalInformation` JSON entry (for SOURCE_NEWSHEAD). `DocumentCluster` has the full text from each document, along with a specific decomposition into sentences and tokens. A `DocumentClusterRetrievalInformation` entry specifies the URL the document was retrieved from, using the [news-please tool](https://github.com/fhamborg/news-please), and the timestamp at which it was retrieved. We provide a masked representation of the document text in order to simplify the task of aligning a re-crawled document to the exact sentence and tokens we used when annotating these documents.

### `DocumentCluster` JSON data dictionary

* **`source_dataset`**: String identifier of the dataset from where the
  documents in the cluster originate. Possible values are "SOURCE_WIKI_CLUSTERS"
  or "SOURCE_NEWSHEAD".
* **`cluster_id`**: 64-bit globally unique random number to identify the cluster
  internally within the dataset.
* **`documents`**: List of per-document dicts, each containing:
  * **`document_id`**: 64-bit random number to identify the document internally
    within the dataset.
  * **`document`**: String with the text of the document (UTF-8).
  * **`sentences`**: Ordered list of per-sentence dicts, each
    containing:
    * **`sentence_text`**: String with the text of the sentence (UTF-8).
    * **`tokens`**: An ordered list of dicts, each representing a
      token extracted from the sentence. Each token is a substring of
      `sentence_text`. Spacing characters are not included in tokens. We treated each contiguous stretch of alphanumeric characters as a token, as well as every non-spacing, non-alphanumeric character as a separate token.
      * **`text`**: UTF-8 text of the token. No separators on either side of the token are included.
      * **`character_offset_of_token_in_sentence`**: Unicode character offset of
        the token's first character withinin `sentence_text`. 0 if the token starts at
        the very beginning of the sentence.

### `DocumentRetrievalInstructions` JSON data dictionary

* **`source_dataset`**: String identifier of the dataset from where the
  documents in the cluster originate. Possible values are "SOURCE_WIKI_CLUSTERS"
  or "SOURCE_NEWSHEAD".
* **`cluster_id`**: 64-bit globally unique random number to identify the cluster
  internally within the dataset.
* **`documents`**: List of per-document dicts, each containing:
  * **`document_id`**: 64-bit random number to identify the document internally
    within the dataset.
  * **`original_url`**: URL from which the document was crawled.
  * **`url_crawl_timestamp`**: The document text used was at the above URL
    approximately as of this timestamp (RFC 3339 date string).
  * **`sentences`**: List of per-sentence dicts, each containing:
    * **`masked_sentence_text`**: Text of the sentence with each token modified
      by replacing every character of the token except the first character with
      an underscore ("_"). UTF-8.    
    * **`tokens`**: An ordered list of dicts, each representing a
      token extracted from the sentence.
      * **`masked_text`**: Masked text of the token. Only
        the first Unicode character is unmasked, each remaining Unicode
        character is replaced with underscore ("_"). UTF-8.
      * **`character_offset_of_token_in_sentence`**: Unicode character offset of
        the token's first character in `sentence_text`. 0 if the token starts at
        the very beginning of the sentence.


##  `PropositionSegmentation` JSON data dictionary

A `PropositionSegmentation` JSON entry represents a sentence segmented by raters into zero or more propositions.

* **`sentence_id`**: A dict uniquely identifying the sentence within
  the dataset, containing:
  * **`cluster_id`**: Id of the document cluster this sentence is in. Matches
    the `cluster_id` of a `DocumentCluster` **or**
    `DocumentClusterRetrievalInformation` JSON entry.
  * **`document_id`**: Id of the document this sentence is in, matching a
    `document_id` field within a `DocumentCluster` **or**
    `DocumentClusterRetrievalInformation` JSON entry.
  * **`sentence_index`**: Sequential 0-based index of the sentence within the
    document's sentences list in the `DocumentCluster` **or**
    `DocumentClusterRetrievalInformation` JSON entry.
* **`segmentation_responses`**: List containing 1 or more dicts, one per distinct rater, with:
  * **`non_informational`**: Boolean value indicating whether the
    rater marked this sentence as non-informational and/or malformed.
  * **`rater_did_not_understand_sentence`**: Boolean value indicating whether
    the rater marked that they could not understand this sentence.
  * **`propositions`**: List of zero or more propositions, each represented as a dict of:
    * **`included_token_indices`**: List of 1 or more tokens included in the
      proposition. The tokens are represented as indices into the `tokens` field
      of the sentence.
  * **`is_reference_response`**: Boolean value representing whether this
    response  the "reference" response for this
    sentence, based on the reconciliation heuristic described in the paper. Set to
    true for at most one `segmentation_responses` entry per sentence.

## Document-to-proposition entailment

For some pairs of documents within a cluster, we provide 
raters' annotation for the entailment relationship, if any, between one document as a premise, and propositions found in all the sentences of the other document as hypotheses. A `PropositionEntailment` JSON entry describes the raters response for a single pair of a premise document and a hypothesis proposition.

### `PropositionEntailment` JSON data dictionary

* **`cluster_id`**: Id of the document cluster the proposition and premise
  document are in. Matches the `cluster_id` of a `DocumentCluster` **or**
  `DocumentClusterRetrievalInformation` JSON entry.
* **`premise_document_id`**: Document id of the premise document, matching a
  `document_id` in a `DocumentCluster` **or** `DocumentClusterRetrievalInformation`
  JSON entry.
* **`hypothesis_proposition`**: Information to identify the hypothesis
  proposition, as a dict of:
  * **`document_id`**: Document id of the containing document, matching a
    `document_id` of a `DocumentCluster` **or** 
    `DocumentClusterRetrievalInformation` JSON entry.
  * **`sentence_index`**: Sequential 0-based index of the sentence within the
    document's sentences list in the `DocumentCluster` **or**
    `DocumentClusterRetrievalInformation` JSON entry.
  * **`proposition`**: Dict describing the hypothesis proposition. Note, this is
    **always** a proposition stored in a `PropositionSegmentation` json entry. Contains:
    * **`included_token_indices`**: List of 1 or more token indices within the
      sentence included in the proposition.
* **`entailment_responses`**: List containing 1 or more entries,
  one per annotator response, each a dict with:
  * **`rater_did_not_understand_premise`**: Boolean value indicating whether the
    annotator marked that they could not understand the premise document.
  * **`rater_did_not_understand_hypothesis`**: Boolean value indicating whether
    the annotator marked that they could not understand the hypothesis
    proposition.
  * **`entailment_status`**: Entailment relationship from the premise document
    to the hypothesis proposition. Has one the following values:
    "RELATIONSHIP_ENTAILS", "RELATIONSHIP_CONTRADICTS", or
    "RELATIONSHIP_NEITHER_ENTAILS_NOR_CONTRADICTS".
  * **`supporting_premise_proposition`**: If `entailment_status` is
    "RELATIONSHIP_ENTAILS" or "RELATIONSHIP_CONTRADICTS", describes the
    best-matched proposition in the premise document that either entails or
    contradicts the hypothesis; otherwise, not set. If set, this field is a
    dict containing:
    * **`sentence_index`**: Sentence index in the premise document where the supporting
      proposition is located.
    * **`proposition`**: The supporting proposition itself. Note,
      this proposition **sometimes does not** appear in a `PropositionSegmentation`
      JSON entry. Dict containing:
      * **`included_token_indices`**: List of 1 or more token indices within the
        sentence included in the proposition.
