# CBO 估算过程的主要输入

CBO 估算过程的主要输入是保存在数据字典中的 `object statistics`。这些统计信息会指出，例如，一个表有多少数据块，因此，要完整读取它需要多少次多块读取。索引也保存有统计信息，因此 CBO 有一定的依据来估算使用索引从表中读取数据需要多少次单块读取。

对象统计信息是 CBO 成本计算算法最重要的输入，但绝不是唯一的输入。`Initialization parameter settings`、`system statistics`、`dynamic sampling, SQL profiles` 以及 `SQL baselines` 都是 CBO 在进行成本估算时考虑的其他因素的例子。

