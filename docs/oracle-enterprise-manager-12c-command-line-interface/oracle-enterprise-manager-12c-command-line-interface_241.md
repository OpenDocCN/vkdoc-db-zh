# --------------------------------------------------------------------------------------------------
if [ -d ${thisDIR} ]; then
   if [ `find ${thisDIR} -name ${thisFILE} | wc -l` -gt 0 ]; then
      NEW_FILE=${thisDIR}/${thisFILE}
      OLD_FILE=${thisDIR}/${thisFILE}_old
      LAST_DATE=`tail -1 ${NEW_FILE} | awk '{ print $1,$2,$3 }'`
      echo "\n 正在截断 ${NEW_FILE}"
      echo "早于 ${LAST_DATE} 的条目将被移除\n"
      mv -f ${NEW_FILE} ${OLD_FILE}
      cat ${OLD_FILE} | grep "${LAST_DATE}" >${NEW_FILE}
         if [ `cat ${NEW_FILE} | wc -l` -gt 0 ]; then
            rm -f ${OLD_FILE}
         else
            mv -f ${OLD_FILE} ${NEW_FILE}
      fi
   fi
   echo "\n\n"
else
   echo "\n 未找到目录 ${thisDIR}\n\n"
fi
}

function SweepThese {
