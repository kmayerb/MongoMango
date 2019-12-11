# Mongo - Joining Information Across Collections

### Example Subjects Table


```python
subjects = \
[{"_id": 1, 
  "subject_id": 1000, 
 "special_status" : 'control', 
 "sex" : None},
{"_id": 2, 
 "subject_id": 1001, 
 "special_status" : 'control', 
 "sex" : None},
{"_id": 3, 
"subject_id": 1025, 
 "special_status" : 'POS', 
 "sex" : "Female"},
{ "_id": 4, 
"subject_id": 1026, 
 "special_status" : 'POS', 
 "sex" : "Male"}]
```

### Example Samples Table


```python
samples = \
[{"_id": 1, 
  "subject_id": 1025, 
 "sample_id" : 1},
 {"_id": 2, 
  "subject_id": 1025, 
 "sample_id" : 2},
 {"_id": 3, 
  "subject_id": 1025, 
 "sample_id" : 3},
 {"_id": 4, 
  "subject_id": 1026, 
 "sample_id" : 4},
 {"_id": 5, 
  "subject_id": 1026, 
 "sample_id" : 5},
 {"_id": 6, 
  "subject_id": 1000, 
 "sample_id" : 6}]
```

### Create the Database 


```python
from pymongo import MongoClient
# open connection to local db
myclient = MongoClient('localhost', 27017) 
# create a new database called trees
mydb = myclient["example"]
mycol = mydb['subjects']
mycol.insert_many(subjects)
mycol = mydb['samples']
mycol.insert_many(samples)
mycol = mydb['samples']
```


```python
[print("{}".format(x)) for x in mydb.samples.find({})]
```

    {'_id': 1, 'subject_id': 1025, 'sample_id': 1}
    {'_id': 2, 'subject_id': 1025, 'sample_id': 2}
    {'_id': 3, 'subject_id': 1025, 'sample_id': 3}
    {'_id': 4, 'subject_id': 1026, 'sample_id': 4}
    {'_id': 5, 'subject_id': 1026, 'sample_id': 5}
    {'_id': 6, 'subject_id': 1000, 'sample_id': 6}



```python
[print("{}".format(x)) for x in mydb.subjects.find({})]
```

    {'_id': 1, 'subject_id': 1000, 'special_status': 'control', 'sex': None}
    {'_id': 2, 'subject_id': 1001, 'special_status': 'control', 'sex': None}
    {'_id': 3, 'subject_id': 1025, 'special_status': 'POS', 'sex': 'Female'}
    {'_id': 4, 'subject_id': 1026, 'special_status': 'POS', 'sex': 'Male'}


```python
mycol = mydb['subjects']
[print("{}".format(x)) for x in mycol.find({"special_status":"POS"})]
```

    {'_id': 3, 'subject_id': 1025, 'special_status': 'POS', 'sex': 'Female'}
    {'_id': 4, 'subject_id': 1026, 'special_status': 'POS', 'sex': 'Male'}


## Joining Subjects

Let's say we want all sample_ids associated with each subject. This is like a one to many join in SQL.

### Very Naive Approach

1. Here we simply query all the unique subject_ids that match our query.
2. Then we use a '$in' query to get all samples associated with those subject_ids


```python
valid_subjects_ids = mydb.subjects.find({"special_status":"POS" }).distinct('subject_id')
print(valid_subjects_ids)
```

    [1025, 1026]



```python
[print("{}".format(x)) for x in mydb.samples.find({"subject_id": {'$in' : valid_subjects_ids}})]
```

    {'_id': 1, 'subject_id': 1025, 'sample_id': 1}
    {'_id': 2, 'subject_id': 1025, 'sample_id': 2}
    {'_id': 3, 'subject_id': 1025, 'sample_id': 3}
    {'_id': 4, 'subject_id': 1026, 'sample_id': 4}
    {'_id': 5, 'subject_id': 1026, 'sample_id': 5}




### More MongoPhonic Approach

https://docs.mongodb.com/manual/reference/operator/aggregation/lookup/

The equivalent of left join is a lookup. This allows for one to many grab of all the records associated with a match btween the localField and the foreignFields


```python
pipeline =  [
   {"$match" :{"special_status":"POS"}}, 
   {"$lookup" : {
       'from': 'samples',
       'localField': 'subject_id',
       'foreignField': 'subject_id',
       'as': "sample_ids" 
     }},
    {'$unwind': '$sample_ids'},
    { "$project": {
          "_id": 1,
          "subject_id": 1,
          "special_status": 1,
          "sex": 1,
          "sample_ids.sample_id": 1 }}
]
```


```python
[print("{}".format(x)) for x in mydb.subjects.aggregate(pipeline)]
```

    {'_id': 3, 'subject_id': 1025, 'special_status': 'POS', 'sex': 'Female', 'sample_ids': {'sample_id': 1}}
    {'_id': 3, 'subject_id': 1025, 'special_status': 'POS', 'sex': 'Female', 'sample_ids': {'sample_id': 2}}
    {'_id': 3, 'subject_id': 1025, 'special_status': 'POS', 'sex': 'Female', 'sample_ids': {'sample_id': 3}}
    {'_id': 4, 'subject_id': 1026, 'special_status': 'POS', 'sex': 'Male', 'sample_ids': {'sample_id': 4}}
    {'_id': 4, 'subject_id': 1026, 'special_status': 'POS', 'sex': 'Male', 'sample_ids': {'sample_id': 5}}


### Look at what each part of the pipeline does:

1. `$match` is a simple query
2. `$lookup` finds a docuemnts in one collection with a matching key.
3. `$unwind`, makes a new document for each unique value of the specified feature
4. `$project`, subsets what is returned, in this case to only return sample_id and ignore the other elements of the document


```python
pipeline2 =  [
   {"$match" :{"special_status":"POS"}}, 
   {"$lookup" : {
       'from': 'samples',
       'localField': 'subject_id',
       'foreignField': 'subject_id',
       'as': "sample_ids" 
     }},
    {'$unwind': '$sample_ids'}
]
```


```python
[print("{}".format(x)) for x in mydb.subjects.aggregate(pipeline2)]
```

    {'_id': 3, 'subject_id': 1025, 'special_status': 'POS', 'sex': 'Female', 'sample_ids': {'_id': 1, 'subject_id': 1025, 'sample_id': 1}}
    {'_id': 3, 'subject_id': 1025, 'special_status': 'POS', 'sex': 'Female', 'sample_ids': {'_id': 2, 'subject_id': 1025, 'sample_id': 2}}
    {'_id': 3, 'subject_id': 1025, 'special_status': 'POS', 'sex': 'Female', 'sample_ids': {'_id': 3, 'subject_id': 1025, 'sample_id': 3}}
    {'_id': 4, 'subject_id': 1026, 'special_status': 'POS', 'sex': 'Male', 'sample_ids': {'_id': 4, 'subject_id': 1026, 'sample_id': 4}}
    {'_id': 4, 'subject_id': 1026, 'special_status': 'POS', 'sex': 'Male', 'sample_ids': {'_id': 5, 'subject_id': 1026, 'sample_id': 5}}
`



```python
pipeline3 =  [
   {"$match" :{"special_status":"POS"}}, 
   {"$lookup" : {
       'from': 'samples',
       'localField': 'subject_id',
       'foreignField': 'subject_id',
       'as': "sample_ids" 
     }},
]
```


```python
[print("{}".format(x)) for x in mydb.subjects.aggregate(pipeline3)]
```

    {'_id': 3, 'subject_id': 1025, 'special_status': 'POS', 'sex': 'Female', 'sample_ids': [{'_id': 1, 'subject_id': 1025, 'sample_id': 1}, {'_id': 2, 'subject_id': 1025, 'sample_id': 2}, {'_id': 3, 'subject_id': 1025, 'sample_id': 3}]}
    {'_id': 4, 'subject_id': 1026, 'special_status': 'POS', 'sex': 'Male', 'sample_ids': [{'_id': 4, 'subject_id': 1026, 'sample_id': 4}, {'_id': 5, 'subject_id': 1026, 'sample_id': 5}]}


