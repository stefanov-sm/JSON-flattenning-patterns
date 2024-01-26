# JSON-flattenning-patterns
### JSONB flattening in PostgreSQL ###
```sql
create table the_table (
 tt_id integer,
 tt_name text,
 details jsonb
);
insert into the_table 
values
(1, 'one',   '{"x":101, "y":201, "z":false, "more":{"weight":23.5, "created":"2023-12-10"}}'),
(2, 'two',   '{"x":102, "y":202, "z":true,  "more":{"weight":8.72}}'),
(3, 'three', '{"x":103, "y":203, "more":{"created":"2024-01-20"}}');
```
#### Using JSONB arrow operators ####
```sql
select tt_id, 
       tt_name, 
       (details->>'x')::numeric as length, 
       (details->>'y')::numeric as width, 
       (details->>'z')::boolean as is_available,
       (details->'more'->>'weight')::numeric as weight,
       (details->'more'->>'created')::date as created
from the_table;
```
#### Using JSONB [substripting](https://www.postgresql.org/docs/current/datatype-json.html#JSONB-SUBSCRIPTING) (PG14+) and JSONB_TO_RECORD ###
```sql
select tt_id, 
       tt_name, 
       x as length,
       y as width,
       z as is_available,
       more['weight']::numeric as weight,
       more['created']::text::date as created
from the_table
 cross join lateral jsonb_to_record(details) as (x numeric, y numeric, z boolean, more jsonb);
```
More "curly" JSON:

    [
     {"x":111, "y":211, "z":true, "more":{"weight":123.5, "even_more":{"created":"2023-12-11"}}},
     {"x":112, "y":212, "z":false, "more":{"weight":18.72, "locations":[91, 92, 93]}},
     {"x":113, "y":233, "more":{"even_more":{"created":"2024-01-21"}}}
    ]

```sql
truncate the_table;
insert into the_table 
values
(1, 'one', '[{"x":101, "y":201, "z":false, "more":{"weight":23.5, "even_more":{"created":"2023-12-10"}}},
             {"x":102, "y":202, "z":true,  "more":{"weight":8.72}},
             {"x":103, "y":203, "more":{"even_more":{"created":"2024-01-20"}}}]'
),
(2, 'two', '[{"x":111, "y":211, "z":true, "more":{"weight":123.5, "even_more":{"created":"2023-12-11"}}},
             {"x":112, "y":212, "z":false,  "more":{"weight":18.72, "location_ids":[91, 92, 93]}},
             {"x":113, "y":233, "more":{"even_more":{"created":"2024-01-21"}}}]'
);
