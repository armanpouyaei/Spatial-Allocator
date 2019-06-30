# Instruction to How to Run PostgreSQL/PostGIS SA Tool
# Created by UNC (11/1/2018)

Only support regular grid surrogates creation, not support Egrid or census track(polygon) surrogates

0. Prerequisites
   - Install the Postgres database server and the PostGIS extension. Make sure the Postgres server is running.
   - Create a new database for the surrogates work. This document uses a database named "surrogates" or "NEI2014" by default.
     > From the psql command line, run "CREATE DATABASE surrogates;"
     > Connect to the new database with the command "\c surrogates"
     > Add the PostGIS extension to the new database: "CREATE EXTENSION postgis;"

   - For the surrogates tool to run, you will need to have a database user with all privileges on the new database.
     :The following psql commands create a new user and assign the appropriate priveleges.
     > CREATE USER <username> WITH PASSWORD '<password>';
     > GRANT ALL PRIVILEGES ON DATABASE surrogates TO <username>;
     > GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO <username>;

   - Install the Java Runtime Environment (if needed).
   - Install the tcsh package (if needed).
   - Download the Spatial Allocator package.

1. Update "pg_setup.csh" based on your location in your server and then "source" it (create/start the PostgreSQL Database)
   > setenv SA_HOME    /opt/srgtool
   > setenv PGHOME     /usr/bin
   > setenv PGBIN      /usr/bin
   > setenv PGDATA     /opt/srgtool/data
   > setenv PG_USER zeadelma
   > setenv DBNAME  NEI2014

2. Download shapefiles or pg shapefile tables
   All shapefiles are dumped to directory pg_tables to facilitate tables restore. 
   - Location? 
   unc location: /proj/ie/proj/EMAQ/Platform/Surrogates/2014/Spatial-Allocator/pg_srgcreate/shapefiles/pg_tables

3. Load pg shapefile tables  
   - Go to ./util directory;
   - Modify pg_restore_tables.csh to restore postgres tables that you need.
    > Update the table_list_2014.txt file to only list the tables you want to restore
   
4. load projection and shapefiles 
   Instead of step 3, a user can choose to load all shapefiles and domain definitions by following the following steps using a different projection or domain. 
 a. Load the output modeling projection:
   - Go to ./util directory
   - Invoke the following command to insert the output projection into your DB. Note: User can use a different number than 900921 if this number is already taken in spatial_ref_sys table in your DB. 

     > "psql -h localhost -d surrogates -U $user -f create_900921.sql"

 b. Load modeling grid domain:
   - Update the following settings and variables to load the modeling domain in the PostgreSQL DB using the "generate_modeling_grid.sh" script
     : Update $dbname, $schemaname, $server, $user
     : Add your own modeling domain information (see the example)
   - Invoke "generate_modeling_grid.sh"

 c. Load data and multiple weight shape files first and then transform them to the output projection (ex: 900921)
   - Update the following settings and variables in the "load_shapefile_reproject_multi.csh" script
       : Add new weight shape files and/or remove already loaded shape files by commentting out.
       : Set "srid" to the output projection (e.g., 900921)
       : Update $dbname, $schemaname, $server, $user
   - Invoke "load_shapefile_reproject_multi.csh"
)

5. Create Surrogates
   - Update the following settings in the "control_variables_pg.csv" input file.
     > SRGCREATE EXECUTABLE
     > GRIDDESC  (Note: grid name can't be larger than 16 chars. ) 
     > OUTPUT_GRID_NAME
     > SURROGATE SPECIFICATION FILE
     > SHAPEFILE CATALOG
     > OUTPUT DIRECTORY
     > OUTPUT SRGDESC FILE
     > PGSCRIPT DIRECTORY
     > PG_SERVER
     > PG_USER
     > PGBIN
     > SRID_FINAL
     > DBNAME
     > COMPUTE SURROGATES,YES  (Note: set this to YES to create surrogate)
     > GAPFILL SURROGATES,NO   (Note: set this to NO since Gapfill will be done by another java tool, not by PostgreSQL/PostGIS (PG) SA tool) 

6.  Update the "surrogate_generation_pg.csv" input file to specify which surrogates to be created

7.  Update the "shapefile_catalog_pg.csv" to specify  schema name under "DIRECTORY" column if it is other than "public". The schema name is determined by data shapefile used in surrogate specification. 

8.  Update the "surrogate_specification_pg.csv" input file
    : Note that all identifiers (including column names) that are not double-quoted are folded to lower case in PostgreSQL. 
    : Column names that were created with double-quotes and thereby retained upper-case letters
      ex:  filter_function="moves2014=4", then moves2014 is case sensitive 
           weight_shape=Hpms2016, "Hpms2016" will be recognized as hpms2016

   - Update the following settings:
     > DATA SHAPEFILE ( same name as in shapefile_catalog_pg.csv )
     > DATA ATTRIBUTE ( better use lower case )
     > WEIGHT SHAPEFILE ( same name as in shapefile_catalog_pg.csv )
     > WEIGHT ATTRIBUTE ( better use lower case )
     > WEIGHT FUNCTION  ( better use lower case )
     > FILTER FUNCTION  ( better use lower case )

9.  Update the "run_pg_srgcreate.csh" script
     : Make sure to use the PG SA tool version of Java tool (SurrogateTools-2.1.jar) that came with this PG SA tool package

     > Invoke the "run_pg_srgcreate.csh" run script
     java -classpath ./SurrogateTools-2.1.jar gov.epa.surrogate.ppg.Main control_variables_pg.csv

     > Script "run_pg_srgcreate_batch.csh" can be used to submit jobs on cluster machine.  

10. Check out the PG SA templates for various types of surrogates (MultyPoly, Point, Line,,,)
    -  Go to the directory "pgscripts"
    -  These templates will be used to generate the internal PG SA run scripts to create surrogates, 4 or 5 steps are included in these templates: 1. weight shapefile cut by data(county) shapefile to combine weight fips with calculated weight attribute in output shapefile, this step can be skipped if weight shapefile already has fips as an attribute; 2. the output from first step or weight shape file cut by grid/domain shapefile to combine grid ids with calculated weight attribute; 3. calculate numerater table by summing weight attribute based on county; 4. calculate denominator by summing weight attribute based on county and grid cell; 5. calculate surrogate for based on county and grid cell.  

11. Gap filling and merge surrogate
   -  Once the surrogates are created by PG SA Tool, update the "control_variables_pg.csv" input file
      > COMPUTE SURROGATES,NO
      > MERGE SURROGATES,YES  #for 165 and 525
      > GAPFILL SURROGATES,YES
   -  Invoke the "run_srgtool_fill.csh" run script
   java -classpath ./SurrogateTools-2.1.jar gov.epa.surrogate.SurrogateTool control_variables_pg.csv

12. Create summary tables
   -  Invoke the "qa_srg.csh" run script
   java -classpath ./SurrogateTools-2.1.jar  gov.epa.surrogate.qa.Main ./outputs/srgdesc_us12k.txt tolerance

13. Normalization tool
   -  Invoke the "normalize_srg.csh" run script 
   java -classpath SurrogateTools-2.1.jar gov.epa.surrogate.normalize.Main SRGDESC_file exclude_list tolerance
   java -classpath SurrogateTools-2.1.jar gov.epa.surrogate.normalize.Main SRGDESC_file tolerance

14. Problems and notes
   - Deleted line segments or points exactly on border 
   - Run interactive job for testing
     srun -t 8:00:00 -p interact -N 1 -n 1 --x11=first --pty /bin/bash
   - surrogate 137 has very big values, but this surrogate seems not used in SMOKE gridding. 

15. Notes
geometry ST_CollectionExtract(geometry collection, integer type);
Type numbers are 1 == POINT, 2 == LINESTRING, 3 == POLYGON.
