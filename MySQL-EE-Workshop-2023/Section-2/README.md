# Document Store
## Demo from MySQL Shell
Using CRUD
```
mysqlsh

shell.connect('root@localhost:33060');

# Retrieve data from JSON Document
session.getSchema('world_x').getCollection('countryinfo').find().limit(2);

# Use variables and fetch data
colCI = session.getSchema('world_x').getCollection('countryinfo');

colCI.find("Name LIKE 'J%'").limit(2);

# Get Population and Surface Area of Asian countries starting with J
colCI.find("Name LIKE 'J%' AND geography.Continent = 'Asia'").fields("Name AS Name", "demographics.Population AS Continent", "geography.SurfaceArea AS SurfaceArea");

session.getSchema('workshop').getCollection('sales').find();
session.getSchema('workshop').getCollection('sales').find().limit(2);
session.getSchema('workshop').getCollection('sales').find('buyer='Paul'");

# Update
session.getSchema('workshop').getCollection('sales').modify('buyer='Paul' and _id=2").set('quantity',5).execute();

# Delete
session.getSchema('workshop').getCollection('sales').remove("buyer='Paul' and _id=2").execute();

# Create Index
session.getSchema('workshop').getCollection('sales').createIndex("buyer", {fields: [{field: '$.buyer'}]});
\sql explain select * from workshop.sales where doc->>'$.buyer'='Paul';
\sql select * from information_schema.innodb_indexes where name='buyer';

# Drop index
session.getSchema('workshop').getCollection('sales').dropIndex('buyer');
\sql select * from information_schema.innodb_indexes where name='buyer';


# Use SQL

## Get same result with SQL
# Switch to SQL mode of MySQL Shell
\sql

use world_x;

# Same result with X Dev API example
# Get Population and Surface Area of Asian countries starting with J
# Result set in table format
SELECT doc->>'$.Name' AS Name, doc->>'$.demographics.Population' AS Population, doc->>'$.geography.SurfaceArea' AS SurfaceArea FROM countryinfo WHERE doc->>'$.Name' LIKE "J%" AND doc->>'$.geography.Continent' = 'Asia';

# Get Population and Surface Area of Asian countries starting with J
# Result set in JSON
SELECT JSON_OBJECT("Name", doc->>'$.Name', "Population", doc->>'$.demographics.Population', "SurfaceArea", doc->>'$.geography.SurfaceArea') AS results FROM countryinfo WHERE doc->>'$.Name' LIKE "J%" AND doc->>'$.geography.Continent' = 'Asia';


# Comparing JSON Functions and JSON Operator ->>
SELECT countryinfo.doc->'$.Name'
	FROM countryinfo WHERE _id = "JPN";

SELECT countryinfo.doc->>'$.Name'
	FROM countryinfo WHERE _id = "JPN";

SELECT JSON_EXTRACT(countryinfo.doc, '$.Name')
	FROM countryinfo WHERE _id = "JPN";

SELECT JSON_UNQUOTE(JSON_EXTRACT(countryinfo.doc, '$.Name'))
	FROM countryinfo WHERE _id = "JPN";


## JOIN Table and JSON Document 
# Check data of each tables
SELECT * FROM countryinfo WHERE _id = "JPN";
SELECT * FROM country WHERE Code = "JPN";
SELECT * FROM city WHERE CountryCode = "JPN" AND Name = "Tokyo";

# Let's JOIN 3 tables with 2 JSON Documents inside
# "how much concentration of the population to the capital?"

# Result set in table format
SELECT
	country.Name AS "Country",
	city.Name AS "Capital", 
	round(city.Info->>'$.Population'/countryinfo.doc->>'$.demographics.Population'*100, 2) AS "Percent"
FROM
	countryinfo, city, country
WHERE
	country.Name LIKE "J%"
	AND
	countryinfo.doc->>'$.geography.Continent' = 'Asia'
    AND
	countryinfo.doc->>'$.Name'= country.Name
	AND
	country.Capital = city.ID
;

# Result set in JSON
SELECT
	JSON_OBJECT(
	"Country", country.Name,
        "Capital", city.Name,
        "Percent", round(city.Info->>'$.Population'/countryinfo.doc->>'$.demographics.Population'*100, 2)
        ) AS Results
FROM
	countryinfo, city, country
WHERE
	country.Name LIKE "J%"
	AND
	countryinfo.doc->>'$.geography.Continent' = 'Asia'
    AND
	countryinfo.doc->>'$.Name'= country.Name
	AND
	country.Capital = city.ID
;
```
