# --------------------------------------------------------------------------------------------------

thisDIR=${EM_INSTANCE_HOME}/${WEBTIERn}/diagnostics/logs/OHS/${OHSn}
   thisAGE=7

thisFILE="em_upload_http_access_log.*"
   SweepThese

thisFILE="em_upload_https_access_log.*"
   SweepThese

thisFILE="access_log.*"
   SweepThese

thisFILE="ohs*.log"
   SweepThese

thisFILE="console~OHS*.log*"
   SweepThese

thisFILE="mod_wl_ohs.log"
   TrimThisFile

