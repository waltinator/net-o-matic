#!/bin/bash
# Monitor for the net going down, Do The Next Thing (from a config file)
# to bring the net up. Implicit assumpition that The Next Thing fixes it
# may be a problem.
# Walt Sullivan

# Note: places where you may need to adjust things to match 
# your environment/taste are marked "#Adjust"

# determine my name
me=$0
me=${me##*/}

# my variables
debug=0				# set via --debug
verbose=0			# set via --verbose
original=""			# set the <config.file>
original_update=0		# time <config.file> was last modified
config="/var/tmp/${me}.$$.config" # my writable copy of config
result=""			  # temporary use

# $ dpkg -S $(type -p nm-online)
# network-manager: /usr/bin/nm-online
# $ dpkg -S $(type -p ip)
# iproute2: /sbin/ip
# man ip-link;man ip-monitor;man ip-address;man 7 regex

# -h or --help or something's wrong in here
help () {
    echo "${me} [-h|--help] [-v|--verbose] <config.file> " >&2
    echo "" >&2
    echo "Monitor the wireless network, and when it goes down, Do The" >&2
    echo "Next Thing (as specified by the <config.file>), to bring" >&2
    echo "the wireless net up." >&2
    echo "" >&2
    echo "The <config.file> contains #comments, blank lines, AND" >&2
    echo "single line commands, of your choice, to correct the" >&2
    echo "wireless network down condition. The first command in the" >&2
    echo "<config.file> will be executed the first time the net goes" >&2
    echo "down (or if the net is down when ${me} begins), the second" >&2
    echo "command will be executed the next time the net goes down," >&2
    echo "and so forth, wrapping around at the end. The number of" >&2
    echo "single line commands in the <config.file> is unlimited." >&2
    exit 2
}

function flip () {
    # Return the first non-blank, non #comment line,
    # and move that line (and all preceeding blank and #comment
    # lines) to the end of (our copy of) the config file.
    #
    # ed pattern includes "/", but not "~" or "."
    #Adjust the ed pattern in both places and in countconfiglines
    ed --quiet "$config" <<EndOfEd
/^[A-Za-z0-9\/]/
1,.t$
1,/^[A-Za-z0-9\/]/d
wq
EndOfEd
}

function countconfiglines () {
    # return number of config file lines
    #Adjust must match the ed pattern in flip()
    configfile="$1"
    egrep --count '^[[:alnum:]/]' "$configfile"
    }

function up-to-date () {
    # updates configuration file if necessary
    new_update="$(/usr/bin/stat --format='%Y' $original )"
    if [[  "$new_update" -ne "$original_update" ]] ; then
	if [[ $(countconfiglines "$original") -eq 0  ]] ; then
	    echo "Invalid configuration in $original" >&2
	    exit 4
	else
	    /bin/cp --force "$original" "$config"
	    original_update="$new_update"
	fi
    fi
    }

function netstate () {
    # Return network state as "UP" or "DOWN"
    #Adjust how you decide net is UP/DOWN
    ip link show | egrep -q 'UP,LOWER_UP.* state UP'
    if [[ $? -eq 0 ]] ; then
	echo "UP"
    else
	echo "DOWN"
    fi
}

# Execution begins here

# parse the args with getopt, adapted from
# /usr/share/doc/util-linux/examples/getopt-parse.bash

TEMP=`getopt -o dhv --long debug,help,verbose \
     -n "${me}" -- "$@"`

if [[ $? != 0 ]] ; then echo "${me} --help for help." >&2 ; exit 1 ; fi

# Note the quotes around `$TEMP': they are essential!
eval set -- "$TEMP"

while true ; do
    case "$1" in
	-d|--debug) debug=1; shift;;
	-h|--help) help; shift;;
	-v|--verbose) verbose=1; shift;;
	--) shift; break;;
	*) echo "Internal error! ${me} --help for help";exit 1;;
    esac
done

# Did we get the <config.file>?
original="$1"
shift
[[ -z "$original" ]] && \
    (echo "Missing config file ${me} -h for help" >&2 ; exit 1)

# If there are more parameters, confusion exists in the mind of the caller
[[ "$#" -ne 0 ]] && help

[[ -r "$original" ]] || (echo "${me}:Cannot read $original" >&2
    			exit 2)

if [[ $(countconfiglines "$original" ) -eq 0 ]] ; then
    echo "${me}:Invalid configuration in $original" >&2
    exit 4
else
    # watch for changes, record "%Y time of last modification,
    # seconds since Epoch"
    original_update="$(/usr/bin/stat --format='%Y' $original )"

    # make a writeable copy for our use, and clean it up at the end
    # unless $debug
    [[ $debug -ne 0 ]] || trap "/bin/rm -f $config" EXIT
    /bin/cp --force "$original" "$config"
fi

# if the net is down, Do The Next Thing right away
#Adjust: how you decide the net is up or down?
[[ "$(netstate)" = "DOWN" ]] && \
    ( result="$(flip)";
    [[ $verbose ]] && echo "$(date):$result" >&2; \
    	eval "$result" )

# Wait for the net to go down, then Do The Next Thing
#Adjust find a better way to watch for net down 
ip monitor address | \
    egrep --line-buffered \
    '^Deleted [[:digit:]]+: [[:alnum:]]+[[:space:]]+inet[[:space:]].* scope global ' | \
    while read line ; do
    up-to-date
    result="$(flip)"
    [[ $verbose -eq 1 ]] && echo "$(date):$result" >&2
# Here is where "The Next Thing" is Done
    eval "$result"
done
# we never exit the while loop, until the world ends.
exit 0
