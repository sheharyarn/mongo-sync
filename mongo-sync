#!/bin/bash

set -e           # exit on error
set -o pipefail  # trace ERR through pipes
set -o errtrace  # trace ERR through 'time command' and other functions
set -o errexit   # exit the script if any statement returns a non-true return value

function cleanup {
    echo "Cleaning up..."
    if [ -z ${TMPDIR+x} ] ; then
        echo -n
    else
        rm -rf $TMPDIR
        unset TMPDIR
    fi
    unset TUNNEL_PORT
    unset TUNNEL_CREDENTIALS
    unset LOCAL_CREDENTIALS
    unset REMOTE_CREDENTIALS
}

function error {
    local parent_lineno="$1"
    local message="$2"
    local code="${3:-1}"
    if [[ -n "$message" ]] ; then
        echo "Error on or near line ${parent_lineno}: ${message}; exiting with status ${code}"
    else
        echo "Error on or near line ${parent_lineno}; exiting with status ${code}"
    fi
    cleanup
    exit "${code}"
}
trap 'error ${LINENO}' ERR

function usage_error {
    echo mongo-sync: "$1"
    echo "usage: mongo-sync push|pull"
    echo ""
    exit
}

function config_not_found {
    echo "failed: '$1' not found, it needs to be in the same dir"
    echo "aborting..."
    echo ""
    exit
}

function parse_yaml {
   local prefix=$2
   local s='[[:space:]]*' w='[a-zA-Z0-9_]*' fs=$(echo @|tr @ '\034')
   sed -ne "s|^\($s\):|\1|" \
        -e "s|^\($s\)\($w\)$s:$s[\"']\(.*\)[\"']$s\$|\1$fs\2$fs\3|p" \
        -e "s|^\($s\)\($w\)$s:$s\(.*\)$s\$|\1$fs\2$fs\3|p"  "$1" |
   awk -F$fs '{
      indent = length($1)/2;
      vname[indent] = $2;
      for (i in vname) {if (i > indent) {delete vname[i]}}
      if (length($3) > 0) {
         vn=""; for (i=0; i<indent; i++) {vn=(vn)(vname[i])("_")}
         printf("%s%s%s=\"%s\"\n", "'$prefix'",vn, $2, $3);
      }
   }'
}

function get_script_dir {
    pushd . > /dev/null
    local SCRIPT_PATH="${BASH_SOURCE[0]}"

    if ([ -h "${SCRIPT_PATH}" ]) then
      while ([ -h "${SCRIPT_PATH}" ]) do cd `dirname "$SCRIPT_PATH"`; SCRIPT_PATH=`readlink "${SCRIPT_PATH}"`; done
    fi

    cd `dirname ${SCRIPT_PATH}` > /dev/null
    local SCRIPT_PATH=`pwd`;
    popd  > /dev/null

    echo $SCRIPT_PATH
}

function get_confirmation() {
    read -p "Are you sure you want to $1? Enter 'yes': " mongo_confr

    case $mongo_confr in
        [yY][Ee][Ss] )  ;;
        *) echo "Incorrect input, aborting..."; exit;;
    esac
}

function load_configs {
    echo "load_configs"
    if [ -z "$CONFIG_FILE" ] ; then
        CONFIG_FILE="config.yml"
    fi
    local config=$CONFIG_FILE
    echo "Parsing '$config'..."

    DIR=$(get_script_dir)
    local FILE="$DIR/$config"

    if [ -f "$FILE" ]; then
       eval $(parse_yaml "$FILE")
       success_msg
    else
       config_not_found $config
    fi

    # Loads:
    # - $local_db
    # - $local_host_port
    # - $local_access_username
    # - $local_access_password
    # - $remote_db
    # - $remote_host_url
    # - $remote_host_port
    # - $remote_access_username
    # - $remote_access_password
    # - $tunnel_on
    # - $tunnel_access_username
    # - $tunnel_access_port

    LOCAL_CREDENTIALS=""
    if [[ ! -z $local_access_username ]] ; then
        LOCAL_CREDENTIALS="-u $local_access_username -p $local_access_password"
    fi

    REMOTE_CREDENTIALS=""
    if [[ ! -z $remote_access_username ]] ; then
        REMOTE_CREDENTIALS="-u $remote_access_username -p $remote_access_password"
    fi

    TUNNEL_CREDENTIALS="$tunnel_access_username@$remote_host_url"
    TUNNEL_PORT=27018

    TMPDIR=/tmp/"$local_db"/dump
}

function banner {
    echo mongo-sync:
    echo -----------
}

function success_msg {
    echo "Success!"
    echo
}

function done_msg {
    echo "Done!"
    echo
}

function open_tunnel {
    echo "Connecting through SSH tunnel..."
    ssh -fqTNM -S db-sync-socket -L $TUNNEL_PORT:127.0.0.1:$remote_host_port $TUNNEL_CREDENTIALS -p $tunnel_access_port
}

function close_tunnel {
    echo "Disconnecting from SSH tunnel..."
    ssh -S db-sync-socket -O exit $TUNNEL_CREDENTIALS 2> /dev/null
}


function pull {
    banner
    if [ "$1" == false ] ; then
        get_confirmation 'pull'
    fi
    load_configs

    if [ "$tunnel_on" == true ] ; then
        open_tunnel
        remote_host_url="localhost"
        remote_host_port=$TUNNEL_PORT
    fi

    echo "Dumping Remote DB to $TMPDIR... "
    mongodump \
        -h "$remote_host_url":"$remote_host_port" \
        -d "$remote_db" \
        $REMOTE_CREDENTIALS \
        -o "$TMPDIR" > /dev/null
    success_msg

    echo "Overwriting Local DB with Dump... "
    mongorestore \
        --port "$local_host_port" \
        -d "$local_db" \
        $LOCAL_CREDENTIALS \
        "$TMPDIR"/"$remote_db" \
        --drop > /dev/null
    success_msg

    if [ "$tunnel_on" == true ] ; then
        close_tunnel
    fi

    cleanup
    done_msg
}

function push {
    banner
    if [ "$1" == false ] ; then
        get_confirmation 'push'
    fi
    load_configs

    if [ "$tunnel_on" == true ] ; then
        open_tunnel
        remote_host_url="localhost"
        remote_host_port=$TUNNEL_PORT
    fi

    echo "Dumping Local DB to $TMPDIR... "
    mongodump \
        --port "$local_host_port" \
        -d "$local_db" \
        $LOCAL_CREDENTIALS \
        -o "$TMPDIR" > /dev/null
    success_msg

    echo "Overwriting Remote DB with Dump... "
    mongorestore \
        -h "$remote_host_url":"$remote_host_port" \
        -d "$remote_db" \
        $REMOTE_CREDENTIALS \
        "$TMPDIR"/"$local_db" --drop > /dev/null
    success_msg

    if [ "$tunnel_on" == true ] ; then
        close_tunnel
    fi

    cleanup
    done_msg
}


## MAIN
## ====

# TODO: Add --overwrite flag

if [[ $# -eq 0 ]] ; then
    usage_error "no arguments provided"
else
    action="$1"
    skip=false
    shift

    while [ "${1+defined}" ]; do
        if [ "$1" == '--config' ] ; then
            shift
            if [ -e "$1" ] ; then
                CONFIG_FILE="$1"
            else
                usage_error "No config file named \"$1\""
            fi
        elif [ "$1" == '-y' ] ; then
            skip=true
        else
            usage_error "unrecognized option \"$1\""
        fi
        shift
    done

    if [[ "$action" == 'push' ]] ; then
        push $skip
    elif [[ "$action" == 'pull' ]] ; then
        pull $skip
    else
        usage_error "unrecognized command \"$action\""
    fi
fi
