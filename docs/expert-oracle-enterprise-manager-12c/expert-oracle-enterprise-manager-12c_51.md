# 3. 定义默认收集项

在默认收集元数据 XML 文件中定义特殊的默认收集项。

在 EDK 提供的三个示例中，只有 `oracle.samples.xsh3` 插件包含了完整的 ECM 实现。`oracle.samples.xsh1` 和 `oracle.samples.xsh2` 插件确实定义了配置指标，但缺少 ECM 元数据，因此不会创建存储库表，这意味着代理收集的配置信息实际上没有被加载到任何地方。忽略前两个示例插件中的指标即可。我怀疑开发人员在将演示插件拆分为多个示例时，只是忘记删除这些指标。

**清单 10-19** 包含了来自 `oracle.samples.xsh3` 插件的 ECM 元数据片段。此文件应放置在暂存区的 `<STAGE>/oms/metadata/snapshotlive` 中。

## 清单 10-19. ECM 元数据片段

```xml
<METADATAS>
  <METADATA SNAP_TYPE="HostSample3Snap" TARGET_TYPE="sample_host3" VER="1">
    <METADATA_UI_NAME>Hostsample Configuration Data</METADATA_UI_NAME>
    <TABLE NAME="MGMT_EMX_HS_SYSTEM" SINGLE_ROW="Y">
      <UI_NAME>Hostsample System Configuration</UI_NAME>
      <COLUMN NAME="hostname" TYPE="STRING" TYPE_FORMAT="256" IS_KEY="N">Hostname</COLUMN>
      <COLUMN NAME="version" TYPE="STRING" TYPE_FORMAT="256" IS_KEY="N">Version</COLUMN>
    </TABLE>
    ...
    <TABLE NAME="MGMT_EMX_HS_NET" SINGLE_ROW="N">
      <UI_NAME>Hostsample Network Configuration</UI_NAME>
      <COLUMN NAME="net_interface" TYPE="STRING" TYPE_FORMAT="32" IS_KEY="Y">Interface</COLUMN>
      <COLUMN NAME="net_mtu" TYPE="NUMBER" IS_KEY="N">Maximum Transmission Unit (bytes)</COLUMN>
      <COLUMN NAME="net_flag" TYPE="STRING" TYPE_FORMAT="16" IS_KEY="N">Flag</COLUMN>
    </TABLE>
  </METADATA>
</METADATAS>
```

ECM 元数据包含根元素 `METADATAS`，它只是一个容器，用于包含由 `METADATA` 元素定义的一个或多个快照。一个 `METADATA` 元素必须匹配到我稍后将在默认收集元数据中定义的一个 `CollectionItem`。可以将其视为一个配置快照。

每个配置快照定义一个或多个表，其中每个表匹配目标类型元数据中定义的一个配置指标。所有配置指标也必须包含在默认收集元数据中匹配的 `CollectionItem` 中。配置表和配置指标之间的列当然也必须匹配。多行表用 `SINGLE_ROW="N"` 属性标记，并且必须包含至少一个使用 `IS_KEY="Y"` 标记为键的列。由于你对元数据定义很熟悉，我就不逐一解释每个属性了。大多数属性都是自解释的，你可以在 *可扩展性程序员参考手册* 的第 6 章中找到完整的细节。

请注意，每次更改 ECM 元数据时，都需要增加其版本号（`VER` 属性），就像目标类型元数据一样。

文档中提到了 `generate_ecm_resources` 实用程序，它大概是 EDK 的一部分。该实用程序旨在生成配置指标模板以及相关的默认收集模板，以便所有表名、列名和其他项目都能匹配——我发现从头开始创建这些可能会相当混乱。不幸的是，在我使用的最新 EDK 发行版（随 EM12c 的 12.1.0.2 版本一起提供）中缺少这个实用程序，因此我无法利用它。

