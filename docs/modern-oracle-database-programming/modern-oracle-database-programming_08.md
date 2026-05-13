# Intersect

`intersect` 操作符仅筛选出同时存在于两个输入集合中的行。上一节中的 `grahams_and_hills` 视图被用来演示此操作。

在 `intersect` 操作符之前的第一个集合是视图中的所有行（共六条记录）。在 `intersect` 操作符之后的第二个集合被筛选为仅包含 Graham Hill（两条记录）。

```
select gh.driverid
, gh.driverref
, gh.forename
, gh.surname
, gh.nationality
from   grahams_and_hills gh
intersect
select gh.driverid
, gh.driverref
, gh.forename
, gh.surname
, gh.nationality
from   grahams_and_hills gh
where  gh.driverref = 'hill'
/
DRIVERID DRIVERREF        FORENA SURNAME   NATIONALITY
---------- ---------------- ------ --------- -------------
289 hill             Graham Hill      British
```

与 `union` 操作符类似，这会对结果集执行去重操作。因此，即使 *Graham Hill* 在两个集合中都出现了两次，他在最终结果集中也只出现一次。

从 Oracle Database 21c 开始，可以通过为其添加 `distinct` 来更明确地编写此 `intersect` 操作符。

```
intersect distinct
```

其工作原理保持不变；它增加了语法上的清晰度。

从 Oracle Database 21c 开始，`intersect` 操作符就有了 `all` 版本。如果将脚本改为使用 `intersect all` 操作符，则会得到不同的结果。

```
select gh.driverid
, gh.driverref
, gh.forename
, gh.surname
, gh.nationality
from   grahams_and_hills gh
intersect all
select gh.driverid
, gh.driverref
, gh.forename
, gh.surname
, gh.nationality
from   grahams_and_hills gh
where  gh.driverref = 'hill'
/
DRIVERID DRIVERREF        FORENA SURNAME   NATIONALITY
---------- ---------------- ------ --------- -------------
289 hill             Graham Hill      British
289 hill             Graham Hill      British
```

正如您在结果集中看到的，Graham Hill 在每个结果集中出现了两次，在最终结果集中也出现了两次。



