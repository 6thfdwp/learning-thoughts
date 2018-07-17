#### §  MongoShell CLI
```sh
# connect
> mongo ds046039-a0.qdl53.fleet.mlab.com:46039/mikdb -u <> -p <>

# query explain
db.FoodOffer.explain('executionStats')
  .find({
    location:{$near:[18.0685808,59.3293234999],$maxDistance:0.07},
    days:'Thu', deleted:{$ne:true}
  })

# export to csv
> mongoexport -h ds046039-a1.qdl53.fleet.mlab.com:46034 -d mikdb -c Order -u <USER> -p <PIN> -o ../Order.csv --type=csv -q '{_p_kitchen: "Kitchen$0c7sVFv6gK", status:{$ne:"CANCELLED"}}' --sort '{_created_at:-1}'
> mongoexport -h ds046039-a1.qdl53.fleet.mlab.com:46034 -d mikdb -c Order -u <USER> -p <PIN> -o ../Order.csv  --type=csv -q '{_created_at:{$gt:{"$date": "2017-09-01T00:00:00.001Z"},$lt:{"$date":""} }}' -f "_p_user,_p_kitchen, status,price,_created_at"

# redirect mongo query result
> mongo ds046039-a1.qdl53.fleet.mlab.com:46034/mikdb   -u <user> -p <> --eval "printjson(db.Offer.find({days:'Thu',available:true}).explain() )"  >> out.json
```

```sh
# Run aggregation pipeline
db.getCollection('_User').aggregate([
  {$match: {"username": {"$ne": null}}},
  # group by username field , add its object id to set, do counting
  {$group: {_id: "$username", uniqueIds: {$addToSet: "$_id"}, count: {$sum: 1}}},
  {$match: {count: {"$gt": 1}}},
  # map result id: [user.id set]
  {$project: {id: "$uniqueIds", username: "$_id", _id : 0} },
  {$unwind: "$id" },
])

db.Order.aggregate([
  {$match:{"paid": true}},
  {$group:{_p_user:"$_p_user", count: {$sum: 1}} },
  {$sort: {count: -1}},
  # directly save result to a new collection
  {$out: "out"}
]);
```

#### [§ Scheme design](https://www.mongodb.com//blog/post/6-rules-of-thumb-for-mongodb-schema-design-part-1)
```
• When prefer embedded objects and when not (many side contains hundreds of items, or
  each item on 'N' side need to be accessed and updated alone, this case use separate table)

• Write/read ratio when denormalizing, heavily read fields is good candidate

• Most importantly, data scheme entirely on the system data access pattern (
  how we query and update the data)
```
**Case study**   
[Model tree structure](https://docs.mongodb.com/manual/tutorial/model-tree-structures/)  
[Materialized path](http://learnmongodbthehardway.com/schema/categoryhierarchy/)  
  -- Retrieve direct children of a category  
  -- Retrieve entire subtree of a category  
  -- Find products for each level of category

[Model for multi languages](http://www.vertabelo.com/blog/technical-articles/data-modeling-for-multiple-languages-how-to-design-a-localization-ready-system)


#### § Indexing strategy
 [Compound index](https://docs.mongodb.com/manual/core/index-compound/#compound-index-prefix)    
 [Remove redundant/unused](http://docs.mlab.com/indexing/#identifying-and-removing-unnecessary-indexes)
- Offer query pattern
```
 available:{$ne:false | true}, location:{...}
 kitchen:{$eq:'kitchenId'}, available:{$ne:false}, location:{...}

 3:available:{$ne:false}, days:'Mon', location:{...}
 4:available:{$eq:true},  days:'Mon', location:{...}
```
| `totalKeysExamined`  | `totalDocsExamined`  | `nReturned`  |
| :-------------: |:-------------:|:-----:|
|     634   | 180 | 48 |
|     634   | 180 | 37 |
|  `location_2d_available_1_days_1  |
|     3:(30, )    | 4:(26, )   |    4:(13, ) |
|     4:(30, )    | 4:(4, )   |    4:(4, ) |


- FoodOffer query pattern
```
 offer:{$in:[]}, included:{$ne:false}
 food:{$eq:'foodId'}, offer:{$existed:true}, included:{$ne:false}
```
