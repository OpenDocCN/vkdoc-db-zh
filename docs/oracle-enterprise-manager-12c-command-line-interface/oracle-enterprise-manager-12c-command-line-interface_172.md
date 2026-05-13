# --------------------------------------------------------------------------------------------------
if [ ${#1} -eq 0 ]; then
   echo "您必须在命令行传入 VCS 集群名称\n\n"
   exit 1
fi

export VCSBIN=< 设置为您的 VRTSvcs 可执行文件的 bin 目录 >

if [ ! -d ${VCSBIN} ]; then
   echo  "在 ${VCSBIN} 中找不到 Veritas 可执行文件"
   export VCSBIN=""
   exit 1
fi

