# AHED-ADS Project
Includes documentation, Jupyter notebooks and files for AHED-ADS Project of matching AHED papers to ADS papers.

## Identifying ADS Bibliographic Gaps Against NASA Astrobiology Habitable Environments Database (AHED)

In the context of the recently announced ADS Expansion Project, ADS seeks to broaden the collections’ bibliographic coverage to include Planetary Science, Heliophysics, and Earth Sciences. In collaboration with NASA’s Astrobiology Habitable Environments Database (AHED), this project was completed to assess the bibliographic holdings of AHED, identify those that have already been included in ADS’s holdings, flag those that are not yet included, and finally curate records and ingest those missing into ADS. This is one example of ADS’s goal to form collaborations with partners and expand access to scientific data and literature.

As a recent addition to the ADS Team supporting curation efforts and assisting in collection management, this appealed to my interests as I have recently begun honing my Python skills, particularly using tools such as Jupyter Notebook and OpenRefine. Jupyter Notebook is especially useful for new Python users because it helps break up scripts into more manageable blocks (cells) and I can include my notes, findings, and documentation along the way. 

In this blog post I intend to outline the goals I established, the steps I took to accomplish them, and lessons learned. To accomplish this project, I used a combination of my own knowledge and expertise, I read API Documentation, searched online for solutions as needed, and I also collaborated with team members to debug and learn the ins and outs of the ADS API.

### Project Background and Goals
The resources used in this project include an excel spreadsheet provided of AHED’s bibliographic holdings (with metadata for Authors, Org code, Title, Journal information, and DOI if available), which was split up into three sheets by Branch (Astrophysics Branch, the Planetary Systems Branch, and the Exobiology Branch). The main overall goal was to match each publication with an ADS bibcode if it exists. In order to accomplish this, my team recommended I use the ADS API to match based on DOIs, then reference strings, and then fill in the rest by Title. I split up my overall goal into three major phases: DOI, Reference Strings, and Title. Each of these phases I created a new Jupyter Notebook and outlined the sub-goals.

## Goal 1 (Notebook 1): Match AHED to ADS Items by DOI

Create a version of the AHED spreadsheet that has a column "bibcode" added to the right of the DOI column. For those publications we are able to match to ADS records, this column will list these bibcodes, otherwise "NA".

<b>Notebook # 1 Outline:</b>
<li>Step 1: Data Cleanup and Prep - combine all the excel sheets to one data set
<li>Step 2: Isolate/create list of DOIs
<li>Step 3: API Connection & Query
<li>Step 4: Match list of bibcodes to original data set

  <b>Step 1: Data Cleanup and Prep</b>
The data provided was an excel spreadsheet, split up into three tabs for AHED’s branches: the Astrophysics Branch, the Planetary Systems Branch, and the Exobiology Branch. After opening up a Jupyter notebook, I began by loading the excel spreadsheets and merging them as one comprehensive data frame.
  
  
```python
import pandas as pd

# Read excel sheets
astro_pubs = pd.read_excel("/Users/sao/Documents/Python-Projects/AHED/SpaceScienceAndAstrobiologyDivision.xlsx", sheet_name=0)
planet_pubs = pd.read_excel("/Users/sao/Documents/Python-Projects/AHED/SpaceScienceAndAstrobiologyDivision.xlsx", sheet_name=1)
exo_pubs = pd.read_excel("/Users/sao/Documents/Python-Projects/AHED/SpaceScienceAndAstrobiologyDivision.xlsx", sheet_name=2)

# Combine the excel sheets into one new/big data frame
# 862 total papers
ahed_pubs = pd.concat([astro_pubs, planet_pubs, exo_pubs], axis='index', ignore_index=True)

# Export to new excel file
ahed_pubs.to_excel("AHED/ahed_pubs.xlsx", index=False)
```

After creating a single data frame, I assessed the data provided and chose to use [OpenRefine](https://openrefine.org/) to clean up, transform, and normalize the data. OpenRefine is an open-source application for working with messy data. I recently attended training at a [Smithsonian Data Carpentries Workshop](https://datascience.si.edu/carpentries) where I learned about OpenRefine, its uses, and benefits for data cleanup. 

Using OpenRefine, I was able to transform the Journal data; normalizing publication titles, volume and issue numbers, and standardizing formatting. I even found additional DOIs that were misrepresented in the Journal field and moved them to the DOI field. In addition, I made transformations such as trimming whitespace, fixing typos as I caught them, renaming column headers, and removing duplicate entries. The transformation and deduplication process narrowed down the AHED paper list to 797 items.

  <b>Step 2: Isolate the list of DOIs</b>
  
With my cleaned up data, I imported the new excel file and isolated all the existing DOIs to prep for querying them in the ADS API.
  
  ```python
# Import new excel sheet, 'ahed_pubs_refined'
ahed_pubs_refined = pd.read_excel("AHED/ahed_pubs_refined.xlsx")

# Isolate the DOIs and drop all the papers that have no DOIs (drop null values)
ahed_dois = ahed_pubs_refined['DOI']
ahed_dois.dropna(inplace = True)

# Convert it from a data frame to a list
ahed_doi_list = ahed_dois.to_list()
print("Original paper list has", len(ahed_doi_list), "DOIs to search.")
```

    Original paper list has 177 DOIs to search.

  <b>Step 3: API Connection & Query</b>
  
Ready to search the ADS API with 177 DOIs, I established the API connection and queried my DOIs, returning the bibcodes and DOIs matched of ADS' holdings. At first, I established a basic DOI input query, where I joined all 177 DOIs in a single string, joined by 'OR' so that the ADS API would search them all at once. This worked, however it was not the most efficient due to a character limit in the q/query. In collaboration with my team, I was able to formulate a loop through my DOI list and append the response bibcodes and DOIs to a new list ('data = []').

```python
import requests
import json

# --- API REQUEST --- 
token = "<my token here>"
url = "https://api.adsabs.harvard.edu/v1/search/query?"

data=[]

for i in range(0, len(ahed_doi_list), 20):
    chunk = ahed_doi_list[i:i + 20]
    tagged = ['doi:' + d for d in chunk]
    query = ' OR '.join(tagged)
    
    params = {"q":query,"fl":"doi,bibcode","rows":200}
    headers = {'Authorization': 'Bearer ' + token}
    response = requests.get(url, params=params, headers=headers)
#     print(data.text, '\n')

    from_solr = response.json()
    if (from_solr.get('response')):
        num_docs = from_solr['response'].get('numFound', 0)
        if num_docs > 0:
            for doc in from_solr['response']['docs']:
                data.append((doc['bibcode'],doc['doi'][0]))
#     print(data)

# Create new data frame of response data
dois_matched = pd.DataFrame(data, columns = ['bibcode','DOI'])

# Export new excel sheet
dois_matched.to_excel("AHED/dois_matched.xlsx",
                  index=False)
```
  
  <b>Step 4: Match Bibcode/DOI Response Data to Original Paper List</b>
 
After the API connection successfully matched 155 existing bibcodes to the DOIs queried, my new step was to join these bibcodes on the DOIs in the AHED paper list. I joined the new data set (consisting of two columns, 'DOI' and 'BIBCODE') to the old as a left join on 'DOI'.

```python
# Merge/Join new table to original, joined on 'DOI'
merged = ahed_pubs_refined.merge(dois_matched, on='DOI', how='left')

# Export merged data
merged.to_excel("AHED/dois_matched.xlsx",
                  index=False)
```

## Goal 2 (Notebook 2): Match AHED to ADS Items by Reference Strings

My next goal was to match additional papers (without DOIs matched) by reference strings with ADS' Reference Service. The ADS Reference Service is an API endpoint that can take a query string of authors and/or journal info (publication name, volume, issue, year) and return the bibcode. 

<b>Notebook #2 Outline:</b>
<li>Step 1: Format file of papers into reference strings
<li>Step 2: Query the Reference API with reference strings, return bibcodes
<li>Step 3: Match the bibcodes back to the paper list

<b>Step 1: Format Reference List</b>
  
First I needed to prep the reference strings. In Goal 1/Notebook 1, I had transformed the data in OpenRefine to normalize the journal titles, volume numbers, and issue/id numbers. The Reference Service takes strings in the following format: [authors],[publication year],[journal name, vol, issue numbers]. So I started by making a new column, joining these metadata together and exporting the column/list to a text file.

```python
import pandas as pd
import numpy as np

# Open my excel sheet as a data frame
df = pd.read_excel("AHED/dois_matched.xlsx")

# String together the fields into single reference strings (Authors, Year, Journal)
df['REFS'] = df['AUTHORS'].astype(str) + ', ' + df['YEAR'].astype(str) + ', ' + df['JOURNAL'].astype(str)

# Grab only rows where DOI is null
dt = df[df['DOI'].isna()]

# Export my reference strings to text file
dt['REFS'].to_csv("AHED/ref_list.txt", index=False, header=False, sep='\t')
```

<b>Step 2: Connect to Reference Service API</b>

The next step was to input my reference list to the API, and return the matching bibcodes.

```python
import sys, os, io
import requests
import argparse
import json

# ADS Prod API Token
token = '<my token here>
domain = 'https://api.adsabs.harvard.edu/v1/'

## REFERENCE SERVICE ##

# --- Function to read my reference strings file and make a list called 'references'
def read_file(filename):

    references = []
    with open(filename, "r") as f:
        for line in f:
            references.append(line)
    return references

# --- Function to connect to Reference Service API, querying my 'references' list
def resolve(references):
    
    payload = {'reference': references}

    response = requests.post(
        url = domain + 'reference/text',
        headers = {'Authorization': 'Bearer ' + token,
                 'Content-Type': 'application/json',
                 'Accept':'application/json'},
        data = json.dumps(payload)
    )
    
    if response.status_code == 200:
        return json.loads(response.content)['resolved'], 200
    else:
        print('From reference status_code is ', response.status_code)
    return None, response.status_code
```

```python
# Read my reference strings file
references = read_file("/Users/sao/Documents/Python-Projects/AHED/ref_list.txt")
references = [ref.replace('\n','') for ref in references]

# Resolve my references, results in 'total results' list
total_results = []

for i in range(0, len(references), 16):
    results, status = resolve(references[i:i+16])
    if results:
        total_results += results

# Method to count how many total bibcodes were matched
bibcodes = []
for record in total_results:
    if record['bibcode']!='...................':
        bibcodes.append(record['bibcode'])

print('Matched',len(bibcodes),'bibcodes')
```

    Matched 385 bibcodes

<b>Step 3: Match Bibcode/Reference Response Data to Original Paper List</b>

After the API successfully found 385 bibcodes from the rest of the paper list (~650 papers), my next step was to match these back to the original AHED paper list and include as bibcodes matched thus far.
  
```python
# Convert my reference results to a data frame and drop null values
ref_results = pd.DataFrame(total_results)
ref_results = ref_results.replace('...................', np.nan)
ref_results = ref_results.dropna(subset=['bibcode'])

# Merge my new ref service results with my original paper list, join by the refstrings
merged = pd.merge(df, ref_results, how='left', left_on='REFS', right_on='refstring')
merged

# Combine bibcode columns, bringing new bibcode column over to the existing bibcode column
merged['BIBCODE'] = merged['BIBCODE'].fillna(merged['bibcode'])

# Cleanup; drop unneeded columns
merged = merged.drop('refstring',axis=1)
merged = merged.drop('REFS',axis=1)
merged = merged.drop('bibcode',axis=1)
merged = merged.drop('score',axis=1)
merged = merged.drop('comment',axis=1)

# Clean up nulls
merged = merged.replace(np.nan,'NA')

# Export merged data to new excel file
merged.to_excel("AHED/refs_matched.xlsx", index=False)
```
