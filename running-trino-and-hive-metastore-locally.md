# How to Run Trino and Hive Metastore Locally

While debugging an issue in a Trino-Iceberg connector I was working on, I decided it was finally time to set up Trino to run locally on my laptop. I've actually run it before using the [TpchQueryRunner](https://github.com/trinodb/trino/blob/master/testing/trino-tests/src/test/java/io/trino/tests/tpch/TpchQueryRunner.java) class, but this uses the TPCH connector, and I needed the Iceberg connector, which requires Hive to be running.

There are various instructions online detailing how to set up Trino to run locally (including the [README](https://github.com/trinodb/trino/blob/master/README.md) of the Trino project itself), and how to set up Hive locally, and even how to connect the two, but none quite served my purposes. For instance, I didn't actually need Hive itself running, just the metastore, which is a lot less work. Apache has some [docs](https://cwiki.apache.org/confluence/display/Hive/AdminManual+Metastore+3.0+Administration) for this, but again, it's just a little lacking for my usecase.

So I've decided to document what I ended up doing to run Trino and Hive metastore locally for Iceberg development. Most of these instructions will be specific to Mac, but they should apply pretty well to Linux systems.

## Prerequisites
You'll need Java 11 for Trino to run. The Hive version I used (3.1.2) recommends Java 8. You will also need Maven, both Trino and Hive are Maven projects.

## Steps
### Hadoop
Trino-Iceberg needs Hive and Hive needs Hadoop, so let's start with that. I installed [Hadoop](https://hadoop.apache.org/release/3.3.1.html) version 3.3.1. Download the tar and unpack it. Put it wherever, then make sure you set `HADOOP_HOME` to point to the directory it is installed in, e.g. I put `export HADOOP_HOME=/opt/hadoop-3.3.1` in my `.zshrc` file. In retrospect, it would have been cleaner to link `/opt/hadoop` to the current version I have installed, in case I ever wanted to upgrade. You also need to make sure this ends up on your `PATH`, so add `$HADOOP_HOME/sbin:$HADOOP_HOME/bin` to your `PATH` however you like.

### External RDBMS
Hive also needs some kind of RDBMS to store metastore objects. Technically, it ships with Apache Derby, but this isn't recommended for use beyond simple testing. You have a choice for what to use, the options being MS SQL, MySQL, MariaDB, Oracle, and Postgres. If you already have one of these just use it, see [here](https://cwiki.apache.org/confluence/display/Hive/AdminManual+Metastore+3.0+Administration) for information on how to set them up. I used Postgres and I'll detail how to set that up.

The easiest way to get Postgres on Mac is the [Postgres.app](https://postgresapp.com). Download this, run it, and click "Initialize" to start a new Postgres server. The default settings should be okay: host `localhost`, port 5432, connection URL `postgresql://localhost`. You can also install with homebrew or other methods, but this is the simplest.

### Hive metastore
Next we need to set up Hive, but really just the Hive metastore. Since I think version 3, Hive has included a standalone Hive metastore module because other tools, such as Trino, only make use of Hive's metastore anyway. This greatly simplifies this step. Instead of setting up a whole Hadoop cluster and a Hive cluster, we can just spawn a single process (and the RDBMS above).

Clone the [Hive](https://github.com/apache/hive) repo, I used version 3.1.2. Build it with Maven, run `mvn clean install -DskipTests`. This will create a directory inside the Hive repo `standalone-metastore/target/` with all the build artifacts for the standalone metastore. Specifically, we want the folder `standalone-metastore/target/apache-hive-metastore-3.1.2-bin/apache-hive-metastore-3.1.2-bin/`, which contains the useful subdirectories `bin/`, `conf/`, and `lib/`.

First, let's set up the `conf/` directory properly. There should be a file in there called `metastore-site.xml`, if not then make it. The contents should be the following (see my comments in caps):
```xml
<?xml version="1.0" encoding="UTF-8" standalone="no"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?><!--
   Licensed to the Apache Software Foundation (ASF) under one or more
   contributor license agreements.  See the NOTICE file distributed with
   this work for additional information regarding copyright ownership.
   The ASF licenses this file to You under the Apache License, Version 2.0
   (the "License"); you may not use this file except in compliance with
   the License.  You may obtain a copy of the License at
       http://www.apache.org/licenses/LICENSE-2.0
   Unless required by applicable law or agreed to in writing, software
   distributed under the License is distributed on an "AS IS" BASIS,
   WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
   See the License for the specific language governing permissions and
   limitations under the License.
-->
<!-- These are default values meant to allow easy smoke testing of the metastore.  You will
likely need to add a number of new values. -->
<configuration>
  <property>
    <name>metastore.thrift.uris</name>
    <value>thrift://localhost:9083</value>
    <description>Thrift URI for the remote metastore. Used by metastore client to connect to remote metastore.</description>
  </property>
  <property>
    <name>metastore.task.threads.always</name>
    <value>org.apache.hadoop.hive.metastore.events.EventCleanerTask</value>
    <!-- SOME TUTORIALS ADVISE ADDING MORE VALUES HERE BUT THIS WORKED FOR ME -->
  </property>
  <property>
    <name>metastore.expression.proxy</name>
    <value>org.apache.hadoop.hive.metastore.DefaultPartitionExpressionProxy</value>
  </property>
  <!-- THE ABOVE MAY BE ALREADY INCLUDED, THE BELOW YOU WILL HAVE TO ADD. THEY WILL BE DIFFERENT FOR DIFFERENT RDBMS'S -->
  <property>
    <name>javax.jdo.option.ConnectionURL</name>
    <value>jdbc:postgresql://localhost/postgres</value>
    <!-- YOUR POSTGRES SERVER STARTED ABOVE SHOULD HAVE A "postgres" SCHEMA ALREADY. PORT ISN'T NECESSARY IF IT IS THE DEFAULT -->
  </property>
  <property>
    <name>javax.jdo.option.ConnectionDriverName</name>
    <value>org.postgresql.Driver</value>
    <!-- SEE NOTES BELOW ON WHERE THIS COMES FROM -->
  </property>
  <property>
    <name>javax.jdo.option.ConnectionUserName</name>
    <value>postgres</value>
    <!-- DEFAULT POSTGRES USERNAME -->
  </property>
  <property>
    <name>javax.jdo.option.ConnectionPassword</name>
    <value>postgres_password</value>
    <!-- THIS SHOULDN'T ACTUALLY MATTER IF YOU DON'T SET UP A PASSWORD ON THE POSTGRES SERVER -->
  </property>
</configuration>
```

Notice the `org.postgresql.Driver` class, Hive will need this to interact with Postgres. Download the [Postgres JDBC driver](https://jdbc.postgresql.org/download.html) and make sure it is in your classpath. The easiest way to do this is to simply copy the jar directly into the `lib/` directory mentioned before.

Apache also recommends setting `metastore.try.direct.sql.ddl` to false when using Postgres, I didn't do this and didn't notice anything, but it's surely a good idea.

Anything in the `target/` directory is ephemeral to Maven. The next time you run `mvn clean`, the XML file and the JAR will be deleted. To make this more permanent, save these files elsewhere and copy them when needed. Even better, you can modify the file at `standalone-metastore/src/main/resources/metastore-site.xml`, this is what gets copied into the `target/` directory. You can also pass the JAR in the Hive commands below via the argument `--auxpath` which sets the environment variable `METASTORE_AUX_JARS_PATH`, allowing you to put the JAR wherever, but I have not tried this.

Before you can run the metastore, you need to do an initialization step. I'll confess I don't know what this does nor how often it is needed. My guess is you have to do it each time you use a new RDBMS, for example, if I wiped the Postgres server. I also guess that it's setting up tables that Hive needs for its own operations. Anyway, for this run `./bin/schematool -initSchema -dbType postgres`. Change the `dbType` if needed.

Finally, we can start the Hive metastore. Run `./bin/start-metastore`.

### Trino
This is like that [bit](https://youtu.be/AbSehcT19u0) in Malcolm in the Middle when Hal comes home and a lightbulb is out, so he goes to get a spare off the shelf but the shelf is broken, then to fix the shelf he grabs his screwdriver from the drawer and notices the rails are squeaky, so he picks up a can of WD-40 but it's empty. He's about to take off to the store to get a new can when his car won't start. Finally his wife comes into the garage asking, "Hal can you change the lightbulb?" and he slides out from under his car, covered in grease, "What does it look like I'm doing?!"

Hopefully this is better since I've laid out the steps in the reverse order I discovered them.

Clone the [Trino](https://github.com/trinodb/trino) repo. I was using Trino version 385. Follow the instructions in the README to run the trino-server-dev, this will expect a Hive process to be running on port 9083 which is what we have. You can connect to the server with the Trino CLI on port 8080. You can use the Hive or Iceberg catalogs like this. If you want to create a schema, use the syntax `create schema iceberg.schema_name with (location='file:////path/to/directory');`, the four slashes are required.
