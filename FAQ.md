# FAQ

### Why would I want to use this program?

To quickly find patterns in large heaps of line-oriented data. 

For instance, let's say you run an Apache web server, and on 2pm Jan 1st, users reported 500 errors.
The ssl_error.log and error.log don't show anything unusual here (except the 500 report). Maybe
there was a pattern in the number of hits at that time. You can use *histogram* on the access_log file
to do some kind of hit-frequency analysis. 

Let's further assume you've done the intelligent thing and customized your log file as a TDF 
(tab-delimited format). A sample input line looks like

    1453381384 14:03:04 2016-01-21 4 03     200             10.0.0.4    -       sample.my.org  443     HTTP/1.1        TLSv1.2 GET    / ...

For this file, you can use `awk` to find lines in a target date-time range in this way:

    awk '$3 == "2016-01-01" && $2 > "11:00:00" && $2 <= "14:00:00"' access.log

This will tell awk to examine the 3rd field to match the desired, and the 2nd field to match a time range. 
(`awk` has other ways to do a time range, but this is the simplest example to show.). To get a feel for how many log lines
occurred each minute in that time range, we can tell `awk` to output the hour and minute, then pass through sort,
(hits don't always occur in order), then uniq -c. Like this:

    awk '$3 == "2016-01-01" && $2 > "11:00:00" && $2 <= "14:00:00" { print substr($3,1,5); }' access.log | sort | uniq -c

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

    $ awk '$3 == "2016-01-01" && $2 > "11:00:00" && $2 <= "14:00:00" { print substr($3,1,5); }' access.log | histogram
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

    $ histogram '$3 == "2016-01-01" && $2 > "11:00:00" && $2 <= "14:00:00" { print substr($3,1,5); }' access.log
    
Now you can visually see where there might be troughs or spikes in the dataset.

### I want to see the ranking of the  frequency and not the "key"? How do I do this?

Use the `-q` option. If you want to see the top hits first, use `-r` in conjunction with `-q`. 

Let's say you want to see the HTTP errors ranked by frequency.

    $ histogram -q -r '{ print $6 }' access.log

You might get something like this:

    200:----------------------------------------------------------------------------
    304:-----------
    302:--
    403:--
    400:-
    404:-
    417:-

Of course, maybe you're more interested in the _bad hits_ only:

    $ histogram -q -r '$6 !~ /^304|200$/ { print $6 }' access.log
    302:----------------------------------------------------------------------------
    403:-------------------------------------------------------------------
    400:------
    404:-
    417:-
    
That's more interesting.

### I like the graph output but I also want to see the _counts_

That's what the `-c` option is for. 

Maybe we want to know "how many non-200/304 hits for each 10-minute interval?" Here's how to do that:

    $ histogram -c -q '$6 !~ /^304|200$/ { print substr($2,1,4) "0" }' access.log
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

### How do I exclude these trivial results from the output?

Use the -t (threshold) option and provide either an absolute number, or a number followed by %. As a shortcut, you can simply specify `-N` or `-N%`. (The option `-1` is special and means force one-line output.) 

Let's say you want to see the different IP addresses that sent requests to your web server. In my log, that's the 4th _tab-delimited_ field. I do this:

    $ histogram -c -F\\t '{ print $4 }' access.log
    
I get hundreds of lines of output, mostly from IP addresses that hit just once or twice. Let's ask for the top 95% by setting the threshold at 5%: 

    $ histogram -c -5% -F\\t '{ print $4 }' access.log
    172.232.42.39:---- (2415)
      38.132.21.6:-------------------------------------------------------------  (37329)
      38.132.13.9:------------------------------------------------------------------  (40487)
          (TOTAL): (80387)

Or I could set a minimum number:

    $ histogram -c -500 -F\\t '{ print $4 }' access.log
    172.232.42.39:---- (2415)
      38.132.21.6:-------------------------------------------------------------  (37329)
      38.132.13.9:------------------------------------------------------------------  (40487)
     112.15.3.104:- (52)
          (TOTAL): (80387)

### The output for each item is split along two lines! How do I fix this!

Probably with the `-1` option. You can also widen your terminal, or specify the terminal width with `-w`, or specify the maximum label size with `-k`. 

_histogram_ determines the terminal width from the COLUMNS environment variable, or 80 if that isn't set. It sizes the graphing area by looking at the length of the longest key (ie, item, or axis-label), and working backwards from there. If they maximum key length leaves less than 60 characters for the graph, it outputs the key on its own line, followed by the graph on a line underneath. 

The `-1` option forces output to one line if it can, truncating each key to 20% of the terminal width. You can set the key width manually with `-k N`. These options affect output only.

The `-x` makes the output less pretty, forces everything to one line, and is useful for processing with other tools. This is quite useful if you want to output the counts with `-c` and then do further processing (with another program or later in a pipeline) on the keys and counts. 

### How do I get a reminder of the available options?

Use `-h` for a short help section, or `-H` for a fuller one. If you use `-?` you will get help from awk. 

### How do I pass option `<whatever>` to awk?

Any options not handled by _histogram_ are passed along to awk. This includes notably `-F` and `-v`. At the moment, there is no way to pass conflicting options. But this would be trivial to add.

### I pass my options to `histogram` but they're not working!!

Probably you're putting your histogram options _after_ your awk options. They need to come before. Once histogram sees an option it doesn't recognize, it stops processing options. (Otherwise, we'd have to figure out the quirks of processing awk's options). 

### Why a wrapper for _awk_? Why not _perl_? 

In fact, histogram will take input from _stdin_, so do your processing however you want, and pipe to _histogram_. I chose awk because for line-oriented and field-oriented files, `awk` rules. It's syntax is simpler than `perl` (or `ruby`). And most of the time, that's what I worked with. 

If you want to help me turn histogram into a Real Perl Module™, we could make "phistogram" to wrap around `perl -lane`. 

### What won't it do currently?

* Gap-interpolation. If your output has a sequence such as `1 2 3 7 8` you may or may not notice that 4, 5, and 6 are missing.
* Statistical analysis.
* 
