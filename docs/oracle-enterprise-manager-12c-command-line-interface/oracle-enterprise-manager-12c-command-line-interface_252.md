# --------------------------------------------------------------------------------------------------

DOMAIN_SERVERS_DIR=${EM_INSTANCE_HOME}/user_projects/domains/GCDomain/servers

thisDIR=${DOMAIN_SERVERS_DIR}/EMGC_ADMINSERVER/logs
   thisAGE=7

thisFILE="GCDomain.log*"
   SweepThese

thisFILE="EMGC_ADMINSERVER.out*"
   SweepThese

thisFILE="EMGC_ADMINSERVER.log*"
   SweepThese

thisFILE="EMGC_ADMINSERVER-diagnostic-*.log"
   SweepThese

thisFILE="access.log*"
   SweepThese

