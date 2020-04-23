# Solution to data pipeline test scenarios
 
 Note: This document was created in a limited span of time and can be further improved with formal test strategy document template. However, it does explain the test strategy in detail for the given scenarios.
 
 ## Test Strategy
For the given scenario, assuming that following are the main business goals for data cleaning:
* Filtering out records without specific attributes
* Converting the strings into their native types
* Removing redundant and bad data 

Data transfer speed and accuracy is vital, 
hence we can assume that following are the main business goals from accuracy and speed perspective
to ensure optimal, consistent runtimes for REA's ETL processes:

* COPY data from multiple, evenly sized files or different file sizes.
* May need workload management to improve ETL runtime
* Perform table maintenance regularly
* Perform multiple steps in a single transaction.
* Loading data in bulk.
* Extract large result sets.
* Monitor daily ETL health using diagnostic queries.

On a high level, the following testing strategy can be followed:

In any data pipeline, logs play a vital role in debugging and data analysis.

We need to make sure that we emit sufficient logs with timestamps and end user 
information (where applicable) to identify the root cause.

Example in this case we can emit logs — with record creation time, stage name, stage status, 
stage verbose logs and output template types for final transfer stage etc.

### Phase 1 ETL:

* Extract  - Make sure access to all data sources is sorted and we're able 
to get chunks of data from the primary source:
    * e.g. using curl -O 'https://data.rea.org/resource/gh4g-9sfh.json?$limit=50000' or some DB queries.
    * Here we need to ensure that extract stage works exactly as required.
    * Gauge the latency and see if that meets business standards, if not then discuss 
    to add more resources to reduce the latency.
    * Speed is important at each stage of the pipeline.
    
* Clean - Automated testing tools in conjunction with certain commandline tools 
can be used to test the Clean functions. For example: 
    * The raw data for listings will contain records that don't include certain attributes 
e.g. postcode or lets say address. 
    * These records are less useful because they fail to solve user's purpose, 
so it might make sense to remove them. 
    * Assuming we will have a set functions defined that takes in the record, 
and yields it only if it has certain attributes example in this case 
it should yield only if a record has both 
a postcode and a address defined:
```
def discard_incomplete(record):
    if 'postcode' in record and 'address' in record:
        yield record
```
We know exactly how each of the clean functions should work, so we can use python or other 
testing tools to write tests against each of them.

If they are flat files we can also use 
linux data-mining using various linux command line tools (awk, grep, sed, etc) 
to extract and manipulate data in text files.

Each stage can have its own isolated automated tests that validates only and only the functionality of that stage.

Along with that some really important integration tests are required to 
validate that data flows correctly from one stage to another.

* Transform - This is the stage where we need check the data type casting along with other data transformations.

Taking an example of datatype casting and assuming that we will have some functions defined that casts the relevant fields to the appropriate types:

Example:
```def convert_types(record):
    #Converts string values to their appropriate type
    # Only the year part of the datetime string is significant
    record['postcode'] = int(record['postcode'][:4])

    record['anotherattribute'] = int(record['anotherattribute']) if 'anotherattribute' in record else None

    address = record['address']
    address['streetname'] = string(address['streetname'])
    address['housenumber'] = int(address['housenumber'])
    address['housenumber'] = int(address['housenumber'])

    mapGeolocation = record['mapGeolocation']
    mapGeolocation['latitude'] = float(mapGeolocation['latitude'])
    mapGeolocation['longitude'] = float(mapGeolocation['longitude'])

    return record
```

For all the transformation functions we should have comprehensive automated tests 
that make sure the transform functions are doing their job.

* After the first transformation stage we need to perform primary source 
to intermediate database data testing

* This will involve 
   * constraint testing:
     * NOT NULL
     * UNIQUE
     * Primary Key
     * Foreign Key Check
     * NULL
     * Default 
   * duplicate Check Testing

### Phase 2 ETL:

* In phase 2 first we need to ensure that we're following presentation level best practices 
wherein we can use techniques such as templates to transform the data in required format.
* Hence template testing and schema validation is the key to make sure all users receive consistent data.
* Tests should be written for extract and transform stage similar to as mentioned in Phase 1 ETL.
* These tests will involve presentation level checks wherein file formats, data correctness, 
data schema should be taken care of.
* Create test users on runtime and replicate real-world integration and data flow scenarios 
that can enable some sort of network throttling and induce network failures.
* Fault Tolerance and Redundancy can also help in mitigating some failures in the pipeline
* A persistent cache, archive and backup can ensure we can rollback to certain data states.

### Testing Categories for overall Data Pipeline

* Metadata testing for
  * Data Type Checks
  * Data Length Checks
  * Same checks across multiple environments if applicable
  * Naming Standards Checks
  * Constraint Checks

* Data Completeness tests for
  * Record Count Validation
    * Source Query
    ``` 
         SELECT count(1) src_count FROM ListingSrc
    ```   
    * Target Query
    ```     
         SELECT count(1) tgt_count FROM ListingCleaned
    ```
  * Column Data Profile Validation or Aggregation Tests
    * Compare unique values in a column between the source and target.
    * Compare max, min, avg, max length, min length values for columns depending on the data type.
    * Compare null values in a column between the source and target.
    * For important columns, compare data distribution (frequency) in a column between the source and target
    * Example: Compare the number of listings by postcode between the source and target.
      * Source Query
         ```
        SELECT postcode, count(*) FROM listingSrc GROUP BY postcode
        ```
      * Target Query
       ```
       SELECT postcode, count(*) FROM listingCleaned GROUP BY postcode
      ```
    * Only if required by domain - then Compare Entire Source and Target Data
        * Use ETL testing tools to compare large amounts of data.
        * Example: Write a source query that matches the data in the target table after transformation.
          * Source Query
          ```
          SELECT record_id, postcode, house_num, street_num, ||’,’|| street_name, latitude, longitude FROM ListingSrc
          ```
          * Target Query
          ```
          SELECT integration_id, postcode, full_address, latitude, longitude FROM ListingCleaned
          ```
    * Automate Data Completeness Testing using some sort of ETL Validation tools to check:
        * Automate Data Profile Test Cases: 
           * that can automatically computes profile of the source and target query results – count, count distinct, nulls, avg, max, min, maxlength and minlength
        * Query Compare Test Cases: 
           * which can simplify the comparison of results from source and target queries.

* Data Quality tests
    * Duplicate Data Checks
        * Example: Business requirement may say that a combination of postcode, house_number and street_name should be unique.
          * Sample query to identify duplicates:
          ```
          SELECT postcode, house_number, street_name, count(1) FROM Listings 
          GROUP BY postcode, house_number, street_name HAVING count(1)>1
          ```
  
 ## Analysis and Problem Solving
 
 #### SITUATION 1: Our users have received the files for today but it only had data from 2 days ago. 
Steps to perform to reach the root cause:
* First check that recent data is available in the intermediate database.
* For each data set sent to the user each day, a record should be maintained as 
metadata with timestamps and other details.
* If the data is present in the intermediate database then we know the issue lies in the second ETL
 process
* There after we should check the logs at each ETL stage to figure out the issue
* If recent data is not present in intermediate database than failure is in Phase 1 ETL, 
we should first check in the primary data source before going through the logs of Phase 1 ETL.
That way we can quickly see if the data was present at all before checking the data loss through logs.

* Solution:
One solution to this is check to see if we have we made date checks in the code to 
 ensure we only send data for certain date thresh-holds.
 Sudo Query:
```
 select count(1) cnt from T1 where recordId = {recordId} 
  and CreatedDate > dateadd(d, -1, GETDATE()) and CreatedDate < GETDATE() 
```

#### SITUATION 2: One of our end users didn’t receive the file. 
* We should first check in Intermediate DB
* If present then check logs for second ETL process
* If not than go to primary source
* Check Logs for stages in descending order
* Personally I always prefer to check DB and query data sets first before moving on to logs.

* Solution:
   * Fault Tolerance, retries and redundancy should be built in the pipeline
   * We can create a functionality that ensures we receive some kind of acknowledgement from the end user.
   * If acknowledgement signal is not received pipeline can retry the data set a number of times within a defined thresh-hold.
   
#### SITUATION 3: User 1 complains that the files they received didn’t have consistent data. 
 * We should first check in Intermediate DB to see if first transformation was successful.
 * Transformation jobs can fail hence we need to check specific stage logs to drill down to the issue.
 * Some poor data leaks from Stage 1 ETL can turn into inconsistent data transformation in Stage 2.
 
 * Solution:
   * We must use templates to transform data such that data is always consistent
   * We need to improve on schema validations which can enforce consistency of data 
   
#### SITUATION 4: Users have raised an issue that file delivery time has been inconsistent, with the following delivery times observed.
     
 * -	01/04/2020 5:45pm
 * -	02/04/2020 6:00pm
 * -	03/04/2020 5:50pm
 * -	04/04/2020 6:30pm
 
 
* Here, the problem can be found in the following areas:
  * Check if data to the users is being sent sequentially.
  * Is the content delivery service guaranteeing the time thresh-hold ?
  * Isolate and analyse content delivery process

* Solution: We need to ensure that we use data delivery techniques that is multi-threaded 
and fault tolerant with very low latency.