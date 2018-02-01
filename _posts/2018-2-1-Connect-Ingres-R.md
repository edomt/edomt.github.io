---
layout: post
title: How to connect R to an Ingres database
excerpt_separator: <!--more-->
---

If your main data is stored in an SQL database, creating a connection to query this database directly from R can save you hours of tedious data exports. The process is usually straightforward, but I recently had to set up a connection to Ingres. Unfortunately, a simple [Google query](https://encrypted.google.com/search?hl=en&q=How%20to%20connect%20R%20to%20an%20Ingres%20database) wasn't quite enough to find good documentation, since Ingres isn't as common as other relational database management systems these days.

<!--more-->

## Setting up Ingres

### Installing the driver

First, you need to choose which version of Ingres and R you want to run: both the Ingres driver and your installation of R need to either be 32-bit, or 64-bit, in order to communicate with each other.

The simplest way to determine your existing version of R is to look at the start-up message when you start R or RStudio. As of January 2018, mine currently shows:

```r
R version 3.4.3 (2017-11-30) -- "Kite-Eating Tree"
Copyright (C) 2017 The R Foundation for Statistical Computing
Platform: x86_64-w64-mingw32/x64 (64-bit)
```

This means that I'm running the 64-bit version of R, and that I need the 64-bit version of Ingres installed in order to create an interface between them.

As explained on [Actian's website](http://esd.actian.com/product/drivers), *"the Ingres ODBC Driver is included with the Actian Vector and Ingres Client Runtime and DBA Tools Packages."* This means that if Ingres is already installed on your computer (again, in the right version!), the driver should be available already.

The easiest way to check this is to open the Windows Start Menu, type "ODBC" in the Search box, and open "XX-bit ODBC Data Source Administrator" (where XX is the version you want to use).

!(https://raw.githubusercontent.com/edomt/edomt.github.io/master/images/start.png)

If you see Ingres in the Drivers tab, then you're good to go. Otherwise, you'll need to [download the Ingres Client Runtime](http://esd.actian.com/product/drivers) that includes the driver.

!(https://raw.githubusercontent.com/edomt/edomt.github.io/master/images/odbc.png)

Depending on the way you've installed Ingres on your computer, you might see different names in the Name column of the Drivers tab, including "Ingres", "Ingres CR", "Ingres VT" or "Ingres II". Take a note of the one you have, as we'll need that information later on.


### Setting up your vnode

A vnode (*virtual node*) is what tells Ingres how to connect to your server. Before we go to R, we need to make sure the right vnode is set up so that R can use it to connect to Ingres.

For this, we need the Actian Network Utility. Again, the easiest way to find it will be to search for "Actian Network Utility" in the Windows Start Menu. A small window will open, with a list of "Nodes/Connections" (potentially empty if this is your first vnode).

To add a vnode, go to Node > Add, and configure a connection to your server hosting Ingres. The configuration here will depend greatly on how your system is set up, so ask for help around you if you're not sure what to use!

!(https://raw.githubusercontent.com/edomt/edomt.github.io/master/images/add_vnode.png)

Once the vnode is created, you should test it by right-clicking it in the "Nodes/Connections" list, and click on "Test Node". You should then see the following dialog box:

!(https://raw.githubusercontent.com/edomt/edomt.github.io/master/images/vnode_test.png)

Success! We can now finally move to R.


## Querying Ingres from R

There are various ways to open a connection between R and a database, but I've found the RODBC package to be the easiest to use with Ingres. Let's install and load it:

```r
install.packages('RODBC')
library(RODBC)
```

And then, here is the simple code you need to open a connection, get data from a query, and close the connection:

```r
dbhandle <- odbcDriverConnect('driver={DRIVER};server=VNODE;database=DATABASE;uid=USERNAME;pwd=PASSWORD'))
result <- sqlQuery(dbhandle, QUERY)
odbcClose(dbhandle)
```

On the first line:

* ***DRIVER*** should be replaced by the name of the Ingres driver that you wrote down earlier.
* ***VNODE*** should be replaced by the name of your vnode in Actian Network Utility.
* ***DATABASE*** should be replaced by the name of the database you want to connect to.
* ***USERNAME*** and ***PASSWORD*** should be replaced by the relevant credentials. You must use the same username/password than when you created the vnode in Actian Network Utility.

On the second line, ***QUERY*** should simply be a string containing an SQL query, such as `"SELECT * FROM my_table"`.

Finally, the third line closes the connection. You can of course run multiple queries before closing the connection.

At the end of this process, `result` should be a data.frame in your environment, containing the results of your query.


### Wrapping queries in a simpler function

To avoid writing those lines multiple times in my code, I use the following wrapper function to open a connection to a given database/server, run a query, transform the resulting data.frame into a data.table with the `setDT()` function (since I do most of my work with the data.table package), and close the connection:

```r
ingres <- function(db, server, query) {
  dbhandle <- odbcDriverConnect(paste0('driver={Ingres};server=', server, ';database=', db, ';uid=USERNAME;pwd=PASSWORD')) # Don't forget to replace USERNAME and PASSWORD here!

  result <- sqlQuery(dbhandle, query)
  setDT(result)

  odbcClose(dbhandle)
  return(result)
}
```

