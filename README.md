
# ATMS


## Data

1] ATMS

ATMS: https://locatiewijzer.geldmaat.nl/nl/

To GeoJSON formaat see [atms.geojson](./data/atms.geojson)

2] Buurten

Source: 

https://www.cbs.nl/nl-nl/dossier/nederland-regionaal/geografische-data/wijk-en-buurtkaart-2024

Download Geopackage wijken.gpkg

Convert to EPSG::4326

```
ogr2ogr -f GPKG -t_srs EPSG:4326 -nlt PROMOTE_TO_MULTI -lco GEOMETRY_NAME=geom -update -overwrite wijken_4326.gpkg wijken.gpkg wijken
```

## Data processing

DuckDB:

Install extensions:

```
INSTALL h3 FROM community;
install spatial;
```

Load extensions: 

```
load spatial;
load h3;
```

Create H3 indexes:

```
CREATE table atms AS FROM ST_Read('./data/atms.geojson');
ALTER TABLE atms ADD COLUMN h3_index BIGINT;
UPDATE atms SET h3_index =  h3_latlng_to_cell(st_y(geom), st_x(geom),6);
```

Create a view with hexagons and count atm's:

```
create view v_atms as
  SELECT 
    h3_index AS hexagoon_id,
    COUNT(*) AS amount,
     ST_GeomFromText(h3_cell_to_boundary_wkt(h3_index)) as geom
FROM (
    SELECT 
        h3_index
    FROM 
        atms
) AS subquery
GROUP BY 
    h3_index;
```

Create hexagon demo geojson:

```
COPY (
select geom as geometry, amount from v_atms) TO 'atms_hex.geojson' 
WITH (FORMAT GDAL, DRIVER 'GeoJSON', LAYER_NAME 'pin');
```

Load wijken

```
create table wijken as
SELECT *  
FROM ST_Read('./data/wijken_4326.gpkg', layer = 'wijken');
```

Analyze

```
create table wijken_without_pin as
SELECT w.*
FROM wijken w where
NOT EXISTS (
    SELECT 1
    FROM atms a
    WHERE ST_Intersects(w.geom, a.geom)
);
```

```
select wijkcode, wijknaam, gemeentenaam,water,aantal_inwoners 
from wijken_without_pin
order by aantal_inwoners
desc 
limit 5;
```

Export 

```
COPY (
select geom as geometry, wijkcode, wijknaam, gemeentenaam,water,aantal_inwoners from wijken_without_pin) TO './results/wijken_without_pin.geojson' 
WITH (FORMAT GDAL, DRIVER 'GeoJSON', LAYER_NAME 'pin');
```
