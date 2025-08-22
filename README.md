# Instructions for Integrating OHDSI/Broadsea with the InterSystems IRIS Data Platform

## Overview
This guide provides step-by-step instructions for installing __InterSystems IRIS for Health__, deploying the __OHDSI Broadsea__ tool-stack, and connecting the two platforms.
It is intended for engineers and data scientists who need a reproducible setup on __Linux__, __macOS__, or __Windows__ (via Docker).

### Prerequisites
- __Hardware__ — 4 CPU, 8 GB RAM minimum (16 GB recommended)
- __Operating System__ — Linux, macOS, Windows
- __Docker Desktop__ — required for container‑based deployment
- __Git__ — for cloning repositories


## Running InterSystems IRIS for Health, deploying OHDSI Broadsea and connecting to InterSystems IRIS
![OHDSI Broadsea](OHDSI_Broadsea.png)
*Fig. 1. OHDSI Broadsea Deployment Architecture*

This section provides links for the __automated installation__ of the full OHDSI tool-chain (ATLAS, WebAPI, HADES, etc.) using the official [Broadsea repository](https://www.ohdsi.org/wp-content/uploads/2023/10/Londhe-Ajit_Broadsea-3.0-BROADening-the-ohdSEA_2023symposium-Ajit-Londhe.pdf).  
The deployment is preconfigured to integrate with InterSystems IRIS for Health.
After completing the setup, you will have:
* A running __IRIS instance__ with the __Eunomia__ test OMOP CDM and vocabularies preloaded
* __Achilles__ analysis results on the Eunomia dataset, accessible directly in ATLAS
* A configured __RStudio/HADES__ environment ready for analytical workflows

Follow the platform-specific setup instructions:

* [macOS (Apple Silicon)](https://github.com/dwellbrock/ohdsi-iris-macos-env/)
* macOS (Intel) - _in develoment_
* Windows - _in development_

## Loading Data into InterSystems IRIS and Running Analysis

This section describes practical options for loading the OMOP Common Data Model (CDM) into an InterSystems IRIS instance and performing Achilles analyses on it.

### Re-run Achilles on the same dataset in the current result schema
Refreshes the analysis results without changing the schema.

1. Clean up the results schema in IRIS - remove all previously generated tables:
```
# --- Initializing the schema ---
resultsSchema <- "OMOPCDM55_RESULTS"              # analysis results on the preloaded Eunomia test dataset

# --- Connection ---
library(DatabaseConnector)
connectionDetails <- DatabaseConnector::createConnectionDetails(dbms = "iris", server = "host.docker.internal", user = "_SYSTEM", password = "_SYSTEM", pathToDriver = "/opt/hades/jdbc_drivers")
conn <- DatabaseConnector::connect(connectionDetails)

# --- Deleting all tables in schema ---
tables <- querySql(conn, sprintf("SELECT TABLE_NAME FROM INFORMATION_SCHEMA.TABLES WHERE TABLE_SCHEMA = '%s';", resultsSchema))

for (tableName in tables$TABLE_NAME) {
  sql <- sprintf('DROP TABLE "%s"."%s";', resultsSchema, tableName)
  message("Dropping table: ", tableName)
  executeSql(conn, sql)
}
```
2. Delete all logs and error reports in RStudio

3. Delete Atlas сache and restart WebAPI:

```
docker exec -it broadsea-atlasdb psql -U postgres -c "DELETE FROM webapi.achilles_cache WHERE source_id=2;"
docker exec -it broadsea-atlasdb psql -U postgres -c "DELETE FROM webapi.cdm_cache      WHERE source_id=2;"
docker restart ohdsi-webapi
```

4. Run the commands in RStudio:

```
# --- Temporary step, should be done in the preconfigured container
remotes::install_github("OHDSI/SqlRender") 
packageVersion("SqlRender") # v1.19.3

# --- Initializing the schemas
cdmSchema     <- "OMOPCDM53"           # preloaded Eunomia test dataset  
resultsSchema <- "OMOPCDM55_RESULTS"   # analysis results on the preloaded Eunomia test dataset 
cdmVersion    <- "5.3"    

# --- Connection ---
library(DatabaseConnector)
connectionDetails <- DatabaseConnector::createConnectionDetails(dbms = "iris", server = "host.docker.internal", user = "_SYSTEM", password = "_SYSTEM", pathToDriver = "/opt/hades/jdbc_drivers")
conn <- DatabaseConnector::connect(connectionDetails)
          
# --- Ensure schema exists 
executeSql(conn, sprintf("CREATE SCHEMA IF NOT EXISTS %s;", resultsSchema))

# --- Run Achilles
library(Achilles)
achilles(
   connectionDetails       = connectionDetails,
   cdmDatabaseSchema       = cdmSchema,
   resultsDatabaseSchema   = resultsSchema,
   cdmVersion              = cdmVersion,
   createTable             = TRUE,
   numThreads              = 1,
   smallCellCount          = 5,
   scratchDatabaseSchema   = resultsSchema,      
   tempEmulationSchema     = resultsSchema,       
   optimizeAtlasCache      = TRUE
  )
```

5. Create table __concept_hierarchy__ in Result schema if it wasn't created during Achilles run:

```
# --- Create table
createTableSql <- sprintf(
  '
  CREATE TABLE "%s"."concept_hierarchy" (
      concept_id INT,
      concept_name VARCHAR(255),
      treemap VARCHAR(255),
      concept_hierarchy_type VARCHAR(50),
      level1_concept_name VARCHAR(255),
      level2_concept_name VARCHAR(255),
      level3_concept_name VARCHAR(255),
      level4_concept_name VARCHAR(255)
  );
  ',
  resultsSchema
)
executeSql(conn, createTableSql)

# --- Populate table
insertSql <- sprintf(
  '
  INSERT INTO "%s"."concept_hierarchy"
  SELECT
      concept_id,
      concept_name,
      domain_id AS treemap,
      ''DOMAIN_ONLY'' AS concept_hierarchy_type,
      domain_id AS level1_concept_name,
      NULL AS level2_concept_name,
      NULL AS level3_concept_name,
      NULL AS level4_concept_name
  FROM "%s"."concept";
  ',
  resultsSchema,
  cdmSchema
)
executeSql(conn, insertSql)
```

### Re-run Achilles on the same dataset in the new result schema
Useful for comparing results with previous runs.
1. Register InterSystems IRIS as a new data‑source in Postgres (WebAPI)
   
 ```
docker exec -it broadsea-atlasdb psql -U postgres -c "
DELETE FROM webapi.source_daimon WHERE source_daimon_id NOT IN (1, 2, 3, 4, 5, 6);
DELETE FROM webapi.source WHERE source_id NOT IN (1, 2);
INSERT INTO webapi.source(source_id, source_name, source_key, source_connection, source_dialect)
VALUES (3, 'my-iris-new', 'IRIS-new', 'jdbc:IRIS://host.docker.internal:1972/USER?user=_SYSTEM&password=_SYSTEM', 'iris');                        # 'my-iris-new' - name of the new analysis in Atlas
INSERT INTO webapi.source_daimon( source_daimon_id, source_id, daimon_type, table_qualifier, priority) VALUES (7, 3, 0, 'OMOPCDM53', 0);          # preloaded Eunomia test dataset 
INSERT INTO webapi.source_daimon( source_daimon_id, source_id, daimon_type, table_qualifier, priority) VALUES (8, 3, 1, 'OMOPCDM53', 10);         # preloaded Eunomia test dataset
INSERT INTO webapi.source_daimon( source_daimon_id, source_id, daimon_type, table_qualifier, priority) VALUES (9, 3, 2, 'OMOPCDM53_RESULTS', 0);  # new resultSchema "
docker restart ohdsi-webapi
```

2. Delete all logs and error reports in RStudio
 
3. Run the same [steps 4 and 5](#re-run-achilles-on-the-same-dataset-in-the-current-result-schema) described previously, updating __resultSchema__ beforehand:
```
resultsSchema <- "OMOPCDM53_RESULTS"  # new results schema
```

### Load new data and run analyses
Import new CDM data and updated vocabularies, then run Achilles to generate fresh results.
1. Register InterSystems IRIS as a new data‑source in Postgres (WebAPI)
   
 ```
docker exec -it broadsea-atlasdb psql -U postgres -c "
DELETE FROM webapi.source_daimon WHERE source_daimon_id NOT IN (1, 2, 3, 4, 5, 6);
DELETE FROM webapi.source WHERE source_id NOT IN (1, 2);
INSERT INTO webapi.source(source_id, source_name, source_key, source_connection, source_dialect)
VALUES (3, 'my-iris-new', 'IRIS-new', 'jdbc:IRIS://host.docker.internal:1972/USER?user=_SYSTEM&password=_SYSTEM', 'iris');                        # 'my-iris-new' - name of the new analysis in Atlas
INSERT INTO webapi.source_daimon( source_daimon_id, source_id, daimon_type, table_qualifier, priority) VALUES (7, 3, 0, 'OMOPCDM54', 0);          # name of a new schema 
INSERT INTO webapi.source_daimon( source_daimon_id, source_id, daimon_type, table_qualifier, priority) VALUES (8, 3, 1, 'OMOPCDM54', 10);         # name of a new schema
INSERT INTO webapi.source_daimon( source_daimon_id, source_id, daimon_type, table_qualifier, priority) VALUES (9, 3, 2, 'OMOPCDM54_RESULTS', 0);  # new resultSchema "
docker restart ohdsi-webapi
```

2. Initializing the CDM Schema <br>

   Before any rows can be inserted, the target database must expose the full set of OMOP tables, constraints, and indexes:

```
# --- Temporary step, should be done in the preconfigured container
remotes::install_github("OHDSI/SqlRender") 
packageVersion("SqlRender") # v1.19.3

# --- Initializing the schemas
cdmSchema     <- "OMOPCDM54"           # name of a new schema  
resultsSchema <- "OMOPCDM54_RESULTS"   # name of a new resultSchema
cdmVersion    <- "5.4"                 # version of your data

library(DatabaseConnector)
library(CommonDataModel)

# --- Connection --- 
connectionDetails <- DatabaseConnector::createConnectionDetails(dbms = "iris", server = "host.docker.internal", user = "_SYSTEM", password = "_SYSTEM", pathToDriver = "/opt/hades/jdbc_drivers")
conn <- DatabaseConnector::connect(connectionDetails)

# --- Ensure schema exists 
executeSql(conn, sprintf("CREATE SCHEMA IF NOT EXISTS %s;", cdmSchema))

# --- Adaptive call to executeDdl() across package versions ---
fargs <- names(formals(CommonDataModel::executeDdl))
args  <- list(
  connectionDetails   = connectionDetails,
  cdmDatabaseSchema   = cdmSchema,
  cdmVersion          = cdmVersion
)
if ("createIndices" %in% fargs)  args$createIndices  <- TRUE
if ("createIndexes" %in% fargs)  args$createIndexes  <- TRUE
# Some package versions also accept these; harmless if ignored:
if ("createConstraints" %in% fargs) args$createConstraints <- TRUE
if ("createPrimaryKeys" %in% fargs) args$createPrimaryKeys <- TRUE
do.call(CommonDataModel::executeDdl, args)
```

3. Download vocabularies<br>

Depending on your dataset, you may need to download additional vocabularies separately. These can be obtained from the official OMOP vocabulary repository at [https://athena.ohdsi.org](https://athena.ohdsi.org). To do this, register or log in, go to the *“Download”* section (Fig. 4), select the vocabularies relevant to your use case, and generate a download package.
![Hades](athena_download.png)
The Community Edition cannot store the full Athena vocabulary set. The following vocabularies are typically sufficient: ICD10CM, ICD9CM, ICD9Proc, CPT4, HCPCS, NDC, RxNorm, RxNorm Extension, SNOMED, LOINC, Visit Type, Drug Type, Procedure Type, Condition Type, Observation Type, Death Type, Note Type, Measurement Type, Device Type, Cohort Type, Gender, Race, Ethnicity, Domain, Relationship, Vocabulary, Concept Class, CDM, Type Concept, UCUM.
Before loading the vocabularies into the database, you need to unzip the vocabulary archive you received from Athena into a convenient directory on your local machine. This directory will be referred to as vocabPath in the code examples below. It should contain CSV files such as CONCEPT.csv, VOCABULARY.csv, RELATIONSHIP.csv, and others. Make sure the path you provide in the code matches the location of the unzipped files.
```
#code here
```

5. Download your data
```
#code here
```

6. Run Achilles:
   
```
# --- Ensure schema exists 
executeSql(conn, sprintf("CREATE SCHEMA IF NOT EXISTS %s;", resultsSchema))

# --- Run Achilles
library(Achilles)
achilles(
   connectionDetails       = connectionDetails,
   cdmDatabaseSchema       = cdmSchema,
   resultsDatabaseSchema   = resultsSchema,
   cdmVersion              = cdmVersion,
   createTable             = TRUE,
   numThreads              = 1,
   smallCellCount          = 5,
   scratchDatabaseSchema   = resultsSchema,      
   tempEmulationSchema     = resultsSchema,       
   optimizeAtlasCache      = TRUE, # create achilles_result_concept_count
  )
```
