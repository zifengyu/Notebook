#work #gdi #fluentd

## Fluentd配置
使用Fluentd，可以将各个数据源的数据，通过TCP传输到yhp系统当中。Fluentd的[安装](https://docs.fluentd.org/)后，通过配置文件，指定数据源`<source>`、`<filter>`和输出`<output>`。

### Source

Fluentd的source插件定义获取数据的数据源，以下给出一些针对不同数据源的配置示例。其中，{{var}}的参数需要根据实际情况指定。

#### 文件

[tail](https://docs.fluentd.org/input/tail)插件可以监控磁盘问上的一个或者多个文本文件，类似于tail -F

```yaml
<source>
  @type tail
  path {{file_path}}
  pos_file {{pos_file_path}}
  tag {{datatype}}
  <parse>
    @type none
  </parse>
</source>
```

其中，

* path表明监控的文件，支持多个文件和`*` 
* tag用于Fluentd路由，并且在_datatype没有设置时，作为默认的datatype值。
* `<parse>`用于解析文件的内容，如果不需要Fluentd进行解析，选择none


### 数据库

[sql](https://github.com/fluent/fluent-plugin-sql)插件可以定期从数据表中抓取数据。注意这个插件和相应的RDBMS驱动需要额外安装gem。

```yaml
<source>
  @type sql
  host {{db_host}}
  database {{db_name}}
  adapter mysql2
  username {{username}}
  password {{password}}
  
  state_file /var/run/fluentd/{{db_name}}_state
  select_limit 1000
  select_interval 10
  tag_prefix {{tag_prefix}}
  <table>
    table {{table_name}}
    update_column {{update_column}}
    tag {{table_tag}}
  </table>
  <table>
    ...
  </table>
</source>
```

其中，

* host, database, username, password为数据库连接参数。
* 可以包含多个`<table>`，从多个表抓取数据。生成数据的tag为{{tag_prefix}}.{{table_tag}}
* update_column为一个值递增的列，用于标记哪些行记录还没有被抓去。具体可参见插件文档。

### Filter

Filter中，主要使用[record_transformer](https://docs.fluentd.org/filter/record_transformer)，设置事件的属性。指定导入的数据类型(_datatype)和数据集(\_event_set)。

```yaml
<filter **>
  @type record_transformer
  remove_keys message
  <record>
    _message ${record["message"]}
    _datatype {{datatype}}
    _event_set {{event_set}}
    _time 0
  </record>
</filter>
```

其中，

* 将原始事件message改名为yhp的属性名_message
* 指定数据类型_datatype（覆盖默认的类型值tag）
* 指定导入的目标数据集_event_set
* 指定时间戳_time为0，表明让yhp自动从事件中检测时间戳。

### Output

输出插件[forward](https://docs.fluentd.org/output/forward)，通过TCP将事件发送给yhp中的parana service，再导入stonewave中。配置如下，具体需要指定到部署的parana服务IP和端口。

```yaml
<match **>
    @type forward
    <server>
      name {{output_name}}
      host {{parana_service_host}}
      port {{parana_service_port}}
    </server>
</match>
```

