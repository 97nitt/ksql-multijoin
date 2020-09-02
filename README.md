# Joining Multiple Streams with ksqlDB

**Question:**
How can I join a reference table with multiple streams into a new Kafka topic, where one of those streams is not required to produce a result?

**Example Use Case:**
Suppose you have an inventory of job postings and a stream of applications to those job postings. You also have some asynchronous process that attempts to score the fit of each application to the job posting.

In this tutorial, we will write a program that joins together the incoming applications with job posting data, and enriches the resulting data set with fit scores if they arrive within a certain time window.

## Try it

 1. Start Kafka infrastructure (requires [Docker Compose](https://docs.docker.com/compose/))
    ```
    $ docker-compose up -d
    ```
    This will start the following Docker containers:

    | Container     | Description                             |
    |---------------|-----------------------------------------|
    | zookeeper     | Required for Kafka cluster coordination |
    | broker        | Kafka broker                            |
    | ksqldb-server | ksqlDB server                           |
    | ksqldb-cli    | Container used to run the ksqlDB CLI    |

 2. Open the ksqlDB CLI
    ```
    $ docker exec -it ksqldb-cli ksql http://ksqldb-server:8088
    ```

 3. Create a Kafka topic and table for job posting data
    ```
    CREATE TABLE jobs (job_id INT PRIMARY KEY, title VARCHAR)
        WITH (
            KAFKA_TOPIC = 'jobs',
            VALUE_FORMAT = 'JSON',
            PARTITIONS = 3,
            REPLICAS = 1);
    ```

 4. Add some job postings to the topic/table
    ```
    INSERT INTO jobs (job_id, title) VALUES (1, 'Cashier');
    INSERT INTO jobs (job_id, title) VALUES (2, 'Cook');
    INSERT INTO jobs (job_id, title) VALUES (3, 'Driver');
    ```

 5. Create a Kafka topic and stream for job applications
    ```
    CREATE STREAM applications (application_id INT KEY, job_id INT, first_name VARCHAR, last_name VARCHAR)
        WITH (
            KAFKA_TOPIC = 'applications',
            VALUE_FORMAT = 'JSON',
            PARTITIONS = 3,
            REPLICAS = 1);
    ```

 6. Create a Kafka topic and stream for the job application scores
    ```
    CREATE STREAM application_scores (application_id INT KEY, score DOUBLE)
        WITH (
            KAFKA_TOPIC = 'application_scores',
            VALUE_FORMAT = 'JSON',
            PARTITIONS = 3,
            REPLICAS = 1);
    ```

 7. Create a Kafka topic and stream that joins the job application and posting data, as well as the asynchronously generated scores for each application
    ```
    CREATE STREAM scored_applications
        WITH (
            KAFKA_TOPIC = 'scored_applications',
            VALUE_FORMAT = 'JSON',
            PARTITIONS = 3,
            REPLICAS = 1)

        AS SELECT applications.application_id as application_id,
                  jobs.job_id as job_id,
                  jobs.title,
                  applications.first_name,
                  applications.last_name,
                  application_scores.score
        FROM applications
        INNER JOIN jobs on jobs.job_id = applications.job_id
        LEFT JOIN application_scores WITHIN 1 HOUR on application_scores.application_id = applications.application_id;
    ```

    We choose an `INNER JOIN` on the `jobs` table so we only produce a result when an application matches a job posting in the active inventory.

    We choose a `LEFT_JOIN` on the `scores` stream so we do not wait for a fit score to produce a result. The fit scoring may be an expensive operation that takes time, and we want to include the application in the result set even if the scoring fails for some reason.

 8. Start a "push" query of the scored application stream that will emit results as we send test data
    ```
    SELECT * FROM scored_applications EMIT CHANGES;
    ```

 9. In a second terminal window, open another ksqlDB CLI we will use to send test data
    ```
    $ docker exec -it ksqldb-cli ksql http://ksqldb-server:8088
    ```

10. Send some applications
    ```
    INSERT INTO applications (application_id, job_id, first_name, last_name) VALUES (1001, 1, 'Jane', 'Doe');
    INSERT INTO applications (application_id, job_id, first_name, last_name) VALUES (1002, 1, 'Joe', 'Smith');
    ```

    You should see the two unscored applications in the push query results:
    ```
    +----------------+--------+---------+------------+-----------+-------+
    |APPLICATION_ID  |JOB_ID  |TITLE    |FIRST_NAME  |LAST_NAME  |SCORE  |
    +----------------+--------+---------+------------+-----------+-------+
    |1001            |1       |Cashier  |Jane        |Doe        |null   |
    |1002            |1       |Cashier  |Joe         |Smith      |null   |
    ```

11. Send an application fit score
    ```
    INSERT INTO application_scores (application_id, score) VALUES (1001, 90.0);
    ```

    The push query should produce a new result with the scored application:
    ```
    +----------------+--------+---------+------------+-----------+-------+
    |APPLICATION_ID  |JOB_ID  |TITLE    |FIRST_NAME  |LAST_NAME  |SCORE  |
    +----------------+--------+---------+------------+-----------+-------+
    |1001            |1       |Cashier  |Jane        |Doe        |null   |
    |1002            |1       |Cashier  |Joe         |Smith      |null   |
    |1001            |1       |Cashier  |Jane        |Doe        |90.0   |
    ```

    We now have a Kafka topic that can be used to provide an employer with notifications of incoming job applications, and those applications will be annotated with a score, when available, so the employer can focus on the best fit applicants.

## Test it

 1. Put the statements creating the table and streams into a file ([src/statements.ksql](src/statements.ksql))

 2. Put test inputs into a file ([test/input.json](test/input.json))

 3. Put expected outputs into a file ([test/output.json](test/output.json))

 4. Run test using the [ksqlDB test runner](https://docs.ksqldb.io/en/latest/developer-guide/test-and-debug/ksqldb-testing-tool/)
    ```
    $ docker exec ksqldb-cli ksql-test-runner -i /opt/app/test/input.json -s /opt/app/src/statements.ksql -o /opt/app/test/output.json
            >>> Test passed!
    ```
