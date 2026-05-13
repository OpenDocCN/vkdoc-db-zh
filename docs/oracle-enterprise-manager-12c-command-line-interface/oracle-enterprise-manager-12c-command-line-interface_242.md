# --------------------------------------------------------------------------------------------------
if [ -d ${thisDIR} ]; then
   find ${thisDIR} -name ${thisFILE} -mtime +${thisAGE} >${WORKFILE}
   if [ `cat ${WORKFILE} | wc -l` -gt 0 ]; then
      echo "\n 清理 ${thisDIR} 目录"
      for thisTRASH in `cat ${WORKFILE}`; do
         echo"\t 移除 ${thisTRASH}"
         rm -f ${thisTRASH}
      done
   else
      echo "\n 没有匹配 ${thisDIR}/${thisFILE} 的"
      echo "超过 ${thisAGE} 天的文件"
   fi
   echo "\n\n"
else
   echo "\n 未找到目录 ${thisDIR}\n\n"
fi
}

