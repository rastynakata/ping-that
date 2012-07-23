ping-that
=========
#!/bin/bash
# - NAME:
#    myping - Ouput pinged summaries for several hosts
#             at once. Traceoute will be ran if there
#             are non-responding hosts.
#
# - SYNOPSIS:
#    myping [-w] npacket host [ host2  host3 ... ]
#
# - DESCRIPTION:
#
# - OPTION(S):
#    -w    Traceoute will not be ran for non-responding
#          hosts. A warning message will be printed if
#          there are one or more non-responding hosts.
#
# - ASSUMPTION(S):
#    myping will work on both HP-UX and RH Linux.
#
# Author: Wing Hei Lai
# Date of creation: 03/04/2011
export PATH=/bin:/usr/bin:/usr/sbin:/usr/contrib/bin

tmpfile1=$(mktemp /tmp/myping.out.XXXX)
tmpfile2=$(mktemp /tmp/myping.lst.XXXX)
##################################################
# Remove temp files when it exits for any reason #
##################################################
trap 'rm -f $tmpfile1 $tmpfile2' EXIT
####################
# Signals handling #
####################
trap '' HUP
trap 'echo terminated >&2' TERM
trap 'echo interrupted >&2' INT

####################################################
# error(): output an error message and return      #
#          the error messages contain the program  #
#          name and are written to standard error. #
####################################################
error() {
    typeset progname=$(basename $0)
    echo "$progname: ERROR: $*" >&2
    echo "$progname: USAGE: $progname [-w] npacket host [ host2  host3 ... ]" >&2
    exit 1
}

################################################
# Check if it's being ran on hills or RH Linux #
################################################
if [ "$(uname)" = 'Linux' ]; then
    arg="c"
    plat="linux"
    nttl="25"
    nquery="1"
elif [ "$(uname)" = 'HP-UX' ]; then
    arg="n"
    nttl="15"
    nquery="2"
fi

##################################################
# [-w] flag to control whether to run traceroute #
##################################################
if [ "$1" = '-w' ]; then
    flag=true
    shift
fi

##########################################
# Check 1st arg. type and number of arg. #
##########################################
if ! echo "$1" | grep -Eq '^[[:digit:]]+$' ; then
    error  "npacket must be an integer - aborted"
elif [ $# -lt 2 ]; then
    error "At least 2 arguments without [-w] - aborted" >&2
fi

######################################################
# Process arguments and append output to a temp file #
######################################################
npacket=$1
line="$(echo $* | tr -s " " | cut -d" " -f2-)"
for site in $line; do
    ping $site -"${arg}" $npacket 1> $tmpfile1 2> $tmpfile1
    # Output format
format='%s %-20s %-50s %s \n'
    if grep -q '100% packet loss' $tmpfile1; then
        info="$(grep '100% packet loss' $tmpfile1 | cut -d"," -f1-3)"
        printf "${format}" "" "$site" "$info"
    elif grep -q 'unknown host' $tmpfile1; then
        info="$(cat $tmpfile1 | tr -s " " | cut -d":" -f2)"
        printf "${format}" "" "$site" "$info"
    else
        info="$(cat $tmpfile1 | tail -n 2 | head -n 1 | cut -d"," -f1-3)"
        if [ "$plat" = linux ]; then
            roundtime="$(cat $tmpfile1 | tail -n 1 | tr -s " " | cut -d"=" -f2 |\
                cut -d" " -f2 | cut -d"/" -f1-3)"
            avg="$(cat $tmpfile1 | tail -n 1 | tr -s " " | cut -d"=" -f2 |\
                cut -d" " -f2 | cut -d"/" -f1-3 | cut -d"/" -f2)"
        else
            roundtime="$(cat $tmpfile1 | tail -n 1 | tr -s " " | cut -d" " -f5)"
            avg="$(cat $tmpfile1 | tail -n 1 | tr -s " " | cut -d" " -f5 |\
                cut -d"/" -f2)"
        fi
        printf "${format}" "$avg" "$site" "$info" "$roundtime"
    fi
done >> $tmpfile2

#########################################################
# Sort and display ping's statistics and error messages #
#########################################################
khost="$(cat $tmpfile2 | grep -v '100% packet loss' | grep -v 'unknown host')"
if ! [ -z "$khost" ]; then
    echo "Known Host(s):"
    perl -e 'print "#" x 90, "\n"'
    grep -v '100% packet loss' $tmpfile2 | grep -v 'unknown host' |\
        sort -t" " -k1n | cut -d" " -f2-
    echo ""
fi

if grep -q 'unknown host' $tmpfile2; then
    echo "Unknown Host(s):"
    perl -e 'print "#" x 90, "\n"'
    grep 'unknown host' $tmpfile2 | cut -c 2-
    echo ""
fi

if grep -q '100% packet loss' $tmpfile2; then
    echo "Non-responding Host(s):"
    perl -e 'print "#" x 90, "\n"'
    grep '100% packet loss' $tmpfile2 | cut -c 2-
    echo ""
fi

############################
# Run traceroute if needed #
############################
if [ "$flag" = true ]; then
    if grep -q '100% packet loss' $tmpfile2; then
        echo "*************WARNING: Non-responding host(s) found!*************"
        echo "Please run traceroute to locate where the network congestion is."
        exit 0
    else
        exit 0
    fi
else
    if grep -q '100% packet loss' $tmpfile2; then
        echo "Traceroute to non-responding host(s):"
        perl -e 'print "#" x 90, "\n"'
        noRespHost="$(cat $tmpfile2 | grep '100% packet loss' | cut -d" " -f2)"
        for badSite in $noRespHost; do
            traceroute -w 2 -m "${nttl}" -q "${nquery}" $badSite
            echo ""
        done
        exit 0
    fi
fi
