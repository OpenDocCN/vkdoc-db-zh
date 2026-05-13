# ----------------------------------------------------
function CopyFiles {
cd $SOURCE_DIR

ls -l | grep -i $ORACLE_SID      >$WORKFILE

for thisFILE in `cat $WORKFILE`; do
   SOURCE_FILE=$SOURCE_DIR/$thisFILE
   TARGET_FILE=$TARGET_DIR/$thisFILE
   cp -f $SOURCE_FILE $TARGET_FILE
done

rm -f $WORKFILE
}

