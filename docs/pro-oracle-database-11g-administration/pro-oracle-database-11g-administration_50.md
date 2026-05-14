# source environment variables (see Chapter 2 for details on oraset)
. /var/opt/oracle/oraset $HOLDSID

BOX=`uname -a | awk '{print$2}'`
MAILX='/bin/mailx'
MAIL_LIST='larry@oracle.com'
NLS_DATE_FORMAT='dd-mon-yy hh24:mi:ss'
date

#----------------------------------------------
LOCKFILE=/tmp/$PRG.lock

if [ -f $LOCKFILE ]; then
    echo "lock file exists, exiting..."
    exit 1
else
    echo "DO NOT REMOVE, $LOCKFILE" > $LOCKFILE
fi

#----------------------------------------------
rman nocatalog <<EOF
connect target /
set echo on;
show all;

