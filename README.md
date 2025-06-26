# Stratton

Dataset from David Stratton's [Australia at the Movies](https://en.wikipedia.org/wiki/David_Stratton#Publications) with relevant [Wikidata](https://www.wikidata.org) identifiers.

### Queries

Is Wikidata ID valid? This query will return a dataframe of all entries which are no longer active, either due to redirection or deletion.

```python
import pandas
import pathlib
import requests

df = pandas.read_csv(pathlib.Path.cwd() / 'stratton.csv').head(100)
wikidata_ids = ' '.join(['wd:'+x for x in df.wikidata.unique() if type(x) == str])

query = '''
    select ?wikidata 
    where {
        values ?wikidata {'''+wikidata_ids+'''}
        ?wikidata wdt:P31 ?type
    } '''

r = requests.get('https://query.wikidata.org/sparql', params={'format': 'json', 'query': query})
df2 = pandas.DataFrame(r.json()['results']['bindings']).dropna()
df2['wikidata'] = df2['wikidata'].apply(lambda x: x['value'].split('/')[-1])
df = df.loc[~df.wikidata.isin(df2.wikidata.unique())]

print(len(df))
df.head()
```

Federate with Wikidata, here pulling in "Country of Origin" (P495) data.

```python
import pandas
import pathlib
import requests

df = pandas.read_csv(pathlib.Path.cwd() / 'stratton.csv').head(100)
wikidata_ids = ' '.join(['wd:'+x for x in df.wikidata.unique() if type(x) == str])

query = '''
    select ?wikidata ?countryLabel  
    where {
        values ?wikidata {'''+wikidata_ids+'''}
        ?wikidata wdt:P495 ?country .
        service wikibase:label { bd:serviceParam wikibase:language "en". }  
    } '''

r = requests.get('https://query.wikidata.org/sparql', params={'format': 'json', 'query': query})
df2 = pandas.DataFrame(r.json()['results']['bindings']).dropna()
df2['wikidata'] = df2['wikidata'].apply(lambda x: x['value'].split('/')[-1])
df2['countryLabel'] = df2['countryLabel'].apply(lambda x: x['value'])
df2 = df2.pivot_table(index=['wikidata'], aggfunc=lambda x: ', '.join(sorted(x.unique()))).reset_index()
df = pandas.merge(df, df2, on='wikidata', how='left')

print(len(df))
df.head()
```

Access third-party identifiers via Wikidata, here pulling in "IMDB ID" (P345) data.

```python
import pandas
import pathlib
import requests

df = pandas.read_csv(pathlib.Path.cwd() / 'stratton.csv').head(100)
wikidata_ids = ' '.join(['wd:'+x for x in df.wikidata.unique() if type(x) == str])

query = '''
    select ?wikidata ?imdb  
    where {
        values ?wikidata {'''+wikidata_ids+'''}
        ?wikidata wdt:P345 ?imdb . 
    } '''

r = requests.get('https://query.wikidata.org/sparql', params={'format': 'json', 'query': query})
df2 = pandas.DataFrame(r.json()['results']['bindings']).dropna()
df2['wikidata'] = df2['wikidata'].apply(lambda x: x['value'].split('/')[-1])
df2['imdb'] = df2['imdb'].apply(lambda x: 'https://www.imdb.com/title/'+x['value'])
df = pandas.merge(df, df2, on='wikidata', how='left')

print(len(df))
df.head()
```