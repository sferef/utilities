# utilities



wait-for-it

#!/usr/bin/env bash
#   Use this script to test if a given TCP host/port are available

cmdname=$(basename $0)

echoerr() { if [[ $QUIET -ne 1 ]]; then echo "$@" 1>&2; fi }

usage()
{
    cat << USAGE >&2
Usage:
    $cmdname host:port [-s] [-t timeout] [-- command args]
    -h HOST | --host=HOST       Host or IP under test
    -p PORT | --port=PORT       TCP port under test
                                Alternatively, you specify the host and port as host:port
    -s | --strict               Only execute subcommand if the test succeeds
    -q | --quiet                Don't output any status messages
    -t TIMEOUT | --timeout=TIMEOUT
                                Timeout in seconds, zero for no timeout
    -- COMMAND ARGS             Execute command with args after the test finishes
USAGE
    exit 1
}

wait_for()
{
    if [[ $TIMEOUT -gt 0 ]]; then
        echoerr "$cmdname: waiting $TIMEOUT seconds for $HOST:$PORT"
    else
        echoerr "$cmdname: waiting for $HOST:$PORT without a timeout"
    fi
    start_ts=$(date +%s)
    while :
    do
        (echo > /dev/tcp/$HOST/$PORT) >/dev/null 2>&1
        result=$?
        if [[ $result -eq 0 ]]; then
            end_ts=$(date +%s)
            echoerr "$cmdname: $HOST:$PORT is available after $((end_ts - start_ts)) seconds"
            break
        fi
        sleep 1
    done
    return $result
}

wait_for_wrapper()
{
    # In order to support SIGINT during timeout: http://unix.stackexchange.com/a/57692
    if [[ $QUIET -eq 1 ]]; then
        timeout $TIMEOUT $0 --quiet --child --host=$HOST --port=$PORT --timeout=$TIMEOUT &
    else
        timeout $TIMEOUT $0 --child --host=$HOST --port=$PORT --timeout=$TIMEOUT &
    fi
    PID=$!
    trap "kill -INT -$PID" INT
    wait $PID
    RESULT=$?
    if [[ $RESULT -ne 0 ]]; then
        echoerr "$cmdname: timeout occurred after waiting $TIMEOUT seconds for $HOST:$PORT"
    fi
    return $RESULT
}

# process arguments
while [[ $# -gt 0 ]]
do
    case "$1" in
        *:* )
        hostport=(${1//:/ })
        HOST=${hostport[0]}
        PORT=${hostport[1]}
        shift 1
        ;;
        --child)
        CHILD=1
        shift 1
        ;;
        -q | --quiet)
        QUIET=1
        shift 1
        ;;
        -s | --strict)
        STRICT=1
        shift 1
        ;;
        -h)
        HOST="$2"
        if [[ $HOST == "" ]]; then break; fi
        shift 2
        ;;
        --host=*)
        HOST="${1#*=}"
        shift 1
        ;;
        -p)
        PORT="$2"
        if [[ $PORT == "" ]]; then break; fi
        shift 2
        ;;
        --port=*)
        PORT="${1#*=}"
        shift 1
        ;;
        -t)
        TIMEOUT="$2"
        if [[ $TIMEOUT == "" ]]; then break; fi
        shift 2
        ;;
        --timeout=*)
        TIMEOUT="${1#*=}"
        shift 1
        ;;
        --)
        shift
        CLI="$@"
        break
        ;;
        --help)
        usage
        ;;
        *)
        echoerr "Unknown argument: $1"
        usage
        ;;
    esac
done

if [[ "$HOST" == "" || "$PORT" == "" ]]; then
    echoerr "Error: you need to provide a host and port to test."
    usage
fi

TIMEOUT=${TIMEOUT:-15}
STRICT=${STRICT:-0}
CHILD=${CHILD:-0}
QUIET=${QUIET:-0}

if [[ $CHILD -gt 0 ]]; then
    wait_for
    RESULT=$?
    exit $RESULT
else
    if [[ $TIMEOUT -gt 0 ]]; then
        wait_for_wrapper
        RESULT=$?
    else
        wait_for
        RESULT=$?
    fi
fi

if [[ $CLI != "" ]]; then
    if [[ $RESULT -ne 0 && $STRICT -eq 1 ]]; then
        echoerr "$cmdname: strict mode, refusing to execute subprocess"
        exit $RESULT
    fi
    exec $CLI
else
    exit $RESULT
fi


common base

#install from ubuntu
FROM ubuntu:14.04

# wait for it script
ADD common/wait-for-it.sh /opt/sbd/scripts/wait-for-it.sh
RUN chmod 700 /opt/sbd/scripts/wait-for-it.sh

# Setting debconf to non interactive
RUN echo 'debconf debconf/frontend select Noninteractive' | debconf-set-selections
RUN echo "deb http://security.ubuntu.com/ubuntu trusty-security main" >>  /etc/apt/sources.list

# Install some software
RUN sudo apt-get -y update
RUN sudo apt-get -yq install software-properties-common


# Install java 8
RUN sudo add-apt-repository -y ppa:webupd8team/java
RUN echo "oracle-java8-installer shared/accepted-oracle-license-v1-1 select true" | sudo debconf-set-selections
RUN sudo apt-get -y update
RUN sudo apt-get install -yq --force-yes oracle-java8-installer



# Install and configure SSH server (needed by hadoop)
RUN sudo apt-get -y install openssh-server
RUN echo "root:sbd" | sudo chpasswd
RUN sed -i "/PermitRootLogin without-password/s/^/#/" /etc/ssh/sshd_config
RUN echo "PermitRootLogin yes" >>  /etc/ssh/sshd_config
RUN ssh-keygen -t dsa -P '' -f ~/.ssh/id_dsa
RUN cat ~/.ssh/id_dsa.pub >> ~/.ssh/authorized_keys
RUN chmod 0600 ~/.ssh/authorized_keys
RUN echo "    StrictHostKeyChecking no" >> /etc/ssh/ssh_config

# curl
RUN apt-get -y install curl

# SET JAVA_HOME (needed to run hadoop, among others)
ENV JAVA_HOME=/usr/lib/jvm/java-8-oracle/

#Cleansing
RUN apt-get clean
RUN rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*



hadoop

FROM sbX-XXX-base

ENV HADOOP_VERSION=2.7.5

RUN mkdir /opt/sds

# Descarga e instala Hadoop en /opt/sds/hadoop
RUN wget http://ftp.cixug.es/apache/hadoop/common/hadoop-${HADOOP_VERSION}/hadoop-${HADOOP_VERSION}.tar.gz -O /opt/hadoop-${HADOOP_VERSION}.tar.gz
RUN cd /opt/ && tar xf hadoop-${HADOOP_VERSION}.tar.gz
RUN ln -s /opt/hadoop-${HADOOP_VERSION} /opt/sds/hadoop

ENV HADOOP_HOME /opt/sds/hadoop

# HDFS configura en todos los nodos con HDFS
RUN mv /opt/sds/hadoop/etc/hadoop/core-site.xml /opt/sds/hadoop/etc/hadoop/core-site.xml.bak
RUN echo "<configuration><property><name>fs.defaultFS</name><value>hdfs://hadoop:8020</value></property></configuration>" > /opt/sds/hadoop/etc/hadoop/core-site.xml
RUN mv /opt/sds/hadoop/etc/hadoop/hdfs-site.xml /opt/sds/hadoop/etc/hadoop/hdfs-site.xml.bak
RUN echo "<configuration><property><name>dfs.replication</name><value>1</value></property><property><name>dfs.permissions</name><value>false</value></property></configuration>" > /opt/sds/hadoop/etc/hadoop/hdfs-site.xml

# Incluye variable de entorno en HDFS y comandos bash.
ENV HADOOP_CONF_DIR /opt/sds/hadoop/etc/hadoop

ENV PATH $PATH:$HADOOP_HOME/bin

# SET Java environment (needed to run hadoop)
RUN sed -i '/export JAVA_HOME=\${JAVA_HOME}/c export JAVA_HOME=/usr/lib/jvm/java-8-oracle' ${HADOOP_CONF_DIR}/hadoop-env.sh

# Formateo del namenode
RUN /opt/sds/hadoop/bin/hdfs namenode -format

USER root

# Bootstrap script
ADD bootstrap.sh /etc/bootstrap.sh
RUN chown root:root /etc/bootstrap.sh
RUN chmod 700 /etc/bootstrap.sh

ENV BOOTSTRAP /etc/bootstrap.sh

ENTRYPOINT ["/etc/bootstrap.sh"]



bootstrap 

#!/bin/bash
service ssh restart &
#start hdfs
/opt/sds/hadoop/sbin/start-dfs.sh

# an endless process to avoid container to stop
tail -f /dev/null

