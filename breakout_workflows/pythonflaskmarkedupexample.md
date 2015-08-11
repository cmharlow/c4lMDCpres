# OpenRefine Standard Reconciliation Service API Workflows, Examples, Templates

## Links

* FAST Reconciliation Service (not hosted, using python): https://github.com/lawlesst/fast-reconcile
  * Check 'pythonflaskmarkedupexample.md' in this workshop's GitHub repo.
* LCNAF - VIAF Reconciliation Service (hosted and not hosted, using python): https://github.com/codeforkjeff/refine_viaf
* VIAF Reconciliation Service Example (hosted, using PHP): https://github.com/rdmpage/phyloinformatics/blob/master/services/reconciliation_viaf.php
* Basic Python/Flask Template: https://github.com/mikejs/reconcile-demo/
* Extended Python/Flask Template for working with CSV files: https://github.com/okfn/helmut
* Extended Python/Flask Template: https://github.com/mblwhoi/reconciliation_service_skeleton

## Marked Up Example (reconcile.py file from Ted Lawless' FAST Reconciliation Service)

### File structure:
	* .gitignore
	* LICENSE
	* README.md <= please include documentation, use cases, links to other examples!
	* reconcile.py <= meat of the reconciliation work, shown below with comments
	* requirements.txt <= tells the local environment what is needed for this particular recon service API to work
	* text.py <= used in reconcile.py for data normalization before querying external data source

### reconcile.py:

"""
An OpenRefine reconciliation service for the API provided by
OCLC for FAST.
See API documentation:
http://www.oclc.org/developer/documentation/fast-linked-data-api/request-types
This code is adapted from Michael Stephens:
https://github.com/mikejs/reconcile-demo
"""

from flask import Flask
from flask import request
from flask import jsonify

import json
from operator import itemgetter
import urllib

#### For scoring results: The fuzzy wuzzy library is used in a number of OpenRefine Recon APIs for matching. It does text matching based off of use of the levenshtein distance metric: https://en.wikipedia.org/wiki/Levenshtein_distance.

from fuzzywuzzy import fuzz
import requests

#### This recon service uses the Flask 'microframework' to create the HTTP-based REST API. Users will use flask commands to get this API running locally, so that it can both take the HTTP requests from OpenRefine, as well as send and receive HTTP requests with the external dataset.

app = Flask(__name__)

#### some config: This is where the external data API-specific information is put in. Below, this url is made into a variable for constructing the API queries. 

api_base_url = 'http://fast.oclc.org/searchfast/fastsuggest'

#### For constructing links to FAST. The following URL is a python structure for then creating URIs for the external data API values returned. This is based off of the structure of that particular dataset - here, FAST URIs. Note that the {0} portion is a way for python to know to put in the retrieved identifers in that part of the URI.

fast_uri_base = 'http://id.worldcat.org/fast/{0}'

#### If it's installed, use the requests_cache library to cache calls to the FAST API. Caching helps improve performance in sending and receiving data from the external API. 

try:
    import requests_cache
    requests_cache.install_cache('fast_cache')
except ImportError:
    app.logger.debug("No request cache found.")
    pass

#### Helper text processing - this imports the external text.py document in this set of code for use in data normalization on the OpenRefine values before the API call. The normalized values are then sent to the external data service. Review the text.py file to get an idea of what data normalization occurs.

import text

#### Map the FAST query indexes to service types. This is where the OpenRefine Reconciliation Service structure comes into play. In order to allow different the OpenRefine user to call different FAST API indices for different reconciliation calls, an OpenRefine entity type is assigned to each, with the required metadata (id, name, index). One only needs to include the default_query for this to work with OpenRefine, however. Note that the ids below are related to how OpenRefine holds the entity types, and the indices are based on how the search indices available with the FAST API.

default_query = {
    "id": "/fast/all",
    "name": "All FAST terms",
    "index": "suggestall"
}

refine_to_fast = [
    {
        "id": "/fast/geographic",
        "name": "Geographic Name",
        "index": "suggest51"
    },
    {
        "id": "/fast/corporate-name",
        "name": "Corporate Name",
        "index": "suggest10"
    },
    {
        "id": "/fast/personal-name",
        "name": "Personal Name",
        "index": "suggest00"
    },
    {
        "id": "/fast/event",
        "name": "Event",
        "index": "suggest11"
    },
    {
        "id": "/fast/title",
        "name": "Uniform Title",
        "index": "suggest30"
    },
    {
        "id": "/fast/topical",
        "name": "Topical",
        "index": "suggest50"
    },
    {
        "id": "/fast/form",
        "name": "Form",
        "index": "suggest55"
    }
]
refine_to_fast.append(default_query)


#### Make a copy of the FAST mappings. This takes the entity types used for searching above, and makes a query object to send to the external data service. 

query_types = [{'id': item['id'], 'name': item['name']} for item in refine_to_fast]

#### Basic service metadata. There are a number of other documented options but this is all we need for a simple service. This is required for OpenRefine to recognize this standard Reconciliation service. The view is where one could put additional search within OpenRefine options, as well as constructs the URL (here, also the FAST URI) for the results.

metadata = {
    "name": "Fast Reconciliation Service",
    "defaultTypes": query_types,
    "view": {
        "url": "{{id}}"
    }
}

#### Function for making the FAST URI, which is also a URL.

def make_uri(fast_id):
    """
    Prepare a FAST url from the ID returned by the API.
    """
    fid = fast_id.lstrip('fst').lstrip('0')
    fast_uri = fast_uri_base.format(fid)
    return fast_uri

#### To be entirely honest, I just know this helps with making the HTTP requests work with JSON data. Beyond that, beyond me.

def jsonpify(obj):
    """
    Helper to support JSONP
    """
    try:
        callback = request.args['callback']
        response = app.make_response("%s(%s)" % (callback, json.dumps(obj)))
        response.mimetype = "text/javascript"
        return response
    except KeyError:
        return jsonify(obj)

#### The function where we tell our OpenRefine service how to create and handle the external data API calls.

def search(raw_query, query_type='/fast/all'):
    """
    Hit the FAST API for names.
    """
    out = []
    unique_fast_ids = []
    query = text.normalize(raw_query).replace('the university of', 'university of').strip()
    query_type_meta = [i for i in refine_to_fast if i['id'] == query_type]
    if query_type_meta == []:
        query_type_meta = default_query
    query_index = query_type_meta[0]['index']
    try:
        #### FAST api requires spaces to be encoded as %20 rather than +
        url = api_base_url + '?query=' + urllib.quote(query)
        url += '&rows=30&queryReturn=suggestall%2Cidroot%2Cauth%2cscore&suggest=autoSubject'
        url += '&queryIndex=' + query_index + '&wt=json'
        app.logger.debug("FAST API url is " + url)
        resp = requests.get(url)
        results = resp.json()
    except Exception, e:
        app.logger.warning(e)
        return out
    for position, item in enumerate(results['response']['docs']):
        match = False
        name = item.get('auth')
        alternate = item.get('suggestall')
        if (len(alternate) > 0):
            alt = alternate[0]
        else:
            alt = ''
        fid = item.get('idroot')
        fast_uri = make_uri(fid)
        #The FAST service returns many duplicates.  Avoid returning many of the
        #same result
        if fid in unique_fast_ids:
            continue
        else:
            unique_fast_ids.append(fid)
        score_1 = fuzz.token_sort_ratio(query, name)
        score_2 = fuzz.token_sort_ratio(query, alt)
        #Return a maximum score
        score = max(score_1, score_2)
        if query == text.normalize(name):
            match = True
        elif query == text.normalize(alt):
            match = True
        resource = {
            "id": fast_uri,
            "name": name,
            "score": score,
            "match": match,
            "type": query_type_meta
        }
        out.append(resource)
    #Sort this list by score
    sorted_out = sorted(out, key=itemgetter('score'), reverse=True)
    #Refine only will handle top three matches.
    return sorted_out[:3]

#### How to handle the HTTP methods here.

@app.route("/reconcile", methods=['POST', 'GET'])
def reconcile():
    #Single queries have been deprecated.  This can be removed.
    #Look first for form-param requests.
    query = request.form.get('query')
    if query is None:
        #Then normal get param.s
        query = request.args.get('query')
        query_type = request.args.get('type', '/fast/all')
    if query:
        # If the 'query' param starts with a "{" then it is a JSON object
        # with the search string as the 'query' member. Otherwise,
        # the 'query' param is the search string itself.
        if query.startswith("{"):
            query = json.loads(query)['query']
        results = search(query, query_type=query_type)
        return jsonpify({"result": results})
    # If a 'queries' parameter is supplied then it is a dictionary
    # of (key, query) pairs representing a batch of queries. We
    # should return a dictionary of (key, results) pairs.
    queries = request.form.get('queries')
    if queries:
        queries = json.loads(queries)
        results = {}
        for (key, query) in queries.items():
            qtype = query.get('type')
            #If no type is specified this is likely to be the initial query
            #so lets return the service metadata so users can choose what
            #FAST index to use.
            if qtype is None:
                return jsonpify(metadata)
            data = search(query['query'], query_type=qtype)
            results[key] = {"result": data}
        return jsonpify(results)
    # If neither a 'query' nor 'queries' parameter is supplied then
    # we should return the service metadata.
    return jsonpify(metadata)

#### Flask specific needs to define how this flask app/the API works.

if __name__ == '__main__':
    from optparse import OptionParser
    oparser = OptionParser()
    oparser.add_option('-d', '--debug', action='store_true', default=False)
    opts, args = oparser.parse_args()
    app.debug = opts.debug
    app.run(host='0.0.0.0')