# PostgreSQL JSONB flattening
### JSONB [subscripting](https://www.postgresql.org/docs/current/datatype-json.html#JSONB-SUBSCRIPTING) (PG14+) syntax ###  
> [!NOTE]
> [Arrow syntax](https://www.postgresql.org/docs/current/functions-json.html#FUNCTIONS-JSON-PROCESSING) (`object->'attribute'`, `object->>'attribute'`) for versions prior to PG14 or as a matter of personal taste  
```sql
create table the_table (
 id integer,
 name text,
 details jsonb
);
```
**JSONB object per record**
------
```sql
insert into the_table 
values
(1, 'one',   '{"x":101, "y":201, "z":false, "more":{"weight":23.5, "created":"2023-12-10"}}'),
(2, 'two',   '{"x":102, "y":202, "z":true, "more":{"weight":8.72}}'),
(3, 'three', '{"x":103, "y":203, "more":{"created":"2024-01-20"}}');
```
* **Using select list expressions only**
```sql
select id, 
       name, 
       details['x']::numeric as length, 
       details['y']::numeric as width, 
       details['z']::boolean as is_available,
       details['more']['weight']::numeric as weight,
       details['more']['created']::text::date as created
from the_table;
```
> [!NOTE]
> Note the cast of jsonb `details['more']['created']` first to text and then to date as a cast of jsonb to date does not exist.
> `details->'more'->>'created'` is text and can be cast to date directly.

* **Using JSONB_TO_RECORD**
```sql
select id, 
       name,
       j.x as length,
       j.y as width,
       j.z as is_available,
       j.more['weight']::numeric as weight,
       j.more['created']::text::date as created
from the_table
 cross join lateral jsonb_to_record(details) as j(x numeric, y numeric, z boolean, more jsonb);
```
|id|name|length|width|is_available|weight|created   |
|--|----|------|-----|------------|------|----------|
|    1|one    |   101|  201|false       |  23.5|2023-12-10|
|    2|two    |   102|  202|true        |  8.72|          |
|    3|three  |   103|  203|            |      |2024-01-20|  
    
**Deeply nested data. JSONB array per record**
------
```sql
truncate the_table;
insert into the_table 
values
(1, 'one', '[{"x":101, "y":201, "z":false, "more":{"weight":23.5, "even_more":{"created":"2023-12-10"}}},
             {"x":102, "y":202, "z":true, "more":{"weight":8.72}},
             {"x":103, "y":203, "more":{"even_more":{"created":"2024-01-20"}}}]'
),
(2, 'two', '[{"x":111, "y":211, "z":true, "more":{"weight":123.5, "even_more":{"created":"2023-12-11"}}},
             {"x":112, "y":212, "z":false, "more":{"weight":18.72, "location_ids":[91, 92, 93]}},
             {"x":113, "y":233, "more":{"even_more":{"created":"2024-01-21"}}}]'
);
```
* **Using JSONB_ARRAY_ELEMENTS**
```sql
select id, 
       name, 
       jdata['x']::numeric as length, 
       jdata['y']::numeric as width,
       case when jdata['z'] is null then 'Do not count on it'
            else case when jdata['z']::boolean then 'Yes' else 'Unfortunately not' end
       end as available,
       jdata['more']['weight']::numeric as weight,
       jdata['more']['even_more']['created']::text::date as created,
       loc::integer as loc_id
from the_table 
 cross join lateral jsonb_array_elements(details) as jdata
 left  join lateral jsonb_array_elements_text(jdata['more']['location_ids']) as l(loc) on true
order by id, weight nulls first;
```
* **Using JSONB_TO_RECORDSET**
```sql
select id, 
       name, 
       x as length, 
       y as width, 
       case when z is null then 'Do not count on it'
            else case when z then 'Yes' else 'Unfortunately not' end
       end as available,
       more['weight']::numeric as weight,
       more['even_more']['created']::text::date as created,
       loc::integer as loc_id
from the_table 
 cross join lateral jsonb_to_recordset(details) as (x numeric, y numeric, z boolean, more jsonb)
 left  join lateral jsonb_array_elements_text(more['location_ids']) as l(loc) on true
order by id, weight nulls first;
```
  
|id|name|length|width|available         |weight|created   |loc_id|
|-|-|-|-|-|-|-|-|
|    1|one    |   103|  203|Do not count on it|      |2024-01-20|   |
|    1|one    |   102|  202|Yes               |  8.72|          |   |
|    1|one    |   101|  201|Unfortunately not |  23.5|2023-12-10|   |
|    2|two    |   113|  233|Do not count on it|      |2024-01-21|   |
|    2|two    |   112|  212|Unfortunately not | 18.72|          | 91|
|    2|two    |   112|  212|Unfortunately not | 18.72|          | 92|
|    2|two    |   112|  212|Unfortunately not | 18.72|          | 93|
|    2|two    |   111|  211|Yes               | 123.5|2023-12-11|   |

> [!NOTE]
> Verbose `cross join lateral` used for explicitness, can be [replaced by a comma](https://www.postgresql.org/docs/current/queries-table-expressions.html#QUERIES-LATERAL) 

