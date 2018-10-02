About pgbench-tools
===================

pgbench-tools automates running PostgreSQL's built-in pgbench tool in a
useful way.  It will run some number of database sizes (the database
scale) and various concurrent client count combinations.
Scale/client runs with some common characteristic--perhaps one
configuration of the postgresql.conf--can be organized into a "set"
of runs.  The program graphs transaction rate during each test,
latency, and comparisons between test sets.

Latest Features (as of september 2018 or december 2017)
================

* Compatibility for PG v 10 (sept 2018)
* **FILLFACTOR**
* graphs for **latency** for all reports
* limited_webreport followed by a comma seperated list of sets
* rates_webreport in the same manner but **only for fixed tps tests**
* **cleanups** (singlevalue, all dirty values, from a value till the end) see "Removing Bad Tests"
* latest_set:  list of tests of the current/latest set (ordered)
* list_orderbyset : lists sets ordered
* lowest_latency (and fastest tests with different degrees of compromise)
* **compromise_params**: allows to see a particular area of scale/client/tps/latency using only sql and no graph

      psql -d results -v lat=15 -v tps=700 -v lscale=900 -v hscale=1000 -v lclients=1 -v hclients=16 -f reports/compromise_params.sql

Latest bug fixes
=================

* Compatibility with V10 (transition from xlog to wal designation with pg_current_wal_lsn)
* Compatibility with 9.6+ random series generation
* log-to-csv_rates otherwise not compatible 
* fix of p90_latency not working for fixed tps rates in some cases : it generated NULL values which in turn displayed no value for latency


pgbench-tools setup
===================


* Create databases for your test and for the results::

      createdb results
      createdb pgbench

  *  Both databases can be the same, but there may be more shared_buffers
     cache churn in that case.  Some amount of cache disruption
     is unavoidable unless the result database is remote, because
     of the OS cache.  The recommended and default configuration
     is to have a pgbench database and a results database.  This also
     keeps the size of the result dataset from being included in the
     total database size figure recorded by the test.

* Initialize the results database by executing::

      psql -f init/resultdb.sql -d results

  Make sure to reference the correct database.

* You need to create a test set with a descritption::

      ./newset 'Initial Config'

  Running the "newset" utility without any parameters will list all of the
  existing test sets.

Running tests
=============

* Edit the config file to reference the test and results database, as
  well as list the test you want to run.  The default test is a
  SELECT-only one that runs for 60 seconds.

* Execute::

      ./runset

  In order to execute all the tests

Results
=======

* You can check results even as the test is running with::

      psql -d results -f reports/report.sql

  This is unlikely to disrupt the test results very much unless you've
  run an enormous number of tests already.  There is also a helper
  script named summary that shows reports/summary.sql

* A helper script named set-times will show how long past tests have taken to
  complete.  This can be useful to get an idea how long the currently running
  test or test set will actually take to finish.

* Other useful reports you can run are in the reports/ directory, including:
   * `fastest.sql`
   * `summary.sql`
   * `bufreport.sql`
   * `bufsummary.sql`
   * `compromise_params.sql` (see below)
 
* Once the tests are done, the results/ directory will include
  a HTML subdirectory for each test giving its results,
  in addition to the summary information in the results database.

* The results directory will also include its own index HTML file (named
  index.html) that shows summary information and plots for all the tests.

* If you manually adjust the test result database, you can
  then manually regenerate the summary graphs by running:

      ./webreport
      
* If you want to generate a report with selected testsets only (for example sets 1, 6 and 7):

      ./limited_webreport 1,6,7
      
* In case you test for static rates amount (for example sets 2, 8 and 9):

      ./rates_webreport 2,8,9

Test sets comparison
====================

Runs of pgbench via the runset command are oriented into test sets.  Each
test that is run will be put into the same test set until you tell the
program to switch to a new set.  Each test set is assigned both a
serial number and a test description.

New test sets are added like this:

    psql -d results -c "INSERT INTO testset (info) VALUES ('set name')"

pgbench-tools aims to help compare multiple setups of PostgreSQL.  That
might be different configuration parameters, different source code builds, or
even different versions of the database.  One reason the results database is
separate from the test database is that you can use a shared results
database across multiple test sets, while connecting to multiple test database
installations.

The graphs generated by the program will generate a seperate graph pair for
each test set, as well as a master graph pair that compares all of them.  The
graphs in each pair are graphed with a X axis of client count and database
scale (size) respectively.  The idea is that you might see whether an
alternate configuration is better at handling larger data sets, or if it
handles concurrency at high client counts better.

Note that all of the built-in pgbench tests use very simple queries.  The
results can be useful for testing read-only SELECT scaling at different
client counts.  They can also be useful for seeing how the server handles
heavy write volume.  But none of these results will change if you alter
server parameters that adjust query execution, such as work_mem or
effective_cache_size.  Many of the useful PostgreSQL parameters to tune
for better query execution on larger servers in particular fall into
this category.  You will not always be able to compare configurations
usefully using the built-in pgbench tests.  Even for parameters that
should impact results, such as shared_buffers or checkpoint_segments,
making useful comparisons with pgbench is often difficult.

There is more information about what pgbench is useful for, as well as
how to adjust the program to get better results, in the pgbench
documentation:  http://www.postgresql.org/docs/current/static/pgbench.html

Version compatibility
=====================

The default configuration now aims to support the pgbench that ships with
PostgreSQL 8.4 and later versions, which uses names such as "pgbench_accounts"
for its tables.  There are commented out settings in the config file that
show what changes need to be made in order to make the program compatible
with PostgreSQL 8.3, where the names were like "accounts" instead.

Support for PostgreSQL versions before 8.3 is not possible, because a
change was made to the pgbench client in that version that is needed
by the program to work properly.  It is possible to use the PostgreSQL 8.3
pgbench client against a newer database server, or to copy the pgbench.c
program from 8.3 into a 8.2 source code build and use it instead (with
some fixes--it won't compile unless you comment out code that refers to
optional newer features added in 8.3).

Regarding compatibility with 9.6 and higher, since the syntax for random changed, 
you can find all the necessary tests in the subfolder test-9.6/ located in the tests/ directory.
Be sure to check the name of de directory in the config script for your version of Postgres.

The default test directory scripts is `TESTDIR="tests/tests-9.6"`

No test were performed on version 10 as of yet. Please, your feedback is needed.

Multiple worker support
-----------------------

Starting in PostgreSQL 9.0, pgbench allows splitting up the work pgbench
does into multiple worker threads or processes (which depends on whether
the database client libraries haves been compiled with thread-safe 
behavior or not).  

This feature is extremely valuable, as it's likely to give at least
a 15% speedup on common hardware.  And it can more than double throughput
on operating systems that are particularly hostile to running the
pgbench client.  One known source of this problem is Linux kernels
using the Completely Fair Scheduler introduced in 2.6.23,
which does not schedule the pgbench program very well when it's connecting
to the database using the default method, Unix-domain sockets.

(Note that pgbench-tools doesn't suffer greatly from this problem itself, as
it connects over TCP/IP using the "-H" parameter.  Manual pgbench runs that
do not specify a host, and therefore connect via a local socket can be
extremely slow on recent Linux kernels.)

Taking advantage of this feature is done in pgbench-tools by increasing the
MAX_WORKERS setting in the configuration file.  It takes the value of `nproc`
by default, or where that isn't available (typically on systems without a
recent version of GNU coreutils), the default can be set to blank, which avoids
using this feature altogether -- thereby remaining compatible not only with
systems lacking the nproc program, but also with PostgreSQL/pgbench versions
before this capability was added.

When using multiple workers, each must be allocated an equal number of
clients.  That means that client counts that are not a multiple of the
worker count will result in pgbench not running at all.

Accordingly, if you set MAX_WORKERS to a number to enable this capability,
pgbench-tools picks the maximum integer of that value or lower that the client
count is evenly divisible by.  For example, if MAX_WORKERS is 4, running with 8
clients will use 4 workers, while 9 clients will shift downward to 3 workers as
the best option.

A reasonable setting for MAX_WORKERS is the number of physical cores
on the server, typically giving best performance.  And when using this feature,
it's better to tweak test client counts toward ones that are divisible by as
many factors as possible.  For example, if you wanted approximately 15
clients, it would be best to use 16, allowing worker counts of 2, 4, or 8, 
all likely to match common core counts.  Second choice would be 14,
compatible with 2 workers.  Third is 15, which would allow 3 workers--not
improving upon a single worker on common dual-core systems.  The worst
choices would be 13 or 17 clients, which are prime and therefore cannot
be usefully allocated more than one worker on common hardware.

Removing bad tests (UPDATED)
==================

If you abort a test in the middle of running, you will end up with a
bad test result entry in the results database.  These will look odd
and can distort averages and graphs.  Ideally you would erase
the entire directory each of those bad test results are in, followed by
removing their main entry from the results database.  You can do that
at a shell prompt like this::

    ./cleanup
    ./webreport 

To cleanup a single value use `./cleanup_singlevalue <testvaluenumber>`
To cleanup all values from a perticular starting point use `./cleanup_fromvalue <startingvalue>`

Known issues
============

* On Solaris, where the benchwarmer script calls tail it may need
  to use `/usr/xpg4/bin/tail` instead
  

TODO: Planned features
=======================

* The client+scale data table used to generate the 3D report would be
  useful to generate in tabular text format as well (not a priority).
* Graphs for buffers/checkpoints throughtout 
* Fix the static number of scales/clients for rates_webreport
* Fix zombie files when crash of bench on OS-stats processes
  
Documentation
=============

The documentation ``README.rst`` for the program is in ReST markup.  Tools
that operate on ReST can be used to make versions of it formatted
for other purposes, such as rst2html to make a HTML version.

Contact
=======

The original project is hosted at https://github.com/gregs1104/pgbench-tools
and is also a PostgreSQL project at http://git.postgresql.org/git/pgbench-tools.git
or http://git.postgresql.org/gitweb

The current project with the featured upgrades and bug fixes can be found at https://github.com/emerichunter/pgbench-tools

If you have any hints, changes or improvements, please contact:

 * Emeric Tabakhoff etabakhoff@gmail.com or e.tabakhoff@loxodata.com
 * Greg Smith gsmith@gregsmith.com
 

Credits
=======

Copyright (c) 2007-2014, Gregory Smith
All rights reserved.
See COPYRIGHT file for full license details and HISTORY for a list of
other contributors to the program.


******
References:
1. Introduction [1](https://emerichunter.github.io/pgbench-tools-p1/) & [2](https://emerichunter.github.io/pgbench-tools-p2/)
and in french [1](https://www.loxodata.com/post/benchmarking-pratique/) & [2](http://www.loxodata.com/post/benchmarking-pratique2/)

