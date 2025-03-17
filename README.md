# RDFox-Solr Integration with Accent-Insensitive Search

This repository demonstrates how to integrate RDFox with Apache Solr, focusing on accent-insensitive (diacritics-insensitive) search capabilities. The system allows you to store data in RDFox, index it in Solr, and perform searches that ignore accents in text.

## Requirements

- [RDFox](https://www.oxfordsemantic.tech/product) (License required)
- [Apache Solr](https://solr.apache.org/) (v9.4.1 or later)
- `curl` for making HTTP requests
- Bash or similar shell environment

## Setup Overview

The integration follows these steps:
1. Set up a Solr collection with accent-insensitive search configuration
2. Export literals from RDFox to Solr
3. Query Solr from RDFox using a tuple table

## Step-by-Step Setup Instructions

### 1. Set up Solr with Accent-Insensitive Search

```bash
# Start Solr
~/solr-9.4.1/bin/solr start

# Create a new collection called 'accents'
~/solr-9.4.1/bin/solr create -c accents

# Replace the default managed-schema with our custom one
cp ./managed-schema.xml ~/solr-9.4.1/server/solr/accents/conf/managed-schema.xml

# Restart Solr to apply the new schema
~/solr-9.4.1/bin/solr restart
```

The custom `managed-schema.xml` includes a field type called `my_text` that uses `ASCIIFoldingFilterFactory` to handle accent-insensitive search:

```xml
<fieldType name="my_text" class="solr.TextField">
  <analyzer>
    <tokenizer class="solr.WhitespaceTokenizerFactory"/>
    <filter class="solr.LowerCaseFilterFactory"/>
    <filter class="solr.ASCIIFoldingFilterFactory"/>
  </analyzer>
</fieldType>
```

### 2. Load Data into Solr

```bash
# Load the literals CSV into Solr
curl 'http://localhost:8983/solr/accents/update?commit=true' --data-binary @literals.csv -H 'Content-type: text/csv'
```

### 3. Start RDFox with Solr Integration

```bash
# Run the RDFox start script
/path/to/RDFox sandbox . start.rdfox
```

The `start.rdfox` script:
- Creates a datastore
- Imports the RDF data 
- Registers Solr as a data source
- Creates a tuple table that RDFox will use to query Solr

## Querying with Accent-Insensitive Search

You can now query the system using SPARQL at http://localhost:12110/console/datastores/sparql?datastore=accents. For example:

```sparql
SELECT ?prod ?actualLabel ?searchLabel
WHERE {
    VALUES ?searchLabel {"besta"}
    TT products_fti_tt {?prod ?actualLabel ?searchLabel}
}
```

This query will find products with labels like "BESTÅ" even when searching for "besta" (without accents).

## Updating Data

RDFox and Solr will need to be maintained up to date separately in this demo.
It is also possible to use [Delta Queries](https://docs.oxfordsemantic.tech/transactions.html#id5) to help keep this in sync.


We can add an extreme example. The following query will find products with labels like "Hàålfoørs" even when searching for "hallfors" (without accents).
```sparql
SELECT ?prod ?actualLabel ?searchLabel
WHERE {
    VALUES ?searchLabel {"hallfors"}
    TT products_fti_tt {?prod ?actualLabel ?searchLabel}
}
```

To add the relevant data, follow the following steps

```bash
# Add new data to RDFox
curl -X POST "localhost:12110/datastores/accents/sparql" --data "update=INSERT DATA {:product_00 rdfs:label \"Hàålfoørs\"}"

# Fetch updated labels from RDFox
curl -X POST "localhost:12110/datastores/accents/sparql" --data "query=SELECT ?item ?label WHERE {?item rdfs:label ?label}" --header "Accept: text/csv" --output ~/accents/literals_update.csv

# Clear Solr collection
curl 'http://localhost:8983/solr/accents/update?commit=true' --header "Content-Type: text/xml" --data-binary '<delete><query>*:*</query></delete>'

# Reload Solr collection with updated data
curl 'http://localhost:8983/solr/accents/update?commit=true' --data-binary @literals_update.csv --header 'Content-type: text/csv'
```

```sparql
SELECT ?prod ?actualLabel ?searchLabel
WHERE {
    VALUES ?searchLabel {"hallfors"}
    TT products_fti_tt {?prod ?actualLabel ?searchLabel}
}
```



## How It Works

1. **Data Storage**: RDFox stores the semantic data with full Unicode support
2. **Text Analysis**: Solr's `ASCIIFoldingFilterFactory` converts accented characters to their ASCII equivalents during indexing
3. **Integration**: RDFox's connects (via tuple table) to Solr via HTTP
4. **Query Processing**: Searches in Solr are accent-insensitive, and results are returned to RDFox, and then used as part of a broader SPARQL query

## Example Use Case: IKEA Product Search

This repository uses IKEA product data as an example. Users can search for products using terms without worrying about accents in product names. For example, searching for "besta" will find products labeled as "Bestå".
