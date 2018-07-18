[§ Scheme design](https://www.mongodb.com//blog/post/6-rules-of-thumb-for-mongodb-schema-design-part-1)
--
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


§ Indexing strategy
---
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
