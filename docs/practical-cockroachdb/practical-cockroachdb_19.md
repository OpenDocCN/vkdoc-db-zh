# 部署拓扑

`region=us-central1,zone=us-central1c`
`{4,6,9}`
```
{
"region=us-central1,zone=us-central1c",
"region=us-central1,zone=us-central1a",
"region=us-central1,zone=us-central1b"
}
```

`region=us-east1,zone=us-east1a`
`{1,2,3}`
```
{
"region=us-east1,zone=us-east1a",
"region=us-east1,zone=us-east1b",
"region=us-east1,zone=us-east1c"
}
```

`region=us-west1,zone=us-west1b`
`{5,7,8}`
```
{
"region=us-west1,zone=us-west1b",
"region=us-west1,zone=us-west1a",
"region=us-west1,zone=us-west1c"
}
```

### 全局表

全局表拓扑是读取密集型场景的一个不错选择，其中可以容忍写入延迟。读取将在区域内本地管理，因此速度很快，而写入将在区域间复制，因此会慢得多。

