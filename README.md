# apache-x-table-docker-tutorial
apache-x-table-docker-tutorial
![Screenshot 2024-08-02 at 9 56 40 AM](https://github.com/user-attachments/assets/9cde76fb-9e5c-4cdc-bcd1-c23ebb1b304b)


### sample glue job
```
from pyspark.sql import SparkSession
from pyspark.sql.types import *

# Initialize Spark session
spark = SparkSession.builder \
    .appName("HudiExample") \
    .config("spark.serializer", "org.apache.spark.serializer.KryoSerializer") \
    .config("spark.sql.hive.convertMetastoreParquet", "false") \
    .getOrCreate()

# Initialize the bucket
table_name = "people"
base_path = "s3://XX/hudi-dataset"

# Define the records
records = [
   (1, 'John', 25, 'NYC', '2023-09-28 00:00:00'),
   (2, 'Emily', 30, 'SFO', '2023-09-28 00:00:00'),
   (3, 'Michael', 35, 'ORD', '2023-09-28 00:00:00'),
   (4, 'Andrew', 40, 'NYC', '2023-10-28 00:00:00'),
   (5, 'Bob', 28, 'SEA', '2023-09-23 00:00:00'),
   (6, 'Charlie', 31, 'DFW', '2023-08-29 00:00:00')
]

# Define the schema
schema = StructType([
   StructField("id", IntegerType(), False),
   StructField("name", StringType(), True),
   StructField("age", IntegerType(), True),
   StructField("city", StringType(), True),
   StructField("create_ts", StringType(), True)
])

# Create a DataFrame
df = spark.createDataFrame(records, schema)

# Define Hudi options
hudi_options = {
   'hoodie.table.name': table_name,
   'hoodie.datasource.write.recordkey.field': 'id',
   'hoodie.datasource.write.precombine.field': 'create_ts',
   'hoodie.datasource.write.partitionpath.field': 'city',
   'hoodie.datasource.write.hive_style_partitioning': 'true'
}

# Write the DataFrame to Hudi
(
   df.write
   .format("hudi")
   .options(**hudi_options)
   .mode("overwrite")  # Use overwrite mode if you want to replace the existing table
   .save(f"{base_path}/{table_name}")
)
```

my_config.yaml
```
sourceFormat: HUDI

targetFormats:
  - ICEBERG
  - DELTA
datasets:
  -
    tableBasePath: s3://<>/hudi-dataset/people/
    tableName: people
    partitionSpec: city:VALUE
    namespace: icebergdb
```



## DocckerFile
```
# Use the official Rocky Linux image as the base
FROM rockylinux:9

# Set the working directory
WORKDIR /home

# Install the required packages
RUN dnf install -y java-1.8.0-openjdk java-1.8.0-openjdk-devel maven git ncurses wget nano

# Set environment variables for Java
ENV JAVA_HOME=/usr/lib/jvm/java-1.8.0
ENV PATH=$JAVA_HOME/bin:$PATH

# Set environment variables from the .env file
ENV AWS_ACCESS_KEY_ID=XXXX
ENV AWS_SECRET_ACCESS_KEY=XXXX
ENV AWS_REGION=us-east-1

# Create a directory for xtable and switch to it
RUN mkdir xtable
WORKDIR /home/xtable

# Download required JAR files
RUN wget -O ./iceberg-spark-runtime-3.2_2.12-1.4.2.jar https://repo1.maven.org/maven2/org/apache/iceberg/iceberg-spark-runtime-3.2_2.12/1.4.2/iceberg-spark-runtime-3.2_2.12-1.4.2.jar \
    && wget -O ./iceberg-spark-runtime-3.3_2.12-1.4.2.jar https://repo1.maven.org/maven2/org/apache/iceberg/iceberg-spark-runtime-3.3_2.12/1.4.2/iceberg-spark-runtime-3.3_2.12-1.4.2.jar \
    && wget -O ./snowflake-jdbc-3.13.28.jar https://repo1.maven.org/maven2/net/snowflake/snowflake-jdbc/3.13.28/snowflake-jdbc-3.13.28.jar \
    && wget -O ./iceberg-aws-1.4.2.jar https://repo1.maven.org/maven2/org/apache/iceberg/iceberg-aws/1.4.2/iceberg-aws-1.4.2.jar \
    && wget -O ./bundle-2.23.9.jar https://repo1.maven.org/maven2/software/amazon/awssdk/bundle/2.23.9/bundle-2.23.9.jar

# Copy the configuration files into the container
COPY ./my_config.yaml /home/xtable/
COPY ./xtable-utilities-0.1.0-SNAPSHOT-bundled.jar /home/xtable/

# Run the Java application
CMD ["java", "-jar", "xtable-utilities-0.1.0-SNAPSHOT-bundled.jar", "--datasetConfig", "my_config.yaml"]


```


# Run it 
```
docker build -t my-xtable-app .
docker run -it my-xtable-app
```

![image](https://github.com/user-attachments/assets/7f503055-0e25-4f6d-9e0b-360abf72c1c1)
