Kiji Enron Emails Dataset
=========================

This repository was the source of the work done for this presentation: [Exploring the Enron Email Dataset with Kiji and Hive](http://www.slideshare.net/wibidata/exploring-the-enron-email-dataset-with-kiji-and-hive)

## Background Information:

The Enron email dataset is a collection of ~500k emails, from ~150 Enron employees(mostly senior management).
It is probably one of the largest publically available datasets of "real" emails, which makes it interesting.

* Data collected from: https://www.cs.cmu.edu/~enron/
We use the most recent(2009) version of this dataset.
* Schema derived from http://www.isi.edu/~adibi/Enron/Enron_Dataset_Report.pdf
* Sentiment words derived from AFINN: http://www2.imm.dtu.dk/pubdb/views/publication_details.php?id=6010

## Contents

This project contains:
* Kiji table schemas for email data.
* Command line importer for this data.  (This isn't a bulk importer since HDFS isn't great at lots of 
small files like in a Maildir format.
* Hive schemas for these tables.
* Basic sentiment producer based on the AFINN word list.

## Loading the dataset into Kiji:

### Compile everything
    mvn clean package

### Copy jar to lib dir
    mkdir hive/lib
    cp hive/target/*.jar hive/lib

### Setup the environment:

From the enron emails directory

    export ENRON_EMAIL_HOME=`pwd`
    export KIJI=kiji://.env/enron_email
    export LIBS_DIR=${ENRON_EMAIL_HOME}/mr/target
    export KIJI_CLASSPATH="${LIBS_DIR}/*"
    export EXPRESS_JOB_ROOT=${PWD}/express/target/express-enron-email-0.0.1-SNAPSHOT-release/express-enron-email-0.0.1-SNAPSHOT
  
### Creating the tables in Kiji:

    kiji install --kiji=${KIJI}
    kiji-schema-shell --kiji=${KIJI} --file=${ENRON_EMAIL_HOME}/ddl/email_schema.ddl

### Run the importer:

    kiji jar ./mr/target/mr-enron-email-1.0-SNAPSHOT.jar org.kiji.enronemail.bulkimport.EmailBulkImporter kiji://.env/enron_email/emails maildir/

### Copy the sentiment file to HDFS

    hadoop fs -copyFromLocal hive/AFINN-111.txt /tmp

### Run the sentiment producer:

    kiji produce --producer=org.kiji.enronemail.produce.SentimentProducer --input="format=kiji table=${KIJI}/emails" --output="format=kiji table=${KIJI}/emails nsplits=2" --lib=${LIBS_DIR}

### Running the completed Email Summary Express job:
    express job ${EXPRESS_JOB_ROOT}/lib/express-enron-email-0.0.1-SNAPSHOT.jar org.kiji.enronemail.job.EnronEmailSummaryCompleted -Dmapred.child.java.opts="-Xmx512m" --input ${KIJI}/emails --output . --hdfs --libjars ${EXPRESS_JOB_ROOT}/lib
    
### Running the TfIdf Express job:

    express job ${EXPRESS_JOB_ROOT}/lib/express-enron-email-0.0.1-SNAPSHOT.jar org.kiji.enronemail.job.EnronEmailTfIdfCompleted -Dmapred.child.java.opts="-Xmx512m" --input ${KIJI}/emails --output . --hdfs --libjars ${EXPRESS_JOB_ROOT}/lib

### Viewing the output of the Express jobs

    hadoop fs -cat top-senders-enron/part-00000 OR hadoop fs -getmerge top-senders-enron top-senders-enron.txt
    hadoop fs -cat top-correspondents-enron/part-00000 OR hadoop fs -getmerge top-correspondents-enron top-correspondents-enron.txt
    hadoop fs -cat tf-idf-matrix/part-00000 OR hadoop fs -getmerge tf-idf-matrix tf-idf-matrix.txt
