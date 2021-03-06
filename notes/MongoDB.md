[§ Scheme design](https://www.mongodb.com/blog/post/6-rules-of-thumb-for-mongodb-schema-design-part-1)
--
```sh
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


§ Indexing strategy
---
 [Compound index](https://docs.mongodb.com/manual/core/index-compound/#compound-index-prefix)    
 [Remove redundant/unused](http://docs.mlab.com/indexing/#identifying-and-removing-unnecessary-indexes)

**Compound index keys order**  
If we have index `status:-1, deliveryDate:-1, customer:1`:   
It only support query whose key(s) are prefix of the indexed keys, i.e  
```sh
 ✓ status: 'CONFIRMED'  
 ✓ status: 'CONFIRMED' and deliveryDate: '20180928'

 # the order does not matter as long as it has first key 'status', still prefix
 ✓ deliveryDate: '20180928' and status: 'CONFIRMED'
 ✓ status: 'CONFIRMED' and deliveryDate: '20180928' and customer: 'A'  

 - status: 'CANCELLED' and customer: 'A' (partial support)
 ✕ deliveryDate: '20180928'  
 ✕ customer: 'A'
```

**Compound index values order**   
If we want to use index to produce sorted result, it will depend on how we sort the index values, i.e -1 (descending) or 1 (ascending).   
Say we have index `status:-1, deliveryDate:-1`:  
```
 ✓ sort({status: -1, deliveryDate: -1})
 ✓ sort({status: 1, deliveryDate: 1})  
 ✕ sort({status: -1, deliveryDate: 1})  
 ✕ sort({status: -1, deliveryDate: 1})  
```
It shows the index supports when all keys to be sorted are the same or reserved as index sorting direction. To understand this, we can visualise how index are arranged:  
```
status        deliveryDate
DELETED        20180920
DELETED        20180910
CONFIRMED      20180928
CONFIRMED      20180925
CONFIRMED      20180920
CANCELLED      20180925
```  
Clearly, we can only traverse in two directions. From top down `sort({status: -1, deliveryDate: -1})` or bottom up.

Let's have some examples to see how index affects query plan.   
**CateringOrder**
```js
{status:'CONFIRMED', sortby: {deliveryDate: 1}}
```
|                         | totalKeysExamined | totalDocsExamined | nReturned | sort in memory |
| :---------------------: | :---------------: | :---------------: | :-------: | :------------: |
|        no index         |         0         |        344        |    113    |       Y        |
|     deliveryDate_1      |        344        |        344        |    113    |       N        |
| status_1_deliveryDate_1 |        113        |        113        |    113    |       N        |

We can see with only `deliveryDate_1` index, db still has to do full scan docs (344) to return matched result which are `status: 'CONFIRMED'`, and then do another index scan (344) to only select those doc references existing in previous matched result and directly produce sorted result. In this case no need to sort again.
```js
{
  completed:{$ne:true}, deleted:{$ne:true}, status:{$ne:'CANCELLED'},
  sortby: {deliveryDate: 1}
}
```
|                         | totalKeysExamined | totalDocsExamined | nReturned | sort in memory |
| :---------------------: | :---------------: | :---------------: | :-------: | :------------: |
|        no index         |         0         |        351        |    58     |                |
| status_1_deliveryDate_1 |        351        |        351        |    58     |       N        |

We can see 'not equal' can not make use of index too much.

**CateringOrderProduct**
```js
{_p_cateringOrder: 'CateringOrder$SdFzECO4ry'}
```
|                    | totalKeysExamined | totalDocsExamined | nReturned | sort in memory |
| :----------------: | :---------------: | :---------------: | :-------: | :------------: |
|      no index      |         0         |        967        |     5     |                |
| _p_cateringOrder_1 |         5         |         5         |     5     |                |

With single index, it gains best query performance. The # of docs scanned equals to what returned.   
⍰ How it achieved only scanning 5 keys (equals to # of returned)

### Mongo shell
```sh
# connect
> mongo <domain:port>/db -u <> -p <>

# query explain
db.FoodOffer.explain('executionStats')
  .find({
    location:{$near:[18.0685808,59.3293234999],$maxDistance:0.07},
    days:'Thu', deleted:{$ne:true}
  })

# redirect mongo query result
> mongo <domain>:<port>/db -u <user> -p <> --eval "printjson(db.Offer.find({days:'Thu',available:true}).explain() )"  >> out.json

> mongoexport -h <host> -d <db> -c Order -u <user> -p <password> -o <output> --type=csv -q '{_created_at:{$gt: { "$date": "2017-01-01T00:00:00.001Z"} }}' -f "<fields>"
# export **json**
mongoexport -h <host> -d <db> -c <collection> -u <user> -p <password> -o out.json
```
Run aggregation pipeline
```sh
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

db.Order.aggregate([
  $match:{_p_cateringOrder:{$exists: true}, status:{$ne: 'CANCELLED'}}, $group:{_id:null, total:{$sum: '$price'}}
])

```
