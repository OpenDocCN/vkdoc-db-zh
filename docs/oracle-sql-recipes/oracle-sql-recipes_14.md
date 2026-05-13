# 引言

在典型的 Oracle 数据库安装过程中，如果通过 Oracle Universal Installer 执行完整安装，将会安装六个示例模式。如果你打算使用的数据库未进行此安装，请无需惊慌。你可以在任何时候通过运行 Linux 或 Unix 系统下的 `ORACLE_HOME/demo/schema/mksample` 脚本，来添加（或刷新）示例模式。在 Windows 系统下，对应的脚本是 `%ORACLE_HOME%\demo\schema\mksample`。

我们建议你重新运行数据库配置助手（Database Configuration Assistant）来代劳，因为这将使你免于在手动脚本提示时费力寻找某些晦涩的选项。

如果你真的想成为示例数据方面的专家，那么你会欣慰地得知，Oracle 的示例模式现已庞大到拥有自己专属的手册。你可以从 Oracle 文档网站 **http://www.oracle.com/docs** 下载《Oracle Database Sample Schemas 11g Release 1》指南（其中包含了错误的复数形式“schemas”）。该手册的零件号是 B28328-03，你会找到 HTML 和 Adobe Acrobat 两种格式的资料。

## 约定

在全书中，我们保持了呈现 SQL 语句及其结果的一致风格。当代码片段、SQL 保留字或 SQL 片段出现在正文中时，它们会以等宽 Courier 字体呈现，例如这个（可运行的）示例：`select * from dual;`

在我们讨论 SQL 命令的语法和选项时，我们采用了对话式的风格，以便你能快速理解命令或技术。这意味着我们没有复制那些更适合参考手册的大型语法图示。

## 联系作者

如果你有任何问题或意见——甚至发现了你认为我们应该知道的错误——你可以通过本书的网站 **www.oraclesqlrecipes.com** 联系作者。如需直接联系 Bob Bryla，请发送电子邮件至 rjbdba@gmail.com。

xxv

[www.it-ebooks.info](http://www.it-ebooks.info/)

[www.it-ebooks.info](http://www.it-ebooks.info/)

