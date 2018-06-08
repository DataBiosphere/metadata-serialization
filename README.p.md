<img src="diagrams/ThinkTank.svg" width="280" />

# Metadata Serialization

Metadata is data about data, and for bioinformatics its critical to understand the context of a dataset. A file containing DNA sequence might get the file name "reads.sam". But what good is that? There are so many details that are crucial to performing a downstream analysis, for example, was the sequence from an animal or plant?

For [SAM](https://samtools.github.io/hts-specs/SAMv1.pdf), there is a saving grace that the file format includes some metadata. However, these metadata usually include just what's necessary to identify the sample, and perhaps some sequencer and alignment metadata. It will never include an individual's medical history, for example.

We could try to enforce that salient metadata be included in filenames, or in headers in the files themselves, however, in practice, these metadata are often stored separately either as [LIM systems](https://en.wikipedia.org/wiki/Laboratory_information_management_system), [spreadsheets](https://www.libreoffice.org/), or a combination of paper notes and experimental procedures.

We'll try to engage the problem of metadata serialization from the perspective of programmer providing bioinformatics infrastructure, and hopefully in such a way that will generalize to other fields.

## The Argument Against TSV

Since the metadata regarding a project most likely originated in a spreadsheet, why not just use TSV or CSV (tab separated or comma separated values)? All of the popular spreadsheet software can serialize and read these formats. Providing TSV metadata is an improvement over adding metadata to the filename, but let's take a moment to demonstrate some drawbacks.

### What's in a Type?

First, TSV and CSV have no immediate way to represent type. These formats represent data in tabular format, however, whether a given cell contains a string, numeric value, or enum is not immediately clear when we first look at the file. One approach might be to introspect on the cell's contents, just look at it: is it a numeric value? This approach quickly becomes problematic when we want to represent dates, times, or booleans.

A very stark example is a field that is meant to represent the sex that the sample was derived from. TSV values are strings or numbers, and so in practice sex will be represented as "FEMALE", "f", "F", "1", "n/a", "-", and so on. This not only makes it difficult to generalize parsers, but can lead to curation problems and incorrect metadata if not treated properly.

Another simple example might be representing truth or falsity, how are we to know if "false" is a string, or representing a boolean? Clearly types are useful when serializing our metadata formats.

### I Like Tables, Why Don't You?

Everyone is accustomed to tabular data and there are reasons both historical and technical why they are still favored. Spreadsheets themselves are an excellent first interface to a computer, but the spreadsheet programs themselves ditched these formats for their internal storage long ago. Aside from typing, they allow you to include macros, complex formatting, and multiple sheets per file.

Serializing tabular data as a table allows one to visually inspect the file more easily. The file itself when viewed in a text editor will often take up screen real estate usefully.

The problem is that metadata is often sparse, and hasn't been normalized. For example, imagine I perform an experiment where I take a sample with some salient characteristics, and I extract a specimen from ten different tissues, and to each of those tissues I apply ten different chemistries to arrive at a different aliquot, and I apply ten different sequencer techniques to each aliquot, each with their own important metadata, on which I perform ten different alignment methods.

Representing our experiment as a single table quickly gets out of hand! We start by creating columns for our sample. Then, we add columns for the metadata of each tissue's metadata. Then, each chemistry has a different set of parameters, which might take another ten keys each. Already, our TSV has over one hundred columns! And most of its cells are empty, since each aliquot had only a single chemistry applied to it, which is some 10 of the 100 columns we needed to add.

If we were to open this file as a csv, we would see a lot of commas, which is another way to say that when viewing the metadata as a single table, it is very sparse.

The spreadsheet solution is to make pivot tables, or foreign keys that allow you to reason about which sheet to look up an aliquot's chemical preparation, which is of course better than a single table. But then how do we guarantee relationships across multiple TSV files?

## XML is a 3-letter word!

As programmers, we often strive to make interfaces that avoid humans having to view markup. I consider it an unfortunate fact of history whenever I open an [XML](https://www.w3.org/XML/) file. However, that in no means speaks against the aspirations our web forebears had.

We can easily answer our type and sparsity woes by representing our metadata as XML. This is because we can annotate fields with hierarchical relatioships, relate fields to other fields, and gradually build up a set of tags that define a specific document. And for however much developers have come to hate XML, it of course drives [HTML](https://www.w3.org/html/) and is many programmer's first introduction to document presentation on the web (other than games and photos).

Today's generations might have lost it for the most part, but there was a time not too long ago when this programming interface was available to everyone (not just programmers), since many sites that allowed client input didn't remove HTML tags.

Now, programmers are accustomed to editing configuration in [YAML](https://en.wikipedia.org/wiki/YAML) files, or YAML Ain't Markup Language. This one does away with carets and enforces indentation to express hierarchical relationships.

So, although XML can be used to represent our metadata, its verbosity means we should aim to never expose it as a human interface. But as an extensible way for machines to interchange semantically related data, it works.

## Everyone Uses JavaScript Objects

So, to begin with, everyone uses XML, remember? The number of people who know HTML exceeds the population of many nations. Schema.org has been working to make the web more semantically useful by defining tags to add to HTML. But the data produced by Web sites quickly decoupled from the document model when they started being driven by Web services.

Although fashions change like the seasons, a general pattern has been to separate the presentation and data layers for many websites. JavaScript allowed documents to retrieve data from a server and interpret it into a document model. For that reason, more and more services and tooling were built so that they could be easily consumed by frontend JavaScript code.

[JavaScript Object Notation](http://json.org/) replaces carets for braces and adds array notation. Like XML, indentation is for readability only. We can have terse documents as our different metadata types can be represented using different Object models. As long as we are clear in defining our schema, we should be able to relate arbitrary metadata in JSON.

### Here be dragons

JSON was originally meant to be parsed by JavaScript, and JavaScript is an interpreted language compiled at runtime. XML was designed in the absence of an explicit runtime, although it found use in many Java environments. To that end, for all of the carets we've removed, we added some complexity.

For example, JavaScript as a runtime has to represent numeric values in memory somehow. Depending on the browser it can safely count up to 2^53 - 1. At that point it will overflow, and so we can't safely store large numbers as `Number` in Javascript.

For this reason, 64-bit integers in our JSON will need to be stored as strings, which means that parsers and deserializers are going to need to work around this! 

Although JSON creaks a little because of its origins, this is presented as a case for offering useful schemas. Because without careful schematization JSON adopts the same type problems as TSV (`null`, "null", `0`, "0", 'false' `false`).

### Schemas seem boring

JSON Schema is a way of defining the types that can be represented and is similar to a XML [Document Type Definition](https://en.wikipedia.org/wiki/Document_type_definition). By adding a link to the schema for an object, downstream processors can introspect on that schema before trying to go to work. If the JSON is some instance of an aliquot, the schema represents what fields are expected to be found there (and perhaps how to validate them)!

That's very boring and powerful, the same way the dictionary is boring and powerful. Yes, few would sit around reading the dictionary, but if you know your way around the dictionary it makes new things more easily readable! Writing code that introspects on schemas means writing code that makes few implicit assumptions about the data, but knowing when to make your own schema or use someone else's is still hard.

When it comes to bioinformatics, the domain moves so quickly that is difficult to get parties to agree on any fixed document model. Every year a new chemistry comes out that offers a new analytic. Developers don't want to be impeded by the need to have a fully formed data model before coding. Ontologies, which scientists maintain as controlled vocabularies around rapidly changing domains, just can't adapt (or reach consensus) fast enough. For these reasons and others, it can be difficult to get folks to think about a better way.

## Did you say Sparkly Turtles?

[JSON-LD](https://www.w3.org/TR/json-ld/) adds some vocabulary to the JSON spec. However, these are just some special key names and JavaScript runtimes treat them just like any other key. For that reason, any parser that can handle JSON works with JSON-LD. JSON-LD enables you to annotate the meaning of a given field, or its semantic context. For example, if a field will always have a gene ontology term, you would add the @context for the [Gene Ontology](http://geneontology.org/), and an $id that pointed to a specific term number there.

Examples of JSON-LD context for bioinformatics data are here in the [prefixcommons](https://github.com/prefixcommons/biocontext)!

Combining well curated controlled vocabularies with our metadata schema will allow downstream processors to further reflect on the content of our metadata. This reduces confusion and repeated curation efforts, since mapping from one schema to another requires a human in the loop.

By using JSON-LD and well formed schemas, it's possible represent our metadata as JSON-LD. JSON-LD can then be serialized as [RDF](https://www.w3.org/RDF/), stored as a turtle file, and can be queried against directly using SPARQL, or indexed into a triplet store. Triplets are statements of the form `subject, predicate, object`, SPARQL provides a query interface for working with graphical data that isn't tied to a single language.

This is the spirit behind FAIR efforts, which make data from scientific results findable, accessible, interoperable, and repeatable. The fully "Linked Open Data" stack allows one a great deal of creativity and enables future-proofing that would otherwise appear very difficult.

### Using Queries to Generate Metadata Indices

If our metadata are stored in such a way that we can write SPARQL against them, then translating from one schema to another is done by writing a query. For example, let's say we want to make a file oriented index from our JSON. Without SPARQL, we might write a simple parser that deserializes the entire JSON into memory, and then iterates over keys to assemble the document structure we want. We would then go key by key until we found the elements we wanted to merge into our resulting document.

If we got a new metadata dump, then we would have to do the same thing again, editing code that introspected on a JSON. By representing our metadata as RDF these aspects could be written as SPARQL queries, which generate the resulting index structure. This has a number of positive externalities. The first is that we can now export and import metadata without writing any code, just a query. 

Another notable benefit is that downstream clients can benefit from controlled vocabularies that include synonyms. For example, imagine our input data includes a well annotated field that contains a "tissue type," and that this tissue type is annotated from a controlled vocabulary that includes synonyms. When a client wants to build an application that makes data findable by the keyword "brain", but we indexed the term "cortical", the client can refer to the ontology before accessing our index, thereby combining careful curation with high performance indexers to powerful effect!

* [grlc.io](http://grlc.io/) turns SPARQL queries in github repositories into HTTP APIs!
* [NIH List of SPARQL Endpoints](https://wiki.nci.nih.gov/display/VKC/SPARQL+Endpoints+List+of+URLs)
* [Ontobee](http://www.ontobee.org/sparql/) list of SPARQL endpoints
* [Zooma](https://www.ebi.ac.uk/spot/zooma/) helps annotate with controlled vocabularies
* [FAIR Projector Builder](https://www.slideshare.net/markmoby/fair-projector-builder-79790642) helps make triplets from spreadsheets
* [hca-bundle-jsonld](https://github.com/simonjupp/hca-bundle-jsonld) add JSON-LD context to HCA metadata
