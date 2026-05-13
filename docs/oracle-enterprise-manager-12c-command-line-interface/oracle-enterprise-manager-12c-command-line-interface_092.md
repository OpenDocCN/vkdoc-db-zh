# --------------------------------------------------------------------------------------------------

emcli login -user=sysman -pass="${SysmanPassword}"
echo "Getting the names of all EM agents from OMS ..."
emcli get_targets | grep oracle_emd | sort -u | awk '{ print $4 }'
}

function GetTargetName {
