# 使用块更改跟踪

`BACKUP INCREMENTAL` 命令会创建一个增量级别 1 的备份，并为其分配指定的标签名称；此备份随后将由 `RECOVER COPY` 命令使用。

此后，每次运行这两行代码时，你都会得到一个定期重复的备份模式。如果你使用映像副本进行备份，可以考虑采用增量更新备份策略，因为使用该策略可以避免在每次备份运行时都创建完整的映像副本。映像副本会在每次备份运行时，根据前一次备份以来的增量更改进行更新。

## 启用块更改跟踪

块更改跟踪是利用一个二进制文件来记录数据库数据文件块更改的过程。其原理在于可以提升增量备份的性能，因为 RMAN 能够利用块更改跟踪文件来准确定位自上次备份以来哪些块发生了更改。这节省了大量时间，否则 RMAN 将不得不扫描所有已备份的块以确定它们是否自上次备份以来发生了变化。

以下是启用块更改跟踪的步骤：

1.  如果尚未启用，请将 `DB_CREATE_FILE_DEST` 参数设置为磁盘上已存在的一个位置；例如，
    ```
    SQL> alter system set db_create_file_dest='/u01/O18C/bct' scope=both;
    ```

2.  通过 `ALTER DATABASE` 命令启用块更改跟踪：
    ```
    SQL> alter database enable block change tracking;
    ```

此示例在 `DB_CREATE_FILE_DEST` 指定的目录中创建了一个使用 OMF 名称的文件。在此示例中，创建的文件被赋予以下名称：
```
/u01/O18C/bct/O18C/changetracking/o1_mf_8h0wmng1_.chg
```

你也可以通过直接指定文件名来启用块更改跟踪，这不需要设置 `DB_CREATE_FILE_DEST`；例如，
```
SQL> alter database enable block change tracking using file '/u01/O18C/bct/btc.bt';
```

你可以通过运行以下查询来验证块更改跟踪的详细信息：
```
SQL> select * from v$block_change_tracking;
```

出于空间规划目的，块更改跟踪文件的大小约为数据库中被跟踪块总大小的 1/30,000。因此，块更改跟踪文件的大小与数据库的大小成正比，而与生成的重做日志量无关。

要禁用块更改跟踪，请运行此命令：
```
SQL> alter database disable block change tracking;
```

**注意**
当你禁用块更改跟踪时，Oracle 会自动删除块更改跟踪文件。

