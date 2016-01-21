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

    $ awk '$3 == "2016-01-01" && $2 > "11:00:00" && $2 <= "14:00:00" { print substr($3,1,5); }' | histogram
    11:00:-----------------------------------------------------------------------
    11:01:---------------------------------------------------------------
    11:02:---------------------------------------------------------------------
    11:03:--------------------------------------------------------------------
    11:04:------------------------------------------------------------------------
    11:05:----------------------------------------------------------------------
    11:06:-------------------------------------------------------------------------
    11:07:------------------------------------------------------------------------
    (... more lines, which we truncated for brevity)

You could also replace `awk` itself with the same output.

    $ histogram '$3 == "2016-01-01" && $2 > "11:00:00" && $2 <= "14:00:00" { print substr($3,1,5); }'
    
Now you can visually see where there might be troughs or spikes in the dataset.

## But I want to see the ranking of the  frequency and not the "key"? How do I do this?

Use the `-q` option. If you want to see the top hits first, use `-r` in conjunction with `-q`. 

Let's say you want to see the HTTP errors ranked by frequency.

    $ histogram -q -r '{ print $6 }'

You might get something like this:

    200:----------------------------------------------------------------------------
    304:-----------
    302:--
    403:--
    400:-
    404:-
    417:-

Of course, maybe you're more interested in the _bad hits_ only:

    $ histogram -q -r '$6 !~ /^304|200$/ { print $6 }'
    302:----------------------------------------------------------------------------
    403:-------------------------------------------------------------------
    400:------
    404:-
    417:-
    
That's more interesting.

## I like the graph output but I also want to see the _counts_

That's what the `-c` option is for. 

Maybe we want to know "how many non-200/304 hits for each 10-minute interval?" Here's how to do that:

    $ histogram -c -q '$6 !~ /^304|200$/ { print substr($2,1,4) "0" }'
    01:40:- (1)
    02:00:- (7)
    03:10:- (1)
    03:40:- (1)
    04:10:- (1)
    04:20:- (4)
    04:50:- (4)
    05:00:------------------------------------------------------------------- (2404)
    05:10:- (1)
    06:10:- (2)
    
## But why a wrapper for _awk_? Why not _perl_? 

In fact, histogram will take input from _stdin_, so do your processing however you want, and pipe to _histogram_. I chose awk because for line-oriented and field-oriented files, `awk` rules. It's syntax is simpler than `perl` (or `ruby`). And most of the time, that's what I worked with. 

If you want to help me turn histogram into a Real Perl Module™, we could make "phistogram" to wrap around `perl -lane`. 

# What won't it do currently?

* Gap-interpolation. If your output has a sequence such as `1 2 3 7 8` you may or may not notice that 4, 5, and 6 are missing.
* Statistical analysis.
* 
