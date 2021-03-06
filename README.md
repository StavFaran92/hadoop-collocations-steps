# Hadoop MapReduce Collocations Extractor
Stav Faran - 308096270, Oshri Rozenberg - 204354344

This application extracts collocations from the Google 2-grams dataset using Amazon EMR.

Results are stored in the following bucket: https://s3.console.aws.amazon.com/s3/buckets/dsp-assignment2-oshri/result
(result for minNPMI = 0.7, relMinNPMI = 0.4)

Running instructions:
--------------------
1. Change the keyPair and bucketName fields in the Main.java file to be your key pair and bucket name for this run. In addition, change the log4jConfPath field to the appropriate path on your machine.
2. Store your AWS credentials in `~/.aws/credentials`
```
   [default]
   aws_access_key_id= ???
   aws_secret_access_key= ???
```
3. Place all the map-reduce jars (Stepi.jar for i=1,2,3,4,5) into your bucket
4. run using `java -jar ExtractCollocations.jar ExtractCollocations minNPMI relMinNPMI`

Steps summary:
-------------
1. Data validation & Count number of bigrams in total and by decade.
2. Count occurrences of the first word.
3. Count occurrences of the second word & Calculate NPMI.
4. Calculate Relative NPMI.
5. Extract Collocations.

Added a Combiner in steps 1 and 4 in order to optimize the runtime of this step.

Implemented a Comparator for step 5 in order to sort data by npmi (descending).

Implemented a Partitioner class that is responsible for distributing:
- a given pair of `[decade] [w1]` to the same reducer (In steps 2 and 3).
- a given `[decade]` to the same reducer (In steps 4 and 5).

First Step:
----------
1. Extracts the bigram from the corpus.
2. Data validation by filtering 1-grams and bigrams that contain stop words.
3. Counts the occurrences of a given bigram in the corpus by decade.
4. Count the total amount of bigrams.
    
Second Step:
-----------
Input - `[decade] [w1] [w2] [bgram count]`
1. First emit `[decade] [w1] [!]` as key for each given value in the mapper, with `[bgram count]` as value.
2. Afterwards, emit the value of the mapper (as key) with zero as value (default value).
The above actions are used as a technique to count the occurrences of the first word. This technique
takes advantage of the key sorting mechanism in hadoop.

3. For a given `[w1]` in a given `[decade]`:
    a. `[sum]` = the occurrences of `[w1]` as a first word in a bigram in `[decade]`.
    b. Emit every `[decade] [w1] [w2] [bgram count]` as key with `[sum]` as value.

Third Step:
----------
Input - `[decade] [w1] [w2] [bgram count] [w1 count]`
1. First emit `[decade] [w2] [!]` as key for each given value in the mapper, with `[bgram count]` as value.
2. Afterwards, emit the value of the mapper (as key) with zero as value (default value).
3. For a given `[w2]` in a given `[decade]`:
    a. `[sum]` = the occurrences of `[w2]` as a first word in a bigram in `[decade]`.
    b. Calculate `[npmi]` for `[w1] [w2]`.
    c. Emit every `[decade] [w1] [w2]` as key with `[npmi]` as value.
    
Fourth Step:
-----------
Input - `[decade] [w1] [w2] [npmi]`
1. First emit `[decade]` as key each given value in the mapper with `[npmi]` as value.
2. Afterwards, emit the value of the mapper (as key) with `[npmi]` as value.
3. For a given `[decade]`:
    a. `[sum]` = the npmis of bigrams in `[decade]`.
    b. Calculate `[relnpmi] = [npmi] / [sum]`.
    c. Emit every `[decade] [w1] [w2] [npmi]` as key with `[relnpmi]`.
    
Fifth Step:
----------
Input - `[decade] [w1] [w2] [npmi] [relnpmi]`
1. Check if a given bigram is a collocation.
2. If it is, emit `[decade] [w1] [w2] [npmi]`
