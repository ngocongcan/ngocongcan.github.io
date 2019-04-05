---
published: true
---


Working as backend developer, we have to deal with many headache Problems, one of them is Out of memory issue when working with large amount of data.


### Problem

In my Symfony project, I have to process RAW data from database then export to excel file, to do that, I use Doctrine with Query Builder to retrieve the data from database. With testing database, no issue happens until deploy the code to production server where the number of records is about 300k. I did some testing and the problem is the memory grows up quickly with large amount of data. Here are some experimental results:

| Records |  Memory usage |
|---------|---------------|
| 1k      |  30 Mb        |
| 20k     |  108 Mb       |
| 30k     |  143 Mb       |
| 50k     |  213 Mb       |
| 200k    |  600 Mb       |

Normally, we set memory_usage = 256 Mb/ 512 Mb in php.ini, that value is good enough for the server. So that, with 200k records we will get a trouble with out of memory issue. Luckily, With Mysql & Doctrine, we can retrieve that number of records without keeping all of them in the memory *Unbuffered queries*,  see [Buffered and Unbuffered queries](https://www.php.net/manual/de/mysqlinfo.concepts.buffering.php).

There is a good article [How to properly use Doctrine ORM to generate a large CSV download without consuming too much memory](https://medium.com/@mark.anthony.r.rosario/how-to-properly-use-doctrine-orm-to-generate-a-large-csv-download-without-consuming-too-much-memory-1edeeab10407) that Markanthonyrosario shows you how to resolve the issue with *Unbuffered queries*.

Unfortunately, the life is not always pink... Another issue happens

    General error: 2014 Cannot execute queries while other unbuffered queries are active.
    Consider using PDOStatement::fetchAll().
    Alternatively, if your code is only ever going to run against mysql, you may enable query buffering by setting the
    PDO::MYSQL_ATTR_USE_BUFFERED_QUERY attribute.

That means when the Unbuffered is executing, no more queries can be executed. So that, I have to rework with another approach.

Splitting the records, why not?

MYSQL also supports LIMIT and OFFSET in the Query, with LIMIT = MAX_ROW_PER_REQUEST (I set it = 30k) and the OFSSET changes, we can process large amount of data with the memory usage allocated to 30k rows.

    $START_TIME = time();
    $offset = 0;
    do {
        $LIMIT_QUERY = str_replace('___LIMIT___', self::MAX_ROW_PER_REQUEST, $RAW_QUERY);
        $FULL_RAW_QUERY = str_replace('___OFFSET___', 0, $LIMIT_QUERY);
        $statement = $this->em->getConnection()->prepare($FULL_RAW_QUERY);
        $statement->execute();
        $limitedRows = $statement->fetchAll();
        $offset += count($limitedRows);
        //______________CALCULATE_DATA_________________

        foreach ($limitedRows as $row) {
          // Processing data
        }
        $EXECUTION_TIME = time() - $START_TIME;
    } while (count($limitedRows) == self::MAX_ROW_PER_REQUEST);
    //  Stop when result count less than LIMIT

  Here is the experimental results:

  | Records |  Memory usage |  Execution time |
|---------|---------------|-----------------|
| 30k     |  142.64 Mb    |  50  seconds    |
| 60k     |  142.68 Mb    |  95  seconds    |
| 90k     |  142.71 Mb    |  140  seconds   |
| 120k    |  142.75 Mb    |  184  seconds   |
| 150k    |  142.79 Mb    |  228  seconds   |
| 180k    |  142.82 Mb    |  274  seconds   |
| 210k    |  142.86 Mb    |  315  seconds   |

So, we can adjust the MAX_ROW_PER_REQUEST to have the accepted memory_usage and EXECUTION_TIME. Both parameters are configured in the php.ini.
