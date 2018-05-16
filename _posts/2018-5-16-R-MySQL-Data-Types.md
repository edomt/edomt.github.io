---
layout: post
title: How to use R to identify variable types before importing a file to MySQL
excerpt_separator: <!--more-->
---

I recently had to import a lot of CSV files into a MySQL database. Given
that I didn't know the data and the format of the files very well, I
wrote this short R script. It prints a data.frame that indicates, for
each variable in the file:

-   Its data type in your R data.frame;
-   Some more information about the data (range for integers and
dates, maximum decimal places for floats, maximum length for
strings);
-   The corresponding data type in MySQL 8.0;
-   Whether the column includes missing values.

<!--more-->

If your files include dates and times, the use of `readr::read_*` is
highly recommended. Other import functions such as `read.*` or
`data.table::fwrite` will correctly identify numbers and strings, but
`read_*` will also recognise dates and times and convert them to the
appropriate format.

Of course the results shown by this script are purely indicative of the
data that currently exists in your file. Just because a string variable
has a maximum length of 255 characters in your data, doesn't mean that
future files won't ever include longer strings. The MySQL data types
obtained with this script should be carefully reviewed, and compared to
the theoretical specifications of the data you're working with.

```r
library(dplyr)
library(lubridate)
library(data.table)

MySQL_type <- function(dataframe, var_name) {
  message(var_name)
  column <- dataframe[[var_name]]
  has_null <- anyNA(column)
  
  if (is.factor(column)) {
    column <- as.character(column)
  }
  
  # Numeric
  if (is.numeric(column)) {
    
    r_type <- ifelse(is.integer(column), "integer", "double")
    has_negative <- any(column < 0, na.rm = TRUE)
    col_max <- max(column, na.rm = TRUE)
    
    if (r_type == "double") {
      
      max_decimal_places <- max(nchar(column %% 1)) - 2
      digits <- max(nchar(column)) - 1
      mysql_type <- case_when(
        max_decimal_places <= 7 ~ "FLOAT",
        max_decimal_places <= 15 ~ "DOUBLE",
        TRUE ~ paste0("DECIMAL(", digits, ",", max_decimal_places, ")")
      )
      
      data <- paste("max decimal places:", max_decimal_places)
      
    } else if (has_negative) {
      
      mysql_type <- case_when(
        col_max <= 127 ~ "TINYINT",
        col_max <= 32767 ~ "SMALLINT",
        col_max <= 8388607 ~ "MEDIUMINT",
        col_max <= 2147483647 ~ "INT",
        TRUE ~ "BIGINT"
      )
      
      data <- paste("range:", min(column, na.rm = TRUE), max(column, na.rm = TRUE))
      
    } else {
      
      mysql_type <- case_when(
        col_max <= 255 ~ "TINYINT UNSIGNED",
        col_max <= 65535 ~ "SMALLINT UNSIGNED",
        col_max <= 16777215 ~ "MEDIUMINT UNSIGNED",
        col_max <= 4294967295 ~ "INT UNSIGNED",
        TRUE ~ "BIGINT UNSIGNED"
      )
      
      data <- paste("range:", min(column, na.rm = TRUE), max(column, na.rm = TRUE))
      
    }
  }
  
  # Logical
  if (is.logical(column)) {
    
    r_type <- "logical"
    mysql_type <- "BOOLEAN"
    data <- ""
    
  }
  
  # Date
  if (is.Date(column)) {
    
    r_type <- "date"
    mysql_type <- "DATE"
    data <- paste("range:", min(column, na.rm = TRUE), max(column, na.rm = TRUE))
    
  } else if (is.timepoint(column)) {
    
    r_type <- "time"
    mysql_type <- "DATETIME"
    data <- paste("range:", min(column, na.rm = TRUE), max(column, na.rm = TRUE))
    
  }
  
  # String
  if (typeof(column) == "character") {
    
    text_length <- nchar(column)
    max_length <- max(text_length)
    r_type <- "character"
    data <- paste("max length:", max_length)
    
    if (length(table(text_length)) == 1 & max_length <= 255) {
      
      mysql_type <- paste0("CHAR(", max_length, ")")
      
    } else {
      
      mysql_type <- case_when(
        max_length <= 255 ~ "TINYTEXT",
        max_length <= 65535 ~ "TEXT",
        max_length <= 16777215 ~ "MEDIUMTEXT",
        TRUE ~ "LONGTEXT"
      )
      
    }
  }
  
  return(data.frame(var_name, r_type, data, mysql_type, has_null))
}

df <- ggplot2::diamonds
# Import your data.frame here. readr::read_* is recommended
# so that data types are correctly identified, including dates and times 
MYSQL_table <- rbindlist(lapply(names(df), FUN = MySQL_type, dataframe = df))
print(MYSQL_table)

##     var_name    r_type                   data        mysql_type has_null
##  1:    carat    double max decimal places: 17     DECIMAL(3,17)    FALSE
##  2:      cut character          max length: 9          TINYTEXT    FALSE
##  3:    color character          max length: 1           CHAR(1)    FALSE
##  4:  clarity character          max length: 4          TINYTEXT    FALSE
##  5:    depth    double max decimal places: 16     DECIMAL(3,16)    FALSE
##  6:    table    double max decimal places: 15            DOUBLE    FALSE
##  7:    price   integer       range: 326 18823 SMALLINT UNSIGNED    FALSE
##  8:        x    double max decimal places: 17     DECIMAL(4,17)    FALSE
##  9:        y    double max decimal places: 17     DECIMAL(4,17)    FALSE
## 10:        z    double max decimal places: 17     DECIMAL(3,17)    FALSE
```

![MySQL + R logos](https://raw.githubusercontent.com/edomt/edomt.github.io/master/images/r_mysql_logos.png)
