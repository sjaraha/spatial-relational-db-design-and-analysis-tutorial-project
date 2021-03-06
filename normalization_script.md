
### Normalization Script

```sql
--create function isnumeric()

CREATE OR REPLACE FUNCTION isnumeric(text) RETURNS BOOLEAN AS $$
DECLARE x NUMERIC;
BEGIN
    x = $1::NUMERIC;
    RETURN TRUE;
EXCEPTION WHEN others THEN
    RETURN FALSE;
END;
$$
STRICT
LANGUAGE plpgsql IMMUTABLE;
-----------------------------------------------------
--CREATE TABLE (FRS_FACILITY_ISNUMERIC)
--FROM FRS_FACILITY
-----------------------------------------------------

--create table: frs_facility_isnumeric
CREATE TABLE frs_facility_isnumeric AS
(SELECT * FROM frs_facility) with no data ;

--insert values: frs_facility_isnumeric
INSERT INTO frs_facility_isnumeric
(SELECT * FROM frs_facility
WHERE isnumeric(longitude83)
AND isnumeric(latitude83));

-----------------------------------------------------
--CREATE TABLE (FACILITY)
--FROM (FRS_FACILITY_ISNUMERIC)
-----------------------------------------------------

--create table: facility
CREATE TABLE facility 
(registry_id character varying PRIMARY KEY,
primary_name character varying,
state character varying,
city character varying,
geom geometry(point,4269));

--insert values: facility
INSERT INTO facility
(SELECT registry_id, primary_name, state_code, city_name, ST_SetSRID(ST_MakePoint(longitude83::double precision, latitude83::double precision),4269)
FROM frs_facility_isnumeric
ORDER BY registry_id);

-----------------------------------------------------
--CREATE TABLE (NAICS)
--FROM (FRS_NAICS) + (NAICS_DESCRIPTION_2012)
-----------------------------------------------------

--create table: naics
CREATE TABLE naics AS
(SELECT DISTINCT registry_id, naics_code, code_description
FROM frs_naics
WHERE naics_code IS NOT NULL);

--add primary key
ALTER TABLE naics
ADD PRIMARY KEY(registry_id, naics_code);

--add column: economic_sector
ALTER TABLE naics
ADD COLUMN economic_sector varchar;

--prepare to update column: economic_sector
--account for range entries in naics_description_2012 (31-33, 44-45, 48-49)
INSERT INTO naics_description_2012 (naics_title_2012, naics_code_2012)
VALUES
('Manufacturing', '31'),
('Manufacturing', '32'),
('Manufacturing', '33'),
('Retail Trade', '44'),
('Retail Trade', '45'),
('Transportation and Warehousing', '48'),
('Transportation and Warehousing', '49');

--update economic_sector
UPDATE naics
SET economic_sector = naics_title_2012
FROM
	(SELECT naics_title_2012, naics_code_2012
	FROM naics_description_2012
	WHERE length(naics_code_2012) = 2) as sub
WHERE naics.naics_code ilike sub.naics_code_2012||'%';

-----------------------------------------------------
--CREATE TABLE (ENVIRONMENTAL_INTEREST)
--FROM (FRS_NAICS) 
-----------------------------------------------------

--create table: environmental_interest
CREATE TABLE environmental_interest
(registry_id varchar,
interest_type varchar,
PRIMARY KEY(registry_id, interest_type));

--insert values: environmental_interest
INSERT INTO environmental_interest
SELECT DISTINCT registry_id, interest_type
FROM frs_naics
ORDER BY registry_id;

-----------------------------------------------------
--CREATE TABLE (NPL_FINAL_SITE)
--FROM (NPL_FINAL) 
-----------------------------------------------------

--create table: npl_final_site
CREATE TABLE npl_final_site
(site_id varchar PRIMARY KEY,
state varchar,
city varchar,
geom geometry(point,4269));

--insert values: npl_final_site
INSERT INTO npl_final_site(site_id, state, city)
(SELECT DISTINCT  (TRIM(both chr(8236) from (TRIM(both chr(32) from TRIM(both chr(8237) from site_id))))), st, city
FROM npl_final
WHERE site_id != '');

UPDATE npl_final_site
SET geom = point
FROM 
	(SELECT 
		 (TRIM(both chr(8236) from (TRIM(both chr(32) from TRIM(both chr(8237) from site_id))))) as sid, 
		ST_SetSRID(ST_MakePoint(
		(TRIM(both chr(8236) from (TRIM(both chr(32) from TRIM(both chr(8237) from longitude)))))::double precision, 
		(TRIM(both chr(8236) from (TRIM(both chr(32) from TRIM(both chr(8237) from latitude)))))::double precision),
		4269) as point
	FROM npl_final
	WHERE longitude != '') as sub
WHERE npl_final_site.site_id=sub.sid;

```

