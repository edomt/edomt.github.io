---
layout: post
title: Efficient file input, output and storage in R
excerpt_separator: <!--more-->
---

Whether used in academia, industry or journalism, working with R involves importing and exporting a lot of data. While the basic functions to read and write files are known to all users, different methods have been developed over the years to optimise this process.

In this article, we'll have a look at the most efficient ways to read and write permanent files (i.e. in plain-text formats such as CSV), and to save and load binary files, a solution often overlooked by R users but much better suited to regular analysis of a given dataset.

<!--more-->


## Setting up our benchmark

We'll be using functions from four different packages ([readr](https://cran.r-project.org/web/packages/readr/), [data.table](https://cran.r-project.org/web/packages/data.table/), [feather](https://cran.r-project.org/web/packages/feather/), and [fst](https://cran.r-project.org/web/packages/fst/)), and comparing their performance using the [microbenchmark](https://cran.r-project.org/web/packages/microbenchmark/) package.

```r
install.packages(c("microbenchmark",
                   "readr",
                   "data.table",
                   "feather",
                   "fst"))

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
* `read_csv` from the `read_*` series of functions in the readr package;
* `fread` from the data.table package.

Let's use microbenchmark to import our file using those three functions; the `microbenchmark()` function will execute each expression 10 times and average the elapsed time.

```r
microbenchmark(data <- read.csv(filename),
               data <- read_csv(filename),
               data <- fread(filename),
               times = 10, unit = "s")

## Unit: seconds
##                        expr       min        lq      mean    median        uq       max neval
##  data <- read.csv(filename) 25.662873 26.829712 27.915110 27.344555 27.681976 35.389826    10
##  data <- read_csv(filename)  2.303900  2.502276  2.971335  3.020295  3.329171  3.618546    10
##     data <- fread(filename)  3.273398  3.467531  3.707234  3.777934  3.938012  4.146957    10
```

The `read_csv` and `fread` functions imported our file in 3 seconds, against 28 seconds for the `read.csv` function. This improvement is mostly due to the way those two functions identify the type of each column - by guessing it based on a sample of values.

More generally, `read_csv` and `fread` tend to assume that your file is quite "clean", more than `read.csv` does: import functions in base R offer a lot of optional arguments to deal with comments, missing values, trailing spaces in strings, etc.

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
##                                              expr        min         lq       mean     median         uq        max neval
##  write.csv(data, "baseR_file.csv", row.names = F) 13.8066424 13.8248250 13.9118324 13.8776993 13.9269675 14.3241311    10
##                 write_csv(data, "readr_file.csv")  3.6742610  3.7999409  3.8572456  3.8690681  3.8991995  4.0637453    10
##                fwrite(data, "datatable_file.csv")  0.3976728  0.4014872  0.4097876  0.4061506  0.4159007  0.4355469    10
```

The results are impressive: readr improved our writing time from 14 seconds in base R to 4 seconds with `write_csv` - but `fwrite` improved this performance *again* by a factor of 10, writing the file in only 0.4 second!

Note that both `write_csv` and `fwrite` include an "automatic" mode for quotes: columns will only be quoted if necessary, i.e. if separators are found in some of their values. In datasets with many columns, this can save space compared to the base R process:

```r
##               File Size.MB
##     baseR_file.csv   123.0
##     readr_file.csv   115.0
## datatable_file.csv   112.5
```


## Efficient storage for analysis

The optimisations described above are known to most users who deal with plain-text files almost everyday. However, many are unaware of the solutions that exist to optimise the frequent loading of the same files for analysis. This is particularly useful for people who work on one (or several) specific datasets for an extended period of time (weeks or even months), and regularly close and open their R session. In this context, importing plain-text files every time can be very long and frustrating, even with optimised functions such as `fread`.

Fortunately, R offers many ways to store R objects (including data frames) in a binary format, reducing the time needed to load those objects back into the environment later:

* One of the better known formats is RDATA, included in base R: it allows the user to save an object or a whole environment into a binary, compressed file, and quickly re-load the objects into memory. Saving and loading a data frame with RDATA thus recreates the exact same data frame, with the same name.
* The RDS format is very similar and also comes in base R: it works similarly and stores the data in the same way as RDATA, but allows the user to reimport an object under a different name.
* The readr package includes an implementation of the RDS format that saves an object into a *non-compressed*, binary format - with the idea that *"space is generally cheaper than time"*.
* Finally, the feather and fst packages aim at improving on those formats, by creating even faster saving and loading solutions.

### Saving R objects

Let's now compare all of those solutions.

A couple of notes:

* By default the `saveRDS()` function uses compression, but it's possible to disable it with `compress = FALSE`, so we'll include this possibility as well.
* The `write_fst()` function can take an argument `compress = N` where N is a value in the range 0 to 100, indicating the amount of compression to use. The default is 50, but we'll test both extremes (0 and 100).

```r
microbenchmark(save(list = "data", file = "RDATA_file.rdata"),
               saveRDS(data, "baseRDS_comp_file.rds", compress = T),
               saveRDS(data, "baseRDS_noncomp_file.rds", compress = F),
               write_rds(data, "readrRDS_file.rds"),
               write_feather(data, "feather_file.feather"),
               write_fst(data, "fst_comp0_file.fst", compress = 0),
               write_fst(data, "fst_comp100_file.fst", compress = 100),
               times = 10, unit = "s")

## Unit: seconds
##                                                     expr        min         lq       mean      median         uq       max neval
##           save(list = "data", file = "RDATA_file.rdata") 7.96338920 8.01461160 8.05888978  8.04906326 8.10588731 8.1770513    10
##     saveRDS(data, "baseRDS_comp_file.rds", compress = T) 7.88657332 7.96057057 8.11137097  8.05012876 8.08823067 8.7744078    10
##  saveRDS(data, "baseRDS_noncomp_file.rds", compress = F) 0.36618762 0.37159785 0.38247426  0.38253665 0.38849595 0.4012216    10
##                     write_rds(data, "readrRDS_file.rds") 0.33957280 0.34512827 0.36789954  0.36944088 0.38139510 0.4033564    10
##              write_feather(data, "feather_file.feather") 0.11037106 0.11065625 0.11337884  0.11259769 0.11543701 0.1191594    10
##      write_fst(data, "fst_comp0_file.fst", compress = 0) 0.08293022 0.08501644 0.08889665  0.08691963 0.09289635 0.1002668    10
##  write_fst(data, "fst_comp100_file.fst", compress = 100) 2.16989012 2.19063069 2.24112161  2.24799947 2.27968396 2.3116543    10

##                       File Size.MB
## 1         RDATA_file.rdata    37.2
## 2    baseRDS_comp_file.rds    37.2
## 3 baseRDS_noncomp_file.rds    70.3
## 4        readrRDS_file.rds    70.3
## 5     feather_file.feather    65.1
## 6       fst_comp0_file.fst    65.3
## 7     fst_comp100_file.fst    32.9
```

It is easy to see in those results the different implementations of those binary formats:

* Among the compressed files, the RDATA and RDS functions in base R create files that are much smaller (37 MB), but much slower (8 seconds). But the `compress = 100` version of fst is even more compressed (33 MB) and only took 2.2 seconds to write to disk!
* When compression isn't required, all implementations generate a file around 65-70 MB. `saveRDS` and `write_rds` took about 0.37 second, `write_feather` only 0.11 second... and `write_fst` with `compress = 0` only 0.08 second.


### Loading R objects

We can then compare the performance of each equivalent reading function:

```r
microbenchmark(load("RDATA_file.rdata"),
               readRDS("baseRDS_comp_file.rds"),
               readRDS("baseRDS_noncomp_file.rds"),
               read_rds("readrRDS_file.rds"),
               read_feather("feather_file.feather"),
               read_fst("fst_comp0_file.fst"),
               read_fst("fst_comp100_file.fst"),
               times = 10, unit = "s")

## Unit: seconds
##                                  expr       min        lq      mean    median        uq       max neval
##              load("RDATA_file.rdata") 0.8775526 0.8892134 0.9321619 0.9253322 0.9480082 1.0236830    10
##      readRDS("baseRDS_comp_file.rds") 0.8749464 0.8875014 0.9349717 0.9151411 0.9591819 1.0452654    10
##   readRDS("baseRDS_noncomp_file.rds") 0.5509024 0.5629992 0.6108047 0.5701881 0.7095914 0.7261663    10
##         read_rds("readrRDS_file.rds") 0.5769474 0.5951807 0.6423258 0.6052121 0.6814075 0.8037006    10
##  read_feather("feather_file.feather") 0.1927875 0.1976274 0.2614764 0.2484733 0.3249248 0.3466373    10
##        read_fst("fst_comp0_file.fst") 0.2129975 0.2145049 0.2475619 0.2207756 0.2458871 0.3549952    10
##      read_fst("fst_comp100_file.fst") 0.2835336 0.2865808 0.3434378 0.3261273 0.3608169 0.4697726    10
```

Here we see that compression and time-to-load are entirely correlated: non-compressed files are *generally* loaded faster (~0.6 second for non-compressed RDS, ~0.25 second for feather and fst with `compress = 0`), while compressed versions take much longer (~0.93 second for RDS and RDATA in base R). However, the fst version with `compress = 100` only took ~0.34 second to load, which is much faster than other compressed files, and not that much longer than the uncompressed solutions!

Ultimately, the trade-off must be judged by each user for each situation, but I would agree with the idea that *space is generally cheaper than time*: if storing the original CSV file is possible, then storing a smaller binary file alongside should rarely be a problem; and saving significant time on each data import will be much more valuable.

However, `write_fst` seems to achieve a great balance, by offering flexibility (you can choose your own compression value anywhere between 0 and 100, with a default of 50) but still loading with very compressed files extremely fast.


## Verdict

Multiple solutions coexist to offer very efficient data input/output and storage in R. However, as of January 2018, the best solutions for most users and files are:

* To read permanent files (e.g. CSV): `readr::read_csv()` and `data.table::fread()` offer considerable improvement over the `read.csv()` function in base R. The base functions remain much more flexible if you have to deal with messy data.

* To write permanent files (e.g. CSV): `data.table::fwrite()`, added to the data.table package in 2016, includes the same optimisation features as the readr functions, but with very strong improvements in writing speeds.

* To write and read optimised binary files to be loaded frequently for analysis: using the native RDS/RDATA is already a big improvement over keeping files in plain-text format, but the `feather` and `fst` formats offer undisputed speed improvements. At this stage I would recommend `fst`, because it does give the user the option of high compression if needed, while still loading the resulting file very quickly.
