---
layout: post
title: Efficient file input, output and storage in R
---

Whether used in academia, industry or journalism, working with R involves importing and exporting a lot of data. While the basic functions to read and write files are known to all users, different methods have been developed over the years to optimise this process.

In this article, we'll have a look at the most efficient ways to read and write permanent files (i.e. in plain-text formats such as CSV), and to save and load binary files, a solution often overlooked by R users but much better suited to regular analysis of a given dataset.


## Setting up our benchmark

We'll be using functions from four different packages (readr, data.table, feather, and fst), and comparing them using the microbenchmark package.

    ```r
    install.packages(c("microbenchmark", "readr", "data.table", "feather", "fst"))

    library(microbenchmark)
    library(readr)
    library(data.table)
    library(feather)
    library(fst)
    ```

The dataset we'll be using as an example contains random data over 20 columns and 500,000 rows, for a reasonable size of 115 MB in CSV format. The 20 variables are a mix of integers, real numbers, dates and strings.

    ```r
    filename <- "dataset.csv"
    ```


## Permanent input/output

For the first part of this analysis, we'll look at permanent input/output, i.e. reading and writing files in common formats in data science, especially when files are shared between people. In other words, what's the most efficient way to open a CSV file you received or downloaded; and what's the most efficient way of outputting your own file to share with somebody else?


### Reading a plain-text file

3 functions are available to us:

* `read.csv` from the `read.*` series of functions in base R;
* `read_csv` from the `read_*` series of functions in the readr package, which is part of the tidyverse;
* `fread` from the data.table package.

Let's use `microbenchmark` to import our file using those three functions (the `microbenchmark()` functions executes each expression 10 times and averages the elapsed time).

    ```r
    microbenchmark(data <- read.csv(filename),
                   data <- read_csv(filename),
                   data <- fread(filename),
                   times = 10, unit = "s")

    ## Unit: seconds
    ##                        expr       min        lq      mean    median
    ##  data <- read.csv(filename) 25.662873 26.829712 27.915110 27.344555
    ##  data <- read_csv(filename)  2.303900  2.502276  2.971335  3.020295
    ##     data <- fread(filename)  3.273398  3.467531  3.707234  3.777934
    ##         uq       max neval
    ##  27.681976 35.389826    10
    ##   3.329171  3.618546    10
    ##   3.938012  4.146957    10
    ```

The `read_csv` and `fread` functions imported our file in 3 seconds, against 28 seconds for the `read.csv` function. This improvement is mostly due to the way those two functions identify the type of each column - by guessing the type based on a sample of values.

More generally, `read_csv` and `fread` tend to assume that the file is quite "clean", a lot more than `read.csv` does: import functions in base R offer a lot of optional arguments to deal with comments, missing values, unnecessary spaces in strings, etc.

But if the file we're working with has been generated in a clean way, `read_csv` and `fread` should deal with it without any error and guess the correct data types, much faster than `read.csv`.

### Writing a plain-text file

We'll now test the three equivalent functions to *write* the same file instead of reading it:

* `write.csv` from the `write.*` series of functions in base R;
* `write_csv` from the `write_*` series of functions in the readr package;
* `fwrite` from the data.table package (introduced in 2016).

    ```r
    microbenchmark(write.csv(data, "baseR_file.csv", row.names = F),
                   write_csv(data, "readr_file.csv"),
                   fwrite(data, "datatable_file.csv"),
                   times = 10, unit = "s")

    ## Unit: seconds
    ##                                              expr        min         lq
    ##  write.csv(data, "baseR_file.csv", row.names = F) 13.8066424 13.8248250
    ##                 write_csv(data, "readr_file.csv")  3.6742610  3.7999409
    ##                fwrite(data, "datatable_file.csv")  0.3976728  0.4014872
    ##        mean     median         uq        max neval
    ##  13.9118324 13.8776993 13.9269675 14.3241311    10
    ##   3.8572456  3.8690681  3.8991995  4.0637453    10
    ##   0.4097876  0.4061506  0.4159007  0.4355469    10
    ```

The results are impressive: readr improved our writing time from 14 seconds in base R to 4 seconds with `write_csv` - but `fwrite` improved the speed *again* by a factor of 10, writing the file in only 0.4 second!

Note that both `write_csv` and `fwrite` include an "automatic" mode for quotes: columns will only be quoted if separators are found in some of their values. In datasets with many columns, this ends up saving some space compared to the base R process:

    ```r
    ##               File Size.MB
    ##     baseR_file.csv   123.0
    ##     readr_file.csv   115.0
    ## datatable_file.csv   112.5
    ```


## Efficient storage for analysis

This optimisations described above are already known to most R users, who deal with plain-text files almost everyday. However, many users are unaware of the solutions that exist to optimise the frequent loading of the same files for analysis. This is particularly useful for people who work on one (or several) specific datasets for an extended period of time (weeks or even months), and regularly close and open their R session. In this context, import plain-text files everytime can take a very long time and be really frustrating, even with optimised functions such as `fread`.

Fortunately, R offers many ways to store R objects (including data frames) in a binary format, reducing the time needed to load those objects back into the environment later:

* One of the better known formats is RDATA, included in base R: it allows the user to save an object or a whole environment into a binary, compressed file, and quickly re-load the objects into memory. Saving and loading a data frame with RDATA thus recreates the exact same data frame, with the same name.
* The RDS format is very similar and also comes in base R: it works similarly and stores the data in the same way as RDATA, but allows the user to reimport an object under a different name; this is why RDS is often recommended over RDATA [LINK].
* The readr package includes an implementation of the RDS format that saves an object into a *non-compressed*, binary format - with the idea that "space is generally cheaper than time".
* Finally, the feather and fst packages aim at improving on those formats, by creating even faster saving and loading functions.

### Saving R objects

Let's now compare all of those solutions:

    ```r
    microbenchmark(save(list = "data", file = "RDATA_file.rdata"),
                   saveRDS(data, "baseRDS_file.rds"),
                   write_rds(data, "readrRDS_file.rds"),
                   write_feather(data, "feather_file.feather"),
                   write_fst(data, "fst_file.fst"),
                   times = 10, unit = "s")

    ## Unit: seconds
    ##                                            expr       min        lq
    ##  save(list = "data", file = "RDATA_file.rdata") 8.0673156 8.0755946
    ##               saveRDS(data, "baseRDS_file.rds") 8.0590230 8.1361188
    ##            write_rds(data, "readrRDS_file.rds") 0.4966769 0.5041497
    ##     write_feather(data, "feather_file.feather") 0.1691139 0.1726092
    ##                 write_fst(data, "fst_file.fst") 0.1440342 0.1460134
    ##       mean    median        uq       max neval
    ##  8.1528986 8.1288394 8.1966048 8.3445479    10
    ##  8.2308732 8.1761716 8.2695996 8.5512118    10
    ##  0.5094116 0.5117017 0.5154003 0.5165610    10
    ##  0.1776742 0.1762208 0.1795757 0.1989124    10
    ##  0.1494708 0.1478471 0.1516870 0.1610215    10

    ##                   File Size.MB
    ## 1     RDATA_file.rdata    37.2
    ## 2     baseRDS_file.rds    37.2
    ## 3    readrRDS_file.rds    70.3
    ## 4 feather_file.feather    65.1
    ## 5         fst_file.fst    50.8
    ```

It is easy to see in those results the different implementation of those binary formats: the RDATA and RDS functions in base R create files that are much smaller (37 MB), but much slower (8 seconds). On the contrary, the readr version of RDS, as well as the feather and fst formats do not compress the data (reaching sizes of ~50-70 MB) but complete the operation extremely fast (from 0.15 to 0.5 second).

### Loading R objects

We can then compare the performance of each equivalent reading function:

    ```r
    microbenchmark(load("RDATA_file.rdata"),
                   readRDS("baseRDS_file.rds"),
                   read_rds("readrRDS_file.rds"),
                   read_feather("feather_file.feather"),
                   read_fst("fst_file.fst"),
                   times = 10, unit = "s")

    ## Unit: seconds
    ##                                  expr       min        lq      mean
    ##              load("RDATA_file.rdata") 0.9555964 0.9615745 1.0295261
    ##           readRDS("baseRDS_file.rds") 0.9567269 0.9606673 1.0314218
    ##         read_rds("readrRDS_file.rds") 0.6399538 0.6862210 0.8017189
    ##  read_feather("feather_file.feather") 0.2624005 0.2770924 0.3620786
    ##              read_fst("fst_file.fst") 0.2948647 0.3006780 0.3674351
    ##     median        uq       max neval
    ##  0.9947425 1.0638597 1.2702307    10
    ##  0.9708909 1.0131944 1.2687111    10
    ##  0.8075097 0.8910611 0.9799408    10
    ##  0.3184593 0.3856978 0.5930344    10
    ##  0.3426191 0.3695636 0.6478421    10
    ```

Same results here: non-compressed files are loaded much faster (~0.3 second for feather and fst), while compressed versions take about 3 times longer (~1 second).

Ultimately the trade-off must be judged by each user for each situation, but I agree with the idea that *space is generally cheaper than time*: if storing the original CSV file is possible, then storing a smaller binary file alongside should not be a problem; saving significant time on each data import will be much more valuable.

Currently and with the example dataset used here, it seems difficult to decide of a clear winner between fst and feather. Both packages achieve very similar performance; if anything, the fst file created in our example was 25% smaller, for the same execution time.



## Verdict

Multiple solutions coexist to offer very efficient data input/output and storage in R. However, as of January 2018, the best solutions for most users and files are:

* To read permanent files (e.g. CSV): `readr::read_csv()` and `data.table::fread()` offer considerable improvement over the read.csv() function in base R. The base functions remain much more flexible if you have to deal with messy data.

* To write permanent files (e.g. CSV): `data.table::fwrite()`, added to the data.table package in 2016, includes the same optimisation features as the readr functions, but with very strong improvements in writing speeds.

* To write and read optimised binary files, to be loaded frequently for analysis: using the native RDS/RDATA is already a big improvement over keeping files in plain-text format, but the `feather` and `fst` formats offer undisputed speeds, at the small expense of lower compression.
