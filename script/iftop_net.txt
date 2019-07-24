#!/bin/bash


INTERFACE=ens34
PORT=(22 443 80)
TOTAL_FLOW_LOG_DIR=/var/log/iftop
TOTAL_FLOW_LOG_FILE=$TOTAL_FLOW_LOG_DIR/iftop_total.log
INPUT_FLOW_LOG_FILE=$TOTAL_FLOW_LOG_DIR/iftop_input.log
OUTPUT_FLOW_LOG_FILE=$TOTAL_FLOW_LOG_DIR/iftop_output.log
IFTOP_PORT_OPTION=''

CHECK_PACKAGE() {
    PACKAGES=(bc iftop)
    for PACKAGE in ${PACKAGES[@]} ; do
        which ${PACKAGE} &> /dev/null
	if [[ $? != 0 ]] ; then
	     echo "No have package: ${PACKAGE}, please install!"
             exit 1
	fi
    done
}


PORT_OPTION () {
TMP=1
for i in ${PORT[@]}; do

  if [[ "${#PORT[@]}" == "${TMP}" ]]; then
    IFTOP_PORT_OPTION="${IFTOP_PORT_OPTION} port ${i}"
  else
    IFTOP_PORT_OPTION="${IFTOP_PORT_OPTION} port ${i} ||"
  fi

  let TMP++

done
}

CHANGE_BIT (){
    if [[ $1 == *K* ]]
    then
    	TRUE_FLOW=`echo $1|awk -FK '{print $1}'`
    	CHANGE_FLOW=`echo "$TRUE_FLOW*1000"|bc`
    elif [[ $1 == *M* ]]
    then
    	TRUE_FLOW=`echo $1|awk -FM '{print $1}'`
    	CHANGE_FLOW=`echo "$TRUE_FLOW*1000*1000"|bc`
    elif [[ $1 == *G* ]]
    then
    	TRUE_FLOW=`echo $1|awk -FG '{print $1}'`
    	CHANGE_FLOW=`echo "$TRUE_FLOW*1000*1000*1000"|bc`
    elif [[ $1 == *T* ]]
    then
    	TRUE_FLOW=`echo $1|awk -FT '{print $1}'`
    	CHANGE_FLOW=`echo "$TRUE_FLOW*1000*1000*1000*1000"|bc`
    elif echo "$1"|[ -n "`sed -n '/^[0-9][0-9]*$/p'`" ]
    then
    	CHANGE_FLOW=$1
    else
    	CHANGE_FLOW=$1
    fi
}


FILTER_FLOW () {
    [ -d $TOTAL_FLOW_LOG_DIR ] || mkdir -p $TOTAL_FLOW_LOG_DIR

    FLOW_TOTAL=$(timeout --signal=9 100 iftop  -t -s 1 -L 1 -i $INTERFACE \
	-f "${IFTOP_PORT_OPTION}" 2> /dev/null | \
	grep -i "$1" | \
	awk -F: '{print $2}' | awk '{print $1}'|awk -Fb '{print $1}')

    CHANGE_BIT $FLOW_TOTAL

    echo "$CHANGE_FLOW" > $2
}


CHECK_PACKAGE
PORT_OPTION

while true
do
	FILTER_FLOW "Total send and receive rate" $TOTAL_FLOW_LOG_FILE
	FILTER_FLOW "Total receive rate" $INPUT_FLOW_LOG_FILE
	FILTER_FLOW "Total send rate" $OUTPUT_FLOW_LOG_FILE
done

