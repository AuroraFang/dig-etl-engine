# Processing Data in myDIG

This guide will explain how to process data in the latest version of myDIG. This version of myDIG provides a lot of freedom and 
flexibility to users.
The guide assumes that you already read the [User Guide](index.md) and that you have already defined a project and used the 
processing pipeline to build a search engine for corpus of documents.

We will cover the following topics in this guide:

- ETK modules - a new way to process your data
- Ingesting different file types (csv, xls, xlsx, html etc)
- Defining DIG fields and their correlation to Knowledge Graph
- Tying everything together in myDIG

## ETK modules

[ETK](https://github.com/usc-isi-i2/etk) modules are python scripts, which are used by myDIG to process data. We have moved on from config based extraction pipeline
to etk modules. This is a significantly big change as compared to the previous version of myDIG. We no longer supply 
supplementary config files to define the extractions, instead the users are given flexibility to write code.

The structure of an ETK module:
```
from etk.etk_module import ETKModule
from etk.document import Document
import ...

class CustomETKModule(ETKModule):
    def __init__(self, etk):
        # initialize extractors and other relevant variables

    def process_document(self, doc: Document):
        """
          The knowledge graph is built in this function.
          Extract values/entities and add them to the Knowledge Graph, extensive examples are in the 
          etk repository.
          Depending on the file type, etk modules vary in their structure, 
          this is discussed later in this guide
        """
        
    def document_selector(self, doc) -> bool:
        """
        Boolean function for selecting document. 
        The purpose of this function is to determine whether the input record, 
        belonging to a particular dataset, should be processed by this ETK module. 
        There maybe numerous datasets in your project and each of them could be 
        processed by a dedicated ETK module
        
        Returns true or false based on some condition, for eg:
          - if url starts with certain string ?
          - if a particular field exists in the input Document?
          - if value of a particular field matches a string ?
        """
        return DefaultDocumentSelector().select_document(doc)

```
### ETK Module with an example
Let's take a look at the elicit events data from the [User Guide](index.md) and the 
corresponding etk module used to process it.

> When we imported the project settings, we imported the etk module as well.

One record from event data has the following structure:

```
  {
    "url": "http://www.ce_news_article.org/2016/01/01/201601010124.html",
    "doc_id": "4371C3A9FDB4BA949C7B895D091957E74E4903EA251AD430F39076024F647AF4",
    "raw_content" : "<html>event related html data in html structure</html>"
  }
```
> This one json line is converted to a [Document](https://github.com/usc-isi-i2/etk/blob/master/etk/document.py) 
object in myDIG and this object is passed to the `process_document` function

Lets dissect the [etk module](https://github.com/usc-isi-i2/dig-etl-engine/blob/development/datasets/etk_modules/em_elicit.py)
for this dataset.

#### Import Statements
```
# Importing the extractors we are going to use
from etk.extractors.html_content_extractor import HTMLContentExtractor, Strategy
from etk.extractors.html_metadata_extractor import HTMLMetadataExtractor
from etk.extractors.date_extractor import DateExtractor
from etk.extractors.glossary_extractor import GlossaryExtractor

# Import etk module related classes
from etk.etk_module import ETKModule
```

#### Define custom class and initialization
```
class DemoElicitETKModule(ETKModule):
    def __init__(self, etk):
        # initialize super class
        ETKModule.__init__(self, etk)
        
        # initialize metadata extractor (extracts metadata from html pages)
        self.metadata_extractor = HTMLMetadataExtractor()
        
        # initialize content extractor (extracts `content` from html page. Tries to identify main text in the html)
        self.content_extractor = HTMLContentExtractor()
        
        # initialize a date extractor (can normalize a variety of date formats)
        self.date_extractor = DateExtractor(self.etk, 'demo_date_parser')
        
        # initialize a glossary extractor for countries with a bunch of extractor level parameters
        self.country_extractor = GlossaryExtractor(
            self.etk.load_glossary(
                "${GLOSSARY_PATH}/countries.txt"),
            "country_extractor",
            self.etk.default_tokenizer,
            case_sensitive=False, ngrams=3)
            
        # initialize a glossary extractor for cities
        self.cities_extractor = GlossaryExtractor(
            self.etk.load_glossary(
                "${GLOSSARY_PATH}/cities.txt"),
            "cities_extractor",
            self.etk.default_tokenizer,
            case_sensitive=False, ngrams=3)
```
> These glossaries should be present in your project.

> Exercise: Find the glossary page in myDIG UI and see if they match the names used in this etk module

#### process_document function (this is where the Knowledge Graph is built)
```
    def process_document(self, doc):
        """
        Add your code for processing the document
        """

        raw = doc.select_segments("$.raw_content")[0]

        doc.store(doc.extract(self.content_extractor, raw, strategy=Strategy.ALL_TEXT), "etk2_text")
        doc.store(doc.extract(self.content_extractor, raw, strategy=Strategy.MAIN_CONTENT_STRICT),
                  "etk2_content_strict")
        doc.store(doc.extract(self.content_extractor, raw, strategy=Strategy.MAIN_CONTENT_RELAXED),
                  "etk2_content_relaxed")
        doc.store(doc.extract(self.metadata_extractor,
                              raw,
                              extract_title=True,
                              extract_meta=True,
                              extract_microdata=True,
                              extract_rdfa=True,
                              ), "etk2_metadata")
        doc.kg.add_value("description", json_path="$.etk2_content_strict")
        doc.kg.add_value("title", json_path="$.etk2_metadata.title")

        description_segments = doc.select_segments(jsonpath='$.etk2_content_strict')
        for segment in description_segments:
            extractions = doc.extract(extractor=self.date_extractor, extractable=segment)
            for extraction in extractions:
                doc.kg.add_value("event_date", value=extraction.value)

                extracted_countries = doc.extract(
                    self.country_extractor, segment)
                doc.kg.add_value('country', value=extracted_countries)

                extracted_cities = doc.extract(self.cities_extractor, segment)
                doc.kg.add_value('city', value=extracted_cities)

        return list()
```

