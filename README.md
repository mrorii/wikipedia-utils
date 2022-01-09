# Wikipedia-Utils: Preprocessing Wikipedia Texts for NLP

This repository maintains some utility scripts for retrieving and preprocessing Wikipedia text for natural language processing (NLP) research.

**Note:**
The scripts have been created for and tested with the Japanese version of Wikipedia only.


## Preprocessed files

Some of the preprocessed files generated by this repo's scripts can be downloaded from the [Releases](https://github.com/singletongue/wikipedia-utils/releases) page.

All the preprocessed files are distributed under the [CC-BY-SA 3.0](https://creativecommons.org/licenses/by-sa/3.0/) and [GFDL](https://www.gnu.org/copyleft/fdl.html) licenses.
For more information, see the [License](#license) section below.


## Example usage of the scripts

- [Get Wikipedia page ids from a Cirrussearch dump file](#get-wikipedia-page-ids-from-a-Cirrussearch-dump-file)
- [Get Wikipedia page HTMLs](#get-wikipedia-page-htmls)
- [Extract paragraphs from the Wikipedia page HTMLs](#extract-paragraphs-from-the-wikipedia-page-htmls)
- [Make a plain text corpus of Wikipedia paragraph/page texts](#make-a-plain-text-corpus-of-wikipedia-paragraphpage-texts)
- [Make a passages file from extracted paragraphs](#make-a-passages-file-from-extracted-paragraphs)
- [Build Elasticsearch indices of Wikipedia passages/pages](#build-elasticsearch-indices-of-wikipedia-passagespages)

### Get Wikipedia page ids from a Cirrussearch dump file

#### [`get_all_page_ids_from_cirrussearch.py`](get_all_page_ids_from_cirrussearch.py)

This script extracts the [page ids](https://www.mediawiki.org/wiki/Manual:Page_table#page_id) and [revision ids](https://www.mediawiki.org/wiki/Manual:Revision_table#rev_id) of all pages from a Wikipedia Cirrussearch dump file (available from [this site](https://dumps.wikimedia.org/other/cirrussearch/).)
It also adds the following information to each item based on the information in the dump file:

- `"num_inlinks"`: the number of incoming links to the page.
- `"is_disambiguation_page"`: whether the page is a disambiguation page.
- `"is_sexual_page"`: whether the page is labeled containing sexual contents.
- `"is_violent_page"`: whether the page is labeled containing violent contents.

```sh
$ python get_all_page_ids_from_cirrussearch.py \
--cirrus_file ~/data/wikipedia/cirrussearch/20211129/jawiki-20211129-cirrussearch-content.json.gz \
--output_file ~/work/wikipedia-utils/20211129/page-ids-jawiki-20211129.json

# If you want the output file sorted by the page id:
$ cat ~/work/wikipedia-utils/20211129/page-ids-jawiki-20211129.json|jq -s -c 'sort_by(.pageid)[]' > ~/work/wikipedia-utils/20211129/page-ids-jawiki-20211129-sorted.json
$ mv ~/work/wikipedia-utils/20211129/page-ids-jawiki-20211129-sorted.json ~/work/wikipedia-utils/20211129/page-ids-jawiki-20211129.json
```

The script outputs a JSON Lines file containing following items, one item per line:

```js
{
    "title": "アンパサンド",
    "pageid": 5,
    "revid": 85364431,
    "num_inlinks": 231,
    "is_disambiguation_page": false,
    "is_sexual_page": false,
    "is_violent_page": false
}
```

### Get Wikipedia page HTMLs

#### [`get_page_htmls.py`](get_page_htmls.py)

This script fetches HTML contents of the Wikipedia pages specified by the page ids in the input file.
It makes use of [Wikimedia REST API](https://www.mediawiki.org/wiki/Wikimedia_REST_API) to accsess the contents of Wikipedia pages.

**Important:**
Be sure to check the terms and conditions of the API documented in [the official page](https://www.mediawiki.org/wiki/Wikimedia_REST_API).
Especially, you may not send more than 200 requests/sec to the API.
You should also set your contact information (e.g., email address) in the User-Agent header so that Wikimedia can contact you quickly if necessary.

```sh
# It takes about 2 days to fetch all the articles in Japanese Wikipedia
$ python get_page_htmls.py \
--page_ids_file ~/work/wikipedia-utils/20211129/page-ids-jawiki-20211129.json \
--output_file ~/work/wikipedia-utils/20211129/page-htmls-jawiki-20211129.json.gz \
--language ja \
--user_agent <your_contact_information> \
--batch_size 20 \
--mobile

# If you want the output file sorted by the page id:
$ zcat ~/work/wikipedia-utils/20211129/page-htmls-jawiki-20211129.gz|jq -s -c 'sort_by(.pageid)[]'|gzip > ~/work/wikipedia-utils/20211129/page-htmls-jawiki-20211129-sorted.json.gz
$ mv ~/work/wikipedia-utils/20211129/page-htmls-jawiki-20211129-sorted.json.gz ~/work/wikipedia-utils/20211129/page-htmls-jawiki-20211129.gz
```

The script outputs a gzipped JSON Lines file containing following items, one item per line:

```js
{
  "title": "アンパサンド",
  "pageid": 5,
  "revid": 85364431,
  "url": "https://ja.wikipedia.org/api/rest_v1/page/mobile-html/%E3%82%A2%E3%83%B3%E3%83%91%E3%82%B5%E3%83%B3%E3%83%89/85364431",
  "html": "<!DOCTYPE html><html prefix=\"dc: http://purl.org/dc/terms/ mw: http://mediawiki.org/rdf/\" ..."
}
```

### Extract paragraphs from the Wikipedia page HTMLs

#### [`extract_paragraphs_from_page_htmls.py`](extract_paragraphs_from_page_htmls.py)

This script extracts paragraph texts from a Wikipedia page HTMLs file generated by `get_page_htmls.py`.
You can specify the minimum and maximum length of the paragraph texts to be extracted.

```sh
# This produces 8,921,367 paragraphs
$ python extract_paragraphs_from_page_htmls.py \
--page_htmls_file ~/work/wikipedia-utils/20211129/page-htmls-jawiki-20211129.json.gz \
--output_file ~/work/wikipedia-utils/20211129/paragraphs-jawiki-20211129.json.gz \
--min_paragraph_length 10 \
--max_paragraph_length 1000
```

### Make a plain text corpus of Wikipedia paragraph/page texts

#### [`make_corpus_from_paragraphs.py`](make_corpus_from_paragraphs.py)

This script produces a plain text corpus file from a paragraphs file generated by `extract_paragraphs_from_page_htmls.py`.
You can optionally filter out disambiguation/sexual/violent pages from the output file by specifying the corresponding command line options.

Here we use [mecab-ipadic-NEologd](https://github.com/neologd/mecab-ipadic-neologd) in splitting texts into sentences so that some sort of named entities will not be split into sentences.

The output file is a gzipped text file containing one sentence per line, with the pages separated by blank lines.

```sh
# 22,651,544 lines from all pages
$ python make_corpus_from_paragraphs.py \
--paragraphs_file ~/work/wikipedia-utils/20211129/paragraphs-jawiki-20211129.json.gz \
--output_file ~/work/wikipedia-utils/20211129/corpus-jawiki-20211129.txt.gz \
--mecab_option '-d /usr/local/lib/mecab/dic/ipadic-neologd-v0.0.7' \
--min_sentence_length 10 \
--max_sentence_length 1000

# 18,721,087 lines from filtered pages
$ python make_corpus_from_paragraphs.py \
--paragraphs_file ~/work/wikipedia-utils/20211129/paragraphs-jawiki-20211129.json.gz \
--output_file ~/work/wikipedia-utils/20211129/corpus-jawiki-20211129-filtered-large.txt.gz \
--page_ids_file ~/work/wikipedia-utils/20211129/page-ids-jawiki-20211129.json \
--mecab_option '-d /usr/local/lib/mecab/dic/ipadic-neologd-v0.0.7' \
--min_sentence_length 10 \
--max_sentence_length 1000 \
--min_inlinks 10 \
--exclude_sexual_pages
```

#### [`make_corpus_from_cirrussearch.py`](make_corpus_from_cirrussearch.py)

This script produces a plain text corpus file by simply taking the `text` attributes of pages from a Wikipedia Cirrussearch dump file.

The resulting corpus file will be somewhat different from the one generated by `make_corpus_from_paragraphs.py` due to some differences in text processing.
In addition, since the `text` attributes in the Cirrussearch dump file does not retain the page structure, it is less flexible to modify the processing of text compared to processing an HTML file with `make_corpus_from_paragraphs.py`.

```sh
$ python make_corpus_from_cirrussearch.py \
--cirrus_file ~/data/wikipedia/cirrussearch/20211129/jawiki-20211129-cirrussearch-content.json.gz \
--output_file ~/work/wikipedia-utils/20211129/corpus-jawiki-20211129-cirrus.txt.gz \
--min_inlinks 10 \
--exclude_sexual_pages \
--mecab_option '-d /usr/local/lib/mecab/dic/ipadic-neologd-v0.0.7'
```

### Make a passages file from extracted paragraphs

#### [`make_passages_from_paragraphs.py`](make_passages_from_paragraphs.py)

This script takes a paragraphs file generated by `extract_paragraphs_from_page_htmls.py` and splits the paragraph texts into a collection of pieces of texts called passages (sections/paragraphs/sentences).

It is useful for creating texts of a reasonable length that can be handled by passage-retrieval systems such as [DPR](https://github.com/facebookresearch/DPR/).

```sh
# Make single passage from one paragraph
# 8,672,661 passages
$ python make_passages_from_paragraphs.py \
--paragraphs_file ~/work/wikipedia-utils/20211129/paragraphs-jawiki-20211129.json.gz \
--output_file ~/work/wikipedia-utils/20211129/passages-para-jawiki-20211129.json.gz \
--passage_unit paragraph \
--passage_boundary section \
--max_passage_length 400

# Make single passage from consecutive sentences within a section
# 5,170,346 passages
$ python make_passages_from_paragraphs.py \
--paragraphs_file ~/work/wikipedia-utils/20211129/paragraphs-jawiki-20211129.json.gz \
--output_file ~/work/wikipedia-utils/20211129/passages-c400-jawiki-20211129.json.gz \
--passage_unit sentence \
--passage_boundary section \
--max_passage_length 400 \
--as_long_as_possible
```

### Build Elasticsearch indices of Wikipedia passages/pages

#### Requirements

- Elasticsearch 6.x with several plugins installed

```sh
# For running build_es_index_passages.py
$ ./bin/elasticsearch-plugin install analysis-kuromoji

# For running build_es_index_cirrussearch.py (Elasticsearch 6.5.4 is needed)
$ ./bin/elasticsearch-plugin install analysis-icu
$ ./bin/elasticsearch-plugin install org.wikimedia.search:extra:6.5.4
```

#### [`build_es_index_passages.py`](build_es_index_passages.py)

This script builds an Elasticsearch index of passages generated by `make_passages_from_paragraphs`.

```sh
$ python build_es_index_passages.py \
--passages_file ~/work/wikipedia-utils/20211129/passages-para-jawiki-20211129.json.gz \
--page_ids_file ~/work/wikipedia-utils/20211129/page-ids-jawiki-20211129.json \
--index_name jawiki-20211129-para

$ python build_es_index_passages.py \
--passages_file ~/work/wikipedia-utils/20211129/passages-c400-jawiki-20211129.json.gz \
--page_ids_file ~/work/wikipedia-utils/20211129/page-ids-jawiki-20211129.json \
--index_name jawiki-20211129-c400
```

#### [`build_es_index_cirrussearch.py`](build_es_index_cirrussearch.py)

This script builds an Elasticsearch index of Wikipedia pages using a Cirrussearch dump file.
Cirrussearch dump files are originally for Elasticsearch bulk indexing, so this script simply takes the page information from the dump file to build an index.

```sh
$ python build_es_index_cirrussearch.py \
--cirrus_file ~/data/wikipedia/cirrussearch/20211129/jawiki-20211129-cirrussearch-content.json.gz \
--index_name jawiki-20211129-cirrus \
--language ja
```

## License

The content of Wikipedia, which can be obtained with the codes in this repository, is licensed under the [CC-BY-SA 3.0](https://creativecommons.org/licenses/by-sa/3.0/) and [GFDL](https://www.gnu.org/copyleft/fdl.html) licenses.

The codes in this repository are licensed under the [Apache License 2.0](https://www.apache.org/licenses/LICENSE-2.0).