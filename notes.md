## elestaicsearch
1. 装在虚拟机后需要转发网络
```yml
  network.host: 0.0.0.0
  http.port: 9200
  transport.host: localhost
  transport.tcp.port: 9300
```

2. 配置说明
```
  +- config
    +- elasticsearch.yml es相关配置
    +- jvm.options jvm 相关配置
    +- log4j2.propeities 日志相关配置
```

3. elasticsearch 配置说明
```
  cluster.name 集群名称，以此作为是否同一集群的判断条件
  node.name 节点名称，以此作为集群中不同节点的区分条件
  network.host/http.prot 网络地址和端口，用于 http 和 transport 服务使用
  path.data 数据存储地址
  path.log 日志存储地址
```

4. elasticsearch 的 development 和 production 模式
```
  以 transport 的地址是否绑定在 localhost 为判断标准 network.host
  development 模式下在启动时会以 warning 的方式提示配置检查异常
  production 模式下在启动时会以 error 的方式提示配置检查异常并退出
```

5. elasticsearch 本地集群启动方式
```
  1. bin/elasticsearch
  2. bin/elasticsearch -Ehttp.prot=8200 -Epath.data=node2
  3. bin/elasticsearch -Ehttp.prot=7200 -Epath.data=node3

  可以在浏览器地址栏中通过 _cat/nodes/v 查看集群
```

## kibana 安装与运行
1. 配置说明
```
  +- config
    +- kibana.yml
  
  server.host/server.port 访问 kibana 用的地址和端口
  elasticsearch.url 待访问 elasticsearch 的地址
```

2. 常用功能
```
  Discover 数据搜索查看
  Visualize 图表制作
  Dashboard 仪表盘制作
  Timelion 时序数据的高级可视化分析
  DevTools 开发者工具
  Management 配置
```

3. elasticsearch 常用术语
```
  Document 文档数据
  Index 索引
  Type 索引中的数据类型
  Field 字段，文档的属性
  Query DSL 查询语法
```

4. elasticsearch CRUD
```
  Create 创建文档
  Read 读取文档
  Update 更新文档
  Delete 删除文档
```

## beats