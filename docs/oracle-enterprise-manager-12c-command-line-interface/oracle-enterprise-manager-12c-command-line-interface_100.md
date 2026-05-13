# --------------------------------------------------------------------------------------------------

emcli login -user=sysman -pass="${SysmanPassword}"
echo "Getting the exact target name for $thisSID from OMS ..."
if [ `emcli get_targets -targets="oracle_database" | grep -i ${thisSID} | wc -l` -gt 0 ]; then
   thisTARGET=`emcli get_targets -targets="oracle_database" -format="name:csv" \
              | grep -i ${thisSID} | cut -d, -f4`
   echo "\t${thisTARGET}"
else
   echo "Sorry, ${thisSID} database is not in the OEM repository"
   ExitCleanly
fi
}

