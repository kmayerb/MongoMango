# Using MongoDB


### Big Idea 

A client can have multiple databases, databases can have multiple collections, 
collections can have multiple documents. Documents can have subdocuments


### Python API 

```python
import pymongo import MongoClient
client = MongoClient
# database name
db_name = "some_database"
db = client[db_name]

````


## Find Documents with Documents 

A query as a document. The query docuement takes the same form as the document within as collection. 
Keys can be left blank.

### Flexible Query Operators 
operators, examples:
* '$in'  - in 
* '$gt'  - greater than
* '$lte' - less than or equal
* '$ne'  - not equal

#### this but not that:

``` python
query_doc : {'attr' : 'value', 'attr2' : {'$ne': 'value'}}
db.collection.count_documents(query_doc)
db.collection.find_one(filter_doc(query_doc))
```

#### Value can take one of many posibilies:

Query all documents where < attr > is in a list of possibilies
``` python
db.collection.count_documents({
	'attr' : {
		'$in' : ['var1','var2']
		}})
```



Other Helpful Ideas

* Idea 1: Think of Mongo as a Database of Nested Dictionaries
* Idea 2: MongoDB consists of collections, which are like tables in SQL.
* Idea 3: Each DB contains admin, local, system.indices collections for record keeping purposes.
