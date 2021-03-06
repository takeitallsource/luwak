Luwak - stored query engine from Flax
=====================================

This project depends on the Flax lucene-solr-intervals fork of Lucene/Solr.
Before building, download lucene-solr-intervals from
https://github.com/flaxsearch/lucene-solr-intervals and follow the maven build
instructions.

Once that's done, you can build and deploy Luwak by running mvn install.

You can find out a bit more about how Flax use Luwak for media monitoring applications in 
this video from Lucene Revolution 2013 http://www.youtube.com/watch?v=rmRCsrJp2A8 and this video
from Berlin Buzzwords 2014 http://berlinbuzzwords.de/session/turning-search-upside-down-search-queries-documents

Running the demo
----------------

A small demo program is included in the distribution that will run queries provided
in a text file over a small corpus of documents from project gutenberg (via nltk).

```sh
./run-demo
```

Using the monitor
-----------------

Basic usage looks like this:

```java
Monitor monitor = new Monitor(new LuceneQueryParser("field"), WildcardNGramPresearcher.DEFAULT);

MonitorQuery mq = new MonitorQuery("query1", "field:text");
monitor.update(mq);

InputDocument doc = InputDocument.builder("doc1")
                        .addField(textfield, document, new StandardTokenizer(Version.LUCENE_50))
                        .build();
SimpleMatcher matches = monitor.match(doc, SimpleMatcher.FACTORY);
```

Adding queries
--------------

The monitor is updated using MonitorQuery objects, which consist of an id, a query string and an
optional highlight query string.  The monitor uses its provided MonitorQueryParser to parse the
query strings and cache query objects.

Matching documents
------------------

Queries selected by the monitor to run against an InputDocument are passed to a CandidateMatcher
class.  Three implementations are provided:
* SimpleMatcher - reports which queries matched the InputDocument
* ScoringMatcher - reports which queries matched, with their scores
* IntervalsMatcher - reports which queries matched, with the individual matches for each query

Filtering out queries
---------------------

The monitor uses a ```Presearcher``` implementation to reduce the number of queries it runs
during a ```match``` run.  Luwak comes with three presearcher implementations.

### MatchAllPresearcher

This Presearcher does no filtering whatsoever, so the monitor will run all its registered
queries against every document passed to ```match```.

### TermFilteredPresearcher

This Presearcher extracts terms from each registered query and indexes the queries against them
in the Monitor's internal index.  At match-time, the passed-in ```InputDocument``` is tokenized
and converted to a disjunction query.  All queries that match this query in the monitor's index
are then run against the document.

Only whole terms are extracted from the ```InputDocument```, so any queries that use fuzzy or
partial matching, such as RegexpQueries, are stored using a special ```AnyToken``` that matches
all documents.

### WildcardNGramPresearcher

A specialization of ```TermFilteredPresearcher``` that also extracts ngrams from ```InputDocument```s,
and matches them against exact substrings of fuzzy terms.  This presearcher trades longer document
preparation times for more exact query filtering.  Whether it is more appropriate than
```TermFilteredPresearcher``` will depend on the queries and documents being used.

Adding new query types
----------------------

```TermFilteredPresearcher``` uses a set of ```Extractor<T extends Query>``` objects to extract terms
from registered queries for indexing.  If a passed-in query does not have a specialised Extractor,
the presearcher will fall back to using a ```GenericTermExtractor```, which just uses ```Query#extractTerms(Set)```.

This will not be appropriate for all custom Query types.  You can create your own custom extractor by
subclassing ```Extractor```, and then pass it to the ```TermFilteredPresearcher``` constructor.

```java
public class CustomQueryExtractor extends Extractor<CustomQuery> {

    public CustomQueryExtractor() {
        super(CustomQuery.class);
    }

    @Override
    public void extract(CustomQuery query, List<QueryTerm> terms,
                            List<Extractor<?>> extractors) {
        terms.add(getYourTermsFromCustomQuery(query));
    }

}

Presearcher presearcher = new TermFilteredPresearcher(new CustomQueryExtractor());
```

Customizing the existing presearchers
-------------------------------------

Not all terms extracted from a query need to be indexed, and the fewer terms indexed, the
more performant the presearcher filter will be.  In general, if a BooleanQuery contains a
single MUST clause, then only the contents of that clause need be indexed.  Which clause
to index is decided by a ```TermWeightor```.

By default, the ```TermFilteredPresearcher``` and ```WildcardNGramPresearcher``` use a
```CompoundRuleWeightor``` which selects clauses based on the number of terms they contain,
how long those terms are, and whether any of them are ```ANYTOKEN``` terms.  You can also
add extra rules to suppress certain terms, or certain fields, or create your own rules
that implement ```WeightRule```.

```java
TermWeightor weightor = CompoundTermWeightor.newWeightor()
                            .addRule(new FieldWeightRule("category", 0.01f))
                            .addRule(new TermWeightRule(ImmutableMap.of("the", 0.3f)))
                            .build();
Presearcher presearcher = WildcardNGramPresearcher.builder().withWeightor(weightor).build();
```

Creating an entirely new type of Presearcher
--------------------------------------------

You can implement your own query filtering code by subclassing ```Presearcher```.  You will need
to implement ```buildQuery(InputDocument, PerFieldTokenFilter)``` which converts incoming documents into queries to
be run against the Monitor's query index, and ```indexQuery(Query)``` which converts registered
queries into a form that can be indexed.

Note that ```indexQuery(Query)``` may not create fields named '_id', '_query' or '_highlight', as these are reserved
by the Monitor's internal index.
