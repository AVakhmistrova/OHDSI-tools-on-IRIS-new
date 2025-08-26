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
This section provides links for the __automated installation__ of the full OHDSI tool-chain (ATLAS, WebAPI, HADES, etc.) using the official [Broadsea repository](https://www.ohdsi.org/wp-content/uploads/2023/10/Londhe-Ajit_Broadsea-3.0-BROADening-the-ohdSEA_2023symposium-Ajit-Londhe.pdf).  
![OHDSI Broadsea](OHDSI_Broadsea.png)*Fig. 1. OHDSI Broadsea Deployment Architecture*<br>

The deployment is preconfigured to integrate with InterSystems IRIS for Health.
After completing the setup, you will have:
* A running __IRIS instance__ with the __Eunomia__ test OMOP CDM and vocabularies preloaded
* __Achilles__ analysis results on the Eunomia dataset, accessible directly in ATLAS
* A configured __RStudio/HADES__ environment ready for analytical workflows

Follow the platform-specific setup instructions:

* [macOS (Apple Silicon)](https://github.com/dwellbrock/ohdsi-iris-macos-env/)
* macOS (Intel) - _in develoment_
* Windows - _in development_
* Linux - _in development_
-----

## Loading Data into InterSystems IRIS and Running Analysis
This section describes practical options for loading the OMOP Common Data Model (CDM) into an InterSystems IRIS instance and performing Achilles analyses on it:
* [re-run analysis in the current result schema](#re-run-achilles-on-the-same-dataset-in-the-current-result-schema) - refresh analysis results without changing the schema
* [re-run analysis in a new result schema](#re-run-achilles-on-the-same-dataset-in-the-new-result-schema) - useful for comparing results with previous runs
* [load new data and run new analysis](#load-new-data-and-run-analyses) - import updated CDM data and vocabularies, then generate fresh Achilles results

### Re-run Achilles on the same dataset in the current result schema
Re-run Achilles on the existing dataset to __refresh the analysis results__ while keeping the same result schema unchanged.


1. Set up IRIS connection and initialize schemas: <br>
_(run the commands in RStudio)_
```R
# --- Connection ---
library(DatabaseConnector)
connectionDetails <- DatabaseConnector::createConnectionDetails(dbms = "iris", server = "host.docker.internal", user = "_SYSTEM", password = "_SYSTEM", pathToDriver = "/opt/hades/jdbc_drivers")
conn <- DatabaseConnector::connect(connectionDetails)

# --- Initializing the schemas ---
cdmSchema     <- "OMOPCDM53"           # schema with CDM data (e.g., preloaded Eunomia test dataset)
resultsSchema <- "OMOPCDM55_RESULTS"   # schema for Achilles results (e.g., Eunomia analysis results)
cdmVersion    <- "5.3"                 # CDM version (e.g., 5.3 for Eunomia)
```
   
2. Clean up the result schema in IRIS - remove all previously generated tables: <br>
_(run the commands in RStudio)_
```R
# --- Deleting all tables in schema ---
tables <- querySql(conn, sprintf("SELECT TABLE_NAME FROM INFORMATION_SCHEMA.TABLES WHERE TABLE_SCHEMA = '%s';", resultsSchema))

for (tableName in tables$TABLE_NAME) {
  sql <- sprintf('DROP TABLE "%s"."%s";', resultsSchema, tableName)
  message("Dropping table: ", tableName)
  executeSql(conn, sql)
}
```
3. Delete all logs and error reports in RStudio.

4. Delete Atlas сache and restart WebAPI: <br>
_(run the comands in Terminal)_

```sch
docker exec -it broadsea-atlasdb psql -U postgres -c "DELETE FROM webapi.achilles_cache WHERE source_id=2;"
docker exec -it broadsea-atlasdb psql -U postgres -c "DELETE FROM webapi.cdm_cache      WHERE source_id=2;"
docker restart ohdsi-webapi
```
_where:_ 
```R
# source_id = 2  -  ID of the preloaded Eunomia analysis results in Atlas
```

4. Run Achilles: <br>
_(run the commands in RStudio)_
```R
# --- Temporary step, should be done in the preconfigured container ---
remotes::install_github("OHDSI/SqlRender") 
packageVersion("SqlRender") # v1.19.3
          
# --- Ensure schema exists ---
executeSql(conn, sprintf("CREATE SCHEMA IF NOT EXISTS %s;", resultsSchema))

# --- Run Achilles ---
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

5. Create table _concept_hierarchy_ in Result schema if it wasn't created during Achilles run: <br>
_(run the commands in RStudio)_
```R
# --- Create table ---
createTableSql <- sprintf(
  '
  CREATE TABLE "%s"."concept_hierarchy" (concept_id INT, concept_name VARCHAR(255), treemap VARCHAR(255), concept_hierarchy_type VARCHAR(50), level1_concept_name VARCHAR(255), level2_concept_name VARCHAR(255), level3_concept_name VARCHAR(255), level4_concept_name VARCHAR(255));
  ', resultsSchema
)
executeSql(conn, createTableSql)

# --- Populate table ---
insertSql <- sprintf(
  '
  INSERT INTO "%s"."concept_hierarchy"
  SELECT concept_id, concept_name, domain_id AS treemap, \'DOMAIN_ONLY\' AS concept_hierarchy_type, domain_id AS level1_concept_name, NULL AS level2_concept_name, NULL AS level3_concept_name, NULL AS level4_concept_name FROM "%s"."concept";
  ', resultsSchema, cdmSchema
)
executeSql(conn, insertSql)
```

### Re-run Achilles on the same dataset in the new result schema
Run Achilles on the same dataset but write the results into a new result schema, making it possible __to compare outcomes__ with previous runs.
1. Register InterSystems IRIS as a new data‑source in Postgres (WebAPI): <br>
_(run the comands in Terminal)_ <br>
 ```sch
docker exec -it broadsea-atlasdb psql -U postgres -c "
INSERT INTO webapi.source(source_id, source_name, source_key, source_connection, source_dialect)
VALUES (3, 'my-iris-new', 'IRIS-new', 'jdbc:IRIS://host.docker.internal:1972/USER?user=_SYSTEM&password=_SYSTEM', 'iris');                        
INSERT INTO webapi.source_daimon( source_daimon_id, source_id, daimon_type, table_qualifier, priority) VALUES (7, 3, 0, 'OMOPCDM53', 0);          
INSERT INTO webapi.source_daimon( source_daimon_id, source_id, daimon_type, table_qualifier, priority) VALUES (8, 3, 1, 'OMOPCDM53', 10);         
INSERT INTO webapi.source_daimon( source_daimon_id, source_id, daimon_type, table_qualifier, priority) VALUES (9, 3, 2, 'OMOPCDM53_RESULTS', 0);   "
docker restart ohdsi-webapi
```
_where:_  
```R
# source_id = 3       - ID of the new analysis in Atlas
# 'my-iris-new'       - name of the new analysis in Atlas
# 'OMOPCDM53'         - schema with CDM data (e.g., preloaded Eunomia test dataset)
# 'OMOPCDM53_RESULTS' - new result schema
```
2. Delete all logs and error reports in RStudio
 
3. Set up IRIS connection and initialize schemas as described in [step 1](#re-run-achilles-on-the-same-dataset-in-the-current-result-schema), updating __resultSchema__ beforehand: <br>
```R
resultsSchema <- "OMOPCDM53_RESULTS"     # new result schema
```
4. Run Achilles and create table _concept_hierarchy_ as described in [steps 4-5](#re-run-achilles-on-the-same-dataset-in-the-current-result-schema)

### Load new data and run analyses
Import new CDM data and vocabularies into the database, then run Achilles to generate fresh analysis results.
1. Register InterSystems IRIS as a new data‑source in Postgres (WebAPI): <br>
_(run the comands in Terminal)_ <br>
 ```sch
docker exec -it broadsea-atlasdb psql -U postgres -c "
INSERT INTO webapi.source(source_id, source_name, source_key, source_connection, source_dialect)
VALUES (3, 'my-iris-new', 'IRIS-new', 'jdbc:IRIS://host.docker.internal:1972/USER?user=_SYSTEM&password=_SYSTEM', 'iris');                       
INSERT INTO webapi.source_daimon( source_daimon_id, source_id, daimon_type, table_qualifier, priority) VALUES (7, 3, 0, 'OMOPCDM54', 0);           
INSERT INTO webapi.source_daimon( source_daimon_id, source_id, daimon_type, table_qualifier, priority) VALUES (8, 3, 1, 'OMOPCDM54', 10);        
INSERT INTO webapi.source_daimon( source_daimon_id, source_id, daimon_type, table_qualifier, priority) VALUES (9, 3, 2, 'OMOPCDM54_RESULTS', 0);"
docker restart ohdsi-webapi
```
_where:_  
```R
# source_id = 3       - ID of the new analysis in Atlas
# 'my-iris-new'       - name of the new analysis in Atlas
# 'OMOPCDM54'         - new CDM schema
# 'OMOPCDM54_RESULTS' - new result schema
```

2. Set up IRIS connection and initialize schemas as described in [step 1](#re-run-achilles-on-the-same-dataset-in-the-current-result-schema), updating __cdm schema__ and  __result schema__ beforehand: <br>
```R
cdmSchema     <- "OMOPCDM54"           # name of a new schema  
resultsSchema <- "OMOPCDM54_RESULTS"   # name of a new resultSchema
```

3. Create CDM tables and metadata. <br>

Before data can be inserted, the target database must contain the full set of OMOP tables, constraints, and indexes:<br>
_(run the commands in RStudio)_
```R
library(CommonDataModel)

# --- Ensure schema exists ---
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

4. Download vocabularies __(optional)__. <br>

Depending on your dataset, you may need to download additional vocabularies from the official [OMOP repository Athena](https://athena.ohdsi.org). To do this:
  * Register or log in.
  * Go to the "Download" section (see Fig. 2).
  * Select the vocabularies relevant to your use case and generate a download package.

     ![Hades](athena_download.png)*Fig. 2. Download vocabularies from Athena*<br>

__Note__: The Community Edition cannot store the full Athena vocabulary set. In most cases the following are sufficient: Gender, Race, Ethnicity, SNOMED, LOINC, ICD10CM, ICD9CM, ICD9Proc, CPT4, HCPCS, NDC, RxNorm, RxNorm Extension.<br>
The required core vocabularies (Domain, Relationship, Vocabulary, Concept Class, CDM, Type Concept, UCUM, Visit Type, Drug Type, Procedure Type, Condition Type, etc.) are included by default in every package.<br>
Before loading into the database unzip the downloaded archive into a convenient directory inside the HADES container (this will be referred to as _vocabFileLoc_). <br>

```R
library(readr)
library(tools)
library(dplyr)
library(stringr)
vocabFileLoc <- "/home/rstudio/vocab"    # path to vocabulary CSV files inside the HADES container
vocabTables <- c(list.files("/home/rstudio/vocab")) 

for (table in vocabTables) {
    message("Table: ", table)
  filepath <- file.path(vocabFileLoc, table)
  df <- read_delim(filepath, delim = "\t", col_types = cols(.default = "c"))    # make sure to insert correct delimeter
  # Identify and convert date columns (e.g., valid_start_date, etc.)
  date_cols <- names(df)[str_detect(names(df), regex("date", ignore_case = TRUE))]
  for (col in date_cols) {
        df[[col]] <- as.Date(df[[col]], format = "%Y%m%d")
  }
  # Get table name without extension
  tableName <- file_path_sans_ext(table)
  DatabaseConnector::insertTable(
    connection = conn,
    tableName = paste0(cdmSchema, ".", tableName),
    data = df,
    dropTableIfExists = FALSE,
    createTable = FALSE,
    tempTable = FALSE
  )
}
```

5. Download your CDM data.
```
#code here
```

6. Run Achilles and create table _concept_hierarchy_ as described in [steps 4-5](#re-run-achilles-on-the-same-dataset-in-the-current-result-schema)
