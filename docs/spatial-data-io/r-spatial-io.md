# Introduction
This section makes use of the following packages
```{r}
library(rpostgis)
library(RPostgreSQL)
```


# Connections

To set up a connection to the database you should use the commands dbDriver and dbConnect.

```{r}
drv <- dbDriver("PostgreSQL")
con <- dbConnect(drv, dbname="dbname",host="hostip",port=port,user="username",password="password" )
```

A handy function to close all connections, just in case you forgot to do in your code.
```{r}
kill_db_connections <- function (x) {

  all_cons <- dbListConnections(PostgreSQL())

  print(all_cons)

  for(con in all_cons)
  {
    dbDisconnect(con)
  }
  print(paste(length(all_cons), " connections killed."))

}
```

# Reading data

### Shapefiles

```{r}
# If the shapefile is simply in the same folder
sp_df <- readOGR(dsn = ".", layer = "shape_filename")

# If the data is stored somewhere else 
data_dir = "/data/shp/"

# Relative to the working directory
dsn_dir <- paste(getwd(),data_dir,sep="")

# Spatial dataframe
sp_df <- readOGR(dsn = dsn_dir, layer = "shape_filename")
```

### PostgresSQL / PostGIS
To list all the tables in a schema you can read from the information schema
```{r}
dbGetQuery(con,
           "SELECT table_name FROM information_schema.tables
           WHERE table_schema='foo'")
```

To read a table and drop the spatial information
```{r}
schema <- "schema"
table <- "table"
df <- dbReadTable(conn,c(schema,table))
```

To read a table and keep the spatial information ([rpostgis documentation](https://rdrr.io/cran/rpostgis/man/pgGetGeom.html){:target="_blank"})
```{r}
library(sf)
library(rpostgis)

schema <- "schema"
table <- "table"
sp_df <- pgGetGeom(conn,c(schema,table))
```



# Transforming data


Several operations on the column names
```{r}
# Lower the case in a spatial data frame, access the data with @data
colnames(sp_df@data) <- tolower(names(sp_df@data))

# Replace a string
colnames(df) <- sub("find","replace",names(df))

# Prepend / append a string to the column name with sep "_"
colnames(df) <- paste("prepend_string", colnames(df), sep = "_")
```



Transform 
```{r}
library(proj4)
library(rgdal)
library(rpostgis)

# Transform the CRS
sp_df_srid = spTransform(shape,CRS("+init=epsg:srid"))

# Summary of the file, to check the CRS
summary(sp_df_srid)
```

# Writing data
Writing a regular data from to Postgresql 
```{r}
output_schema <- "schema"
output_table <- "table"
# Write the table, overwrite the existing table
dbWriteTable(con,c(output_schema,output_table),df,row.names=FALSE,overwrite=TRUE)
```

Writing a spatial data frame to Postgresql 
```{r}
output_schema <- "schema"
output_table <- "table"
# Insert the records, overwrite the existing table, geometry column name is "geom"
pgInsert(conn,name=c(output_schema,output_table),sp_df,overwrite=TRUE,geom = "geom")
# Adding an index if necessary
dbIndex(conn, name=c(output_schema,output_table), "geom", method = c("gist"))
```