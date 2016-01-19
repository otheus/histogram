#!/usr/bin/perl
use strict;

=pos

=head1 NAME

histogram - output a bar-graph representing frequency of inputs

=head1 SYNOPSIS

Normal operation:

B<histogram> [I<options>] ['I<awk-program>'] [<input-file> ...]

Provide summary usage information concerning options:

B<histogram> -h 

Provide detailed usage information concerning options:

B<histogram> -H

=head1 DESCRIPTION

Creates a histogram of the input data, using an awk-defined filter,
and outputs to the tty a horizontal bar-graph representing the histogram.

After all I<options> are processed, the first non-option argument is
tested to see if it is a file that exists; if not, it is treated as an
I<awk-program> and is provided to B<awk>. Otherwise, the I<awk-program>
provided to awk is simply C<{print}>, so that awk behaves like B<cat>. All
arguments, including any awk-specific options and input-files, are
passed as arguments to awk. Standard-input is also passed to awk. The
output from awk is then read in by B<histogram>, and each input line is
grouped and counted. After all inputs have been consumed, B<histogram>
outputs a bar graph, where each output line represents one of the input
lines and the frequency of that line's occurrence in the input stream.
Unless otherwise specified (B<-w>), the entire width of the TTY is used,
with the graph for the item of highest frequency using the full line
width.  If an item is found only once, its frequency will be represented
by at least one glyph.  A threshold (B<-t>) can be set so that
only items which appear I<n> times in the input will appear in the output.

The output is by default sorted in alphanumeric order by the input line,
or optionally by frequency (B<-q>) of occurrence. This output order
can be reversed (B<-r>).  Optionally, the output can display an exact
count (B<-c>) beside each graph. Unless otherwise specified (with either
B<-x> or B<-k>), the keys are right-aligned to a column which is the lesser of
the longest key-length or 20% of the TTY width. If the longest key is
greater than this 20%, the output for each item is split onto two lines,
one for the key and the other for the graph, but one-line output can be
forced (B<-x>, B<-1>). The graph will be scaled from 1 to the maximum 
frequency, but it can be also scaled from the minimum frequency found 
to the maximum (B<-m>). The maximum can be treated as the total number
of items found and scaled accordingly; in this case, a graph representing
this total is shown for comparison purposes (B<-T>).

An I<awk-program> looks like this:
   <expression> { I<program> } 

See the B<awk> manual for how to formulate I<expression> and
I<program>. It's usually as simple as a regular expression. See
L<EXAMPLES> below.

=head1 OPTIONS

Many, some of which are passed to awk. See the output of either -h or -H.
The input-files, if any, are passed to awk; if none are provided, awk
will use B<histogram>'s standard-input. The options -f, -F, -v, and -W are
typical awk options; B<histogram> always passes these arguments on to awk.

=head1 ENVIRONMENT

=over 5

=item AWKPATH

The path of the awk program to be used.

=item COLUMNS

The width of the TTY, usually changed by the shell upon SIGWINCH, but can
be overridden on the command line with B(<-w>).

=back

=head1 RETURN VALUE

Zero on help and if something was output and no error.
One if there was no input from awk to be processed (grep-like behavior).
Otherwise, the error-code from awk is given. 

=head1 SEE ALSO

awk

=cut


sub usage { 
    my $detail = (shift || 0);
    local $0;
    $0 =~ s:^.*/::g;
    print <<"USAGE"; 

Usage: $0 [<options>] [<awk-options>] ['<awk-filter>'] [<input-files>]

  Outputs to the tty a simple horizontal bar graph, each bar representing 
  the number of occurences of unique items found in the input files or stream.

  Where <awk-options> are anything awk recognizes, the awk-filter is the 
  usual '/pattern/ {program}' syntax of awk, and any input-files (or standard 
  input if none given). 

USAGE

  print <<"SUMMARY" 
  Option summary. Use -H for detailed help.
    -t N|N% -N -N%       set threshold to N or N%
    -q -r                sort by freQuency, and/or Rerverse the order
    -w N                 set output width
    -k N                 set maximum key width
    -T -c -m -1 -x       other output options -- run $0 with -H
SUMMARY
    if $detail == 0 ; 

  print <<'DETAIL'
  Counting options: 
    -t N  Threshold  - an item must be present N times 
    -t N% Threshold  - an item must be present in at least N% of total
    -N    shortcut for -t N (where N is an integer)
    -N%   shortcut for -t N% (where N is a decimal number)
          An item must be present at least once, so -t 1 means nothing,
          and thus -1 is used below for the "single-line" option.
 
  Output options:
    -T    Act as if the total number of items is an item;
          the last output line is a graph representing the total;
          all graphs are scaled to this one.
    -c    Show counts at end of each graph
    -m    Scale graph from min to max - default is 1 to max
    -q    Sort by freuQency - default is sort by keyname (ignoring case)
    -r    reverse sort order
 
  Display options:
    -w N  display width (or use COLUMNS environment variable or 80). 
    -k N  display k characters of key - default is term-width minus 60
    -1    Force output to use a single line for each item, and set the key-
          width to 20% of -w, truncating longer keys. This is useful because 
          if -k is not specified and if the largest key is greater than the
          default key-width, output is separated onto two lines. 
    -x    Like -1, but keys are not right-aligned, but flush-left and never 
          truncated. Scale the graph as if each key had 0 characters.
          This is useful for outputting to a file for further processing.

  Typical AWK options:

   -F<r>    Fields are separated by the regular expression <r>. By default,
            awk uses any whitespace to separate fields. 
            Use \\\\t for tab-separated.
 
   -v <var>=<value>
            Assign <var> a <value>. <var> is then used in the script and 
            replaced with <value>. For instance:
               awk -v pi=3.14 '{ print pi*$1*$1}'  
            would print the area of a circle whose radius is the first field 
            of input.
DETAIL
    if $detail == 1 ;
  print "\n";
   0;
}

###############################################################################
###############################################################################
## INTERNAL DOCUMENTATION
###############################################################################
###############################################################################

# Program structure
#  1. Parse options and handle environment variables
#     a. Extract and validate options we use internally
#     b. Handle awk options and command line arguments; validate
#     c. Handle TTY-related environment settings and options
#  2. Read input lines and store keys/counts in hash
#     Track max key-width and total item-count
#     Close input file handle
#  3. Scale data based on input key-width, max frequency, 
#     TTY width, and user-specified options.
#  4. Output 

#
# #1a# - Parse Options
#      for various reasons, the optargs module won't work here.
#      so we build our own. To make 'strict' mode happy, we 
#      specify the possible variables with "our"
#
our($opt_c,$opt_q,$opt_r,$opt_1,$opt_m,$opt_x,$opt_T,$opt_h,$opt_H);
our($opt_t,$opt_w,$opt_k,$opt_n);

while ($ARGV[0]) { 
  $_=$ARGV[0];
  if ( /^-(c|q|r|1|m|x|T|h|H)$/ ) { 
    eval "\$opt_$1 = 1";
  }
  elsif ( /^-(\d+|[0-9]+(?:\.[0-9]*)?%)$/ ) { 
    $opt_t = $1;
  }
  elsif ( /^-(w|k|t|n)$/ ) {
    eval "\$opt_$1 = q($ARGV[1])";
    shift @ARGV;
  }
  else {
    last;
  }
  shift @ARGV;
} 

# User needs help. Give it and exit.
if ($opt_h) { &usage; exit 0; }
if ($opt_H) { &usage(1); exit 0; }

# Warn about incompatible options
if (defined $opt_x && ( defined $opt_k || defined $opt_1 )) { 
  print STDERR "$0: WARNING: Option -x overrides key-width settings (-k,-1)\n";
}

#
# #1b#  - Now handle awk arguments ...
#

# define variables for this part -- again for strict mode
our $awkcommand;
my ($AWKPATH,@awkfiles,@awkopts,$awkfilter);
my $uses_external_script;

# Get the path to the awk that will be used.
$AWKPATH=($ENV{'AWKPATH'} or "awk");

# 
# #1b.1#  - find awk options and arguments on command line
# 

# First, assume the arguments at the end are filenames -- if they exist. 
# If not, leave them alone.
while (-f $ARGV[-1]) { 
  push @awkfiles, pop @ARGV;
}

# first, any awk options. Handle Multipart options too
while ($ARGV[0] =~ /^-/) {
  $_=shift @ARGV;
  push @awkopts,qq('$_');
  if ( /^--$/ ) {
    last;
  }
  #elsif (/^--\w+/) { } # do nothing
  #elsif (/^-F.+/) { }  # do nothing
  elsif (/^-f$/) { 
    $uses_external_script=1;
    push @awkopts,"'".(shift @ARGV)."'";
  }  
  elsif ( /^-(v|W|m[fr]|F)$/ ) { 
    push @awkopts,"'".(shift @ARGV)."'";
  }
}

#
# #1b.2#  - Build awk command
#

# Finally, the remaining arggument is the filter/program. if not, use "{print}"
# so that awk acts basically like "cat". If external program is specified with
# -f option, then don't use any filter (unless one is also provided). 
# lastly, put single-quotes around any filter so that it's handled properly by
# the shell.

$awkfilter=$ARGV[0];
$awkfilter="{print}" if (!defined $ARGV[0] && !$uses_external_script);
$awkfilter="'$awkfilter'" if defined $awkfilter;

# build the entire command string
$awkcommand=join(" ",$AWKPATH,@awkopts,$awkfilter,@awkfiles);

# 
# #1b.3#  - Validate awk command
#

# Now open the awkcommand with perl's pipe-open.
# Do it now in to detect failure ASAP.
open(FILTER,$awkcommand." |") || do {
  print "$0: ERROR: Failed to run awk command: $!\n";
  print "    ".$awkcommand."\n";
  exit ($? >> 8);
};

# 
# #1c#   - Handle TTY environment and options
#

# Process internal values from other arguments, such as terminal width.
# Again, define variables to make "strict" happy.
my ($termwidth, $keywidthmax, $lines, $cwidth, $scale, $use_two_lines);

$termwidth  = (defined $opt_w ? $opt_w : ($ENV{'COLUMNS'} || 80));
if ($opt_1 && !defined $opt_k) { $opt_k = $termwidth / 5; }
$keywidthmax = (defined $opt_k ? $opt_k : ($termwidth - 60));
# $cwidth defined once max-count is known

#
# #2#  -  read output from awk and collect statistics
#      each line is the key, and its value is the count
#      track max and min.
#
my $min = 1;
my $max = 1;
my $kwidth = 0;
my %h;

while (<FILTER>) { 
  chop;
  ++$h{$_};
  $max = $h{$_} if ($h{$_} > $max);
  if (!defined $opt_x && !defined $opt_t && length($_) > $kwidth)  {
    $kwidth = length($_);
  }
}
$lines=$.; # $. goes away once we do a close. close() needed to get $?
close (FILTER);
exit $?>>8 if ($?); # exit if awk failed

if ($lines == 0) { 
  print STDERR "$0: WARNING: No output from awk\n";
  exit 1; 
}

# 
# #3#   - post-processing
# 

# #3a#  - threshold handling
#	Go through all entries and remove those that don't meet the threshold.
#	reset kwidth to match those that make the cut.
#
if ($opt_t) { 
  $kwidth = 0;
  if (substr($opt_t,-1,1) eq "%") { 
    # User provided a percentage. Determine actual based on known max 
    $opt_t = ($max + 0.0) * substr($opt_t,0,length($opt_t)-1) / 100.0;
  }
  while ( my($key,$val) = each(%h) ) { 
    if ($opt_t > $val) {
      delete $h{$key} 
    }
    elsif (!defined $opt_x && length($key) > $kwidth)  {
	# recalc kwidth again, since we removed keys
	$kwidth = length($key);
    }
  }
}

#
# #3b# - output scaling
#      Find min for scaling -- cannot do this in the read-loop
#      start with the known max and find the lowest of the values
#      (separate loop from opt_t loop to avoid messy code)
#
if ($opt_m) { 
  $min = $max;
  while ( my($key,$val) = each(%h) ) { 
    $min = $val if ($val < $min);
  }
}

if (defined $opt_x) { 
  # If opt_x, set kwidth to 0 for $scale to work properly
  # then subtract 1 to negate the offset used for the colon
  $kwidth = -1;
}
elsif (defined $opt_k) { 
  # if -k was specified, its value is in keywidthmax
  $kwidth = $keywidthmax;
}
elsif ($kwidth > $keywidthmax) { 
  # it won't fit on one line, so separate into two.
  $kwidth=0;
  $use_two_lines=1;
}

# if -T, the max is the total
$max = $lines if $opt_T;

# for proper scaling, adjust max and min downward by min.
$max -= $min;

$cwidth = (defined $opt_c ? length("".$lines) + 3 : 0);
$scale= ($termwidth - 1 - $kwidth - $cwidth) / $max;
#                     |
# for the colon ------^

$kwidth=1 if $kwidth==0; # kwidth needs to be at least 1 from now on.

# 
#  #4#  - output graph
# 
foreach (&do_sort(\%h, $opt_r))  {
  if ($opt_x) {
    print $_,":";
  }
  elsif ($use_two_lines) { 
    print $_,":\n";
  }
  else { 
    printf "%*s:",$kwidth,substr($_,0,$kwidth);
  }
  my $graphlen = int(0.9999 + $scale * ($h{$_} - $min));
  print "-" x ($graphlen ? $graphlen : 1);
  print " (".$h{$_}.")" if $opt_c;
  print "\n";
}
if ($opt_c || $opt_T) { 
  printf "%*s:%s",$kwidth,"(TOTAL)",($opt_T ? "-" x int($scale * $max) : "");
  printf " (%d)",$lines if $opt_c;
  print "\n";
}
exit 0;


#
# Internal Sort routine
#   Arg1 -- ref to hash containing histogram key/count pairs
#   Arg2 -- whether output should be reversed (1) or not (0)
#   Arg3 -- sort by frequency (1) or key (0)
#
sub do_sort { 
  my $rhash = shift;
  my $opt_r = shift;
  my $opt_q = shift;

  if ($opt_q) {
    # sort by frequency of ocurrence
    #  (optimize for spbeed, not code size)
    if ($opt_r) { 
      return sort { ($rhash->{$b} <=> $rhash->{$a} || lc($rhash->{$b}) cmp lc($rhash->{$a})) } keys %$rhash;
    } else {
      return sort { ($rhash->{$a} <=> $rhash->{$b} || lc($rhash->{$a}) cmp lc($rhash->{$b})) } keys %$rhash;
    }
  }
  else {
    # sort by key (lexigraphical order)
    if ($opt_r) { 
      return sort { ($b <=> $a || lc($b) cmp lc($a)) } keys %$rhash;
    } else {
      return sort { ($a <=> $b || lc($a) cmp lc($b)) } keys %$rhash;
    }
  }
}

=head1 EXAMPLES

=over 3

=item Group error messages from /var/log/messages. 

 grep 'error' /var/log/messages | histogram

=item Use histogram's built-in awk filter to histogram the hours of those messages:

 grep 'error' /var/log/messages | histogram '{ print substr($3,1,2) "h"; }'  

=item The above command, where the graph is sorted by decreasing frequency, not the hour.

 grep 'error' /var/log/messages | histogram -q -r '{ print substr($3,1,2) "h"; }'  

=item Just like the first command, now grouped by hostname; sort & truncate hostnames to 10 characters.

 histogram -q -k 10 '/error/ { print $4 }' /var/log/messages  

=item Same as the previous example, but include a graph representing the total items found:

 histogram -q -k 10 -T '/error/ { print $4 }' /var/log/messages  

=item Same as before, but don't truncate or align the keys.

 histogram -q -T -x '/error/ { print $4 }' /var/log/messages 

=back



=head1 VERSION

Version 1.2

=head1 AUTHOR & COPYRIGHT

Otheus <otheus@gmail.com>

=head1 LICENSE

Licensed under GNU Public License (GPL) 2.0 or greater.

=cut

## CHANGELOG
## 0.1 initial version
##
## 0.2 force minimum of 1 dash; right-flush keys to column
##     Try sorting with int, then alnum
##
## 1.0 add option handling for sorting, threshold, display parameters.
##
## 1.1 other fixes, better awk option handling
##
## 1.2 expanded help
