#!/bin/bash

DISK=

E_OK=0
E_WARNING=1
E_CRITICAL=2
E_UNKNOWN=3


show_help() {
	echo "$0 -d DEVICE | -h"
	echo
	echo "This plugin is used to show disk IO and IOPS."
	echo
	echo "Example: "
	echo "  -d DEVICE     DEVICE must be without /dev (ex: -d sda)"
	echo
	echo "If no disk device is specified then plugin checks all active disks in the system"
}

# process args
while [ ! -z "$1" ]; do 
	case $1 in
		-d)	shift; DISK=$1 ;;
		-h)	show_help; exit 1 ;;
	esac
	shift
done

mkdir -p /tmp/diskstat

if [ -z "$DISK" ]; then 

	for DEVICE in `ls /sys/block`; do
   		if [ -L /sys/block/$DEVICE/device ]; then
		array+=($DEVICE)
fi
done

else

OIFS=$IFS # keep original IFS (internal field separator)
IFS=',' # set delimiter to ','
DEVICE=$DISK

for x in $DEVICE
do
	array+=($x)
	IFS=$OIFS # return original IFS values
done
fi
for DEVICE in ${array[@]}; do
# generate HISTFILE filename

HISTFILE=/tmp/diskstat/check_diskstat.$DEVICE
readdiskstat() {
	if [ ! -f "/sys/block/$DEVICE/stat" ]; then
		return $E_UNKNOWN
	fi

	cat /sys/block/$DEVICE/stat
}

readhistdiskstat() {
	[ -f $HISTFILE ] && cat $HISTFILE
}



NEWDISKSTAT=$(readdiskstat $DEVICE)

if [ $? -eq $E_UNKNOWN ]; then
	echo "Cannot read disk stats, check your /sys filesystem for $DEVICE"
	exit $E_UNKNOWN
fi

if [ ! -f $HISTFILE ]; then
	for DEVICE in ${array[@]}; do
		HISTFILE=/tmp/diskstat/check_diskstat.$DEVICE # reassign variable from external cycle
		NEWDISKSTAT=$(readdiskstat $DEVICE) # reassign variable from external cycle
		echo $NEWDISKSTAT >$HISTFILE
		echo "UNKNOWN - Initial buffer creation for $DEVICE..." 
done
	exit $E_UNKNOWN

else

OLDDISKSTAT=$(readhistdiskstat)
if [ $? -ne 0 ]; then
	echo "Cannot read histfile $HISTFILE..."
	exit $E_UNKNOWN
fi
OLDDISKSTAT_TIME=$(stat -c %y $HISTFILE)
OLDDISKSTAT_EPOCH=$(date -d "$OLDDISKSTAT_TIME" +%s)
NEWDISKSTAT_EPOCH=$(date +%s)

echo $NEWDISKSTAT >$HISTFILE
# now we have old and current stat; 
# let compare it
OLD_SECTORS_READ=$(echo $OLDDISKSTAT | awk '{print $3}')
NEW_SECTORS_READ=$(echo $NEWDISKSTAT | awk '{print $3}')
OLD_READ=$(echo $OLDDISKSTAT | awk '{print $1}')
NEW_READ=$(echo $NEWDISKSTAT | awk '{print $1}')
OLD_WRITE=$(echo $OLDDISKSTAT | awk '{print $5}')
NEW_WRITE=$(echo $NEWDISKSTAT | awk '{print $5}')

OLD_SECTORS_WRITTEN=$(echo $OLDDISKSTAT | awk '{print $7}')
NEW_SECTORS_WRITTEN=$(echo $NEWDISKSTAT | awk '{print $7}')

# kernel handles sectors by 512bytes
# http://www.mjmwired.net/kernel/Documentation/block/stat.txt
SECTORBYTESIZE=512

let "SECTORS_READ = $NEW_SECTORS_READ - $OLD_SECTORS_READ"
let "SECTORS_WRITE = $NEW_SECTORS_WRITTEN - $OLD_SECTORS_WRITTEN"
let "TIME = $NEWDISKSTAT_EPOCH - $OLDDISKSTAT_EPOCH"
let "BYTES_READ_PER_SEC = $SECTORS_READ * $SECTORBYTESIZE / $TIME"
let "BYTES_WRITTEN_PER_SEC = $SECTORS_WRITE * $SECTORBYTESIZE / $TIME"
let "TPS=($NEW_READ - $OLD_READ + $NEW_WRITE - $OLD_WRITE) / $TIME"

let "KBYTES_READ_PER_SEC = $BYTES_READ_PER_SEC / 1024"
let "KBYTES_WRITTEN_PER_SEC = $BYTES_WRITTEN_PER_SEC / 1024"

OUTPUT=""
EXITCODE=$E_OK

RESULT+=$(echo "${OUTPUT} ${DEVICE} $TPS io/s, read ${DEVICE} $SECTORS_READ sectors (${KBYTES_READ_PER_SEC}kB/s), write ${DEVICE} $SECTORS_WRITE sectors (${KBYTES_WRITTEN_PER_SEC}kB/s) in $TIME seconds")

PERFDATA+=$(echo "${DEVICE}_tps=${TPS};;; ${DEVICE}_read=${BYTES_READ_PER_SEC};;; ${DEVICE}_write=${BYTES_WRITTEN_PER_SEC};;; ")
fi
done
echo "$RESULT | $PERFDATA"
exit $EXITCODE
