# FAQ

## Why would I want to use this program?

To quickly find patterns in large heaps of line-oriented data. 

For instance, let's say you run an Apache web server, and on 2pm Jan 1st, users reported 500 errors.
The ssl_error.log and error.log don't show anything unusual here (except the 500 report). Maybe
there was a pattern in the number of hits at that time. You can use *histogram* on the access_log file
to do some kind of hit-frequency analysis. 

Let's further assume you've done the intelligent thing and customized your log file as a TDF 
(tab-delimited format). A sample input line looks like

    1453381384 14:03:04 2016-01-21 4 03     200             10.0.0.4    -       sample.my.org  443     HTTP/1.1        TLSv1.2 GET    / ...

For this file, you can use `awk` to find lines in a target date-time range in this way:

    awk '$3 == "2016-01-01" && $2 > "11:00:00" && $2 <= "14:00:00"'

This will tell awk to examine the 3rd field to match the desired, and the 2nd field to match a time range. 
(`awk` has other ways to do a time range, but this is the simplest example to show.). To get a feel for how many log lines
occurred each minute in that time range, we can tell `awk` to output the hour and minute, then pass through sort,
(hits don't always occur in order), then uniq -c. Like this:

    awk -F\\t '$3 == "2016-01-01" && $2 > "11:00:00" && $2 <= "14:00:00" { print substr($3,1,5); }' | sort | uniq -c

With output (first 8 lines) like:
     70 11:00
     62 11:01
     68 11:02
     67 11:03
     71 11:04
     69 11:05
     72 11:06
     71 11:07

*Histogram is a wrapper for `awk .. | sort | uniq -c` and pretty-prints the output.* And more. To do this with histogram, just replace
replace `|sort|uniq -c`| with `|histogram`, and the output looks like this:

    $ awk -F\\t '$3 == "2016-01-01" && $2 > "11:00:00" && $2 <= "14:00:00" { print substr($3,1,5); }' | sort | uniq -c11:00:-----------------------------------------------------------------------
    11:01:---------------------------------------------------------------
    11:02:---------------------------------------------------------------------
    11:03:--------------------------------------------------------------------
    11:04:------------------------------------------------------------------------
    11:05:----------------------------------------------------------------------
    11:06:-------------------------------------------------------------------------
    11:07:------------------------------------------------------------------------
    (... more lines, which we truncated for brevity)

But
