# Fluentd-Configuration

<div  style="font-size: 26px; transform:translate(649px,58px)"><strong>Riku</strong></div> <div style="font-size: 17px; transform:translate(625px,58px)">
    <strong>2021年5月6日</strong>
</div>	



## 一：conf配置解释

[toc]
### 1. 配置文件的语法

配置文件位置：

```shell
/etc/td-agent/td-agent.conf
```

指令列表：

1. source
   - 决定输入源
2. match
   - 决定输出目的地
3. filter
   - 决定事件处理管道
4. system
   - 设置系统范围的配置
5. label
   - 作为内部路由对输出和过滤器进行分组
6. @include
   - 包括其他文件

### 2. source：这些数据从何而来

通过使用源指令选择和配置所需的输入插件，可以启用Fluentd输入源。

Fluentd标准输入插件包括**http**和**forward**。

- http提供一个http端点来接受传入的http消息。

- forward提供一个TCP端点来接受TCP数据包。

当然，这两者也可以同时进行。您可以根据需要添加多个源配置。

```shell
# Receive events from 24224/tcp
# This is used by log forwarding and the fluent-cat command
<source>
  @type forward
  port 24224
</source>

# http://<ip>:9880/myapp.access?json={"event":"data"}
<source>
  @type http
  port 9880
</source>
```

每个source指令必须包含一个@type参数来指定要使用的输入插件。

### 3. Routing

source将事件提交给Fluentd路由引擎。

一个事件由三个实体组成:**tag**、**time** 和**record**。

- **tag**是一个由点分隔的字符串(例如myapp.access)，用作Fluentd内部路由引擎的方向。

- **time**字段由输入插件指定，且必须为Unix时间格式。
- **record**是一个JSON对象。

> Fluentd接受所有非句号字符作为标记的一部分。然而，由于标签有时会在不同的上下文中被输出目的地使用(例如表名、数据库名、键名等)，**因此强烈建议您坚持使用小写字母、数字和下划线**(例如^[a-z0-9_]+$)。

在前面的例子中，HTTP输入插件提交了以下事件:

```shell
# generated by http://<ip>:9880/myapp.access?json={"event":"data"}
tag: myapp.access
time: (current time)
record: {"event":"data"}
```

### 4. match:告诉fluentd要做什么!

**match**指令查找带有匹配**tags**的事件并处理它们。match指令最常见的用法是将事件输出到其他系统。由于这个原因，与match指令对应的插件被称为**输出插件**。Fluentd标准输出插件包括**file**和**forward**。让我们将这些添加到配置文件中。

```shell
# Receive events from 24224/tcp
# This is used by log forwarding and the fluent-cat command
<source>
  @type forward
  port 24224
</source>

# http://<ip>:9880/myapp.access?json={"event":"data"}
<source>
  @type http
  port 9880
</source>

# Match events tagged with "myapp.access" and
# store them to /var/log/fluent/access.%Y-%m-%d
# Of course, you can control how you partition your data
# with the time_slice_format option.
<match myapp.access>
  @type file
  path /var/log/fluent/access
</match>
```

每个**match**指令必须包含一个匹配模式和一个@type参数。只有带有与模式匹配的**tag**事件才会被发送到输出目的地(在上面的示例中，只有带有标记**myapp.access**的事件才会匹配。**@type**参数指定要使用的输出插件。

### 5. filter:事件处理管道

**filter**指令具有与match相同的语法，但是filter可以被链接用于处理管道。使用**filters**，事件流是这样的:

```shell
Input -> filter 1 -> ... -> filter N -> Output
```

添加标准record_transformer过滤器来match示例。

```shell
# http://this.host:9880/myapp.access?json={"event":"data"}
<source>
  @type http
  port 9880
</source>

<filter myapp.access>
  @type record_transformer
  <record>
    host_param "#{Socket.gethostname}"
  </record>
</filter>

<match myapp.access>
  @type file
  path /var/log/fluent/access
</match>
```

 [Visualizaion可视化：](https://link.calyptia.com/6mk)

![image-20210505153634314](https://gitee.com/chobits01/xiaoming/raw/master/img/image-20210505153634314.png)接收到的事件**{"event":"data"}**首先转到**record_transformer**过滤器。record_transformer过滤器将**host_param字段**添加到事件;

然后过滤事件**{"event":"data"，"host_param":"webserver1"}**转到**file**输出插件。

### 6. 设置系统范围的配置:system指令

系统范围的配置是由**system**指令设置的。它们中的大多数也可以通过命令行选项获得。例如，有以下配置:

- log_level
- suppress_repeated_stacktrace
- emit_error_log_interval
- suppress_config_dump
- without_source
- process_name（仅在系统指令中可用。没有fluentd选择）

例如：

```shell
<system>
  # equal to -qq option
  log_level error
  # equal to --without-source option
  without_source
  # ...
</system>
```

具体设置参考：

<center><a href="https://docs.fluentd.org/deployment/system-config">System Configuration</a></center>

**process_name:**

如果设置此参数，fluentd的supervisor和worker进程名称将被更改。

```shell
<system>
  process_name fluentd1
</system>
```

通过配置，ps命令显示如下结果:

```shell
% ps aux | grep fluentd1
foo      45673   0.4  0.2  2523252  38620 s001  S+    7:04AM   0:00.44 worker:fluentd1
foo      45647   0.0  0.1  2481260  23700 s001  S+    7:04AM   0:00.40 supervisor:fluentd1
```

### 7. filter和output形成的组：label指令

**label** 指令组过滤并输出内部路由。**label** 降低了tag处理的复杂性。

**label**参数是一个内置插件参数，所以需要@前缀。

下面是一个配置示例:

```shell
<source>
  @type forward
</source>

<source>
  @type tail
  @label @SYSTEM
</source>

<filter access.**>
  @type record_transformer
  <record>
    # ...
  </record>
</filter>
<match **>
  @type elasticsearch
  # ...
</match>

<label @SYSTEM>
  <filter var.log.middleware.**>
    @type grep
    # ...
  </filter>
  <match **>
    @type s3
    # ...
  </match>
</label>
```

![image-20210505153442440](https://gitee.com/chobits01/xiaoming/raw/master/img/image-20210505153442440.png)

在此配置中，forward事件被路由到record_transformer  filter 然后到 elasticsearch输出，in_tail事件被路由到@SYSTEM label内的grep filter 然后到 s3输出。

label参数对于没有标记前缀的事件流分离非常有用。

**@ERROR标签**

@ERROR标签是一个内置标签，用于插件的emit_error_event API发出的错误记录。

如果设置了<label @ERROR>，当相关错误被触发时(例如缓冲区已满或记录无效)，事件将被路由到这个标签。

### 8. 重用你的配置:@include指令

单独的配置文件中的指令可以使用@include指令导入

```shell
# Include config files in the ./config.d directory
@include config.d/*.conf
```

@include指令支持常规**文件路径**、**glob模式**和**http URL**约定:

```shell
# absolute path
@include /path/to/config.conf

# if using a relative path, the directive will use
# the dirname of this config file to expand the path
@include extra.conf

# glob match pattern
@include config.d/*.conf

# http
@include http://example.com/fluent.conf
```

注意，对于glob模式，文件是按字母顺序展开的。如果有a.conf和b.conf，那么fluentd首先解析a.conf。但是，你不应该写依赖于这个顺序的配置。它是如此容易出错，因此，为了安全起见，使用多个独立的@include指令。

```shell
# If you have a.conf, b.conf, ..., z.conf and a.conf / z.conf are important

# This is bad
@include *.conf

# This is good
@include a.conf
@include config.d/*.conf
@include z.conf
```

#### 共享相同的参数

@include指令可以在section下使用，以共享相同的参数:

```shell
# config file
<match pattern>
  @type forward
  # ...
  <buffer>
    @type file
    path /path/to/buffer/forward
    @include /path/to/out_buf_params.conf
  </buffer>
</match>

<match pattern>
  @type elasticsearch
  # ...
  <buffer>
    @type file
    path /path/to/buffer/es
    @include /path/to/out_buf_params.conf
  </buffer>
</match>

# /path/to/out_buf_params.conf
flush_interval    5s
total_limit_size  100m
chunk_limit_size  1m
```

### 10. 匹配模式是如何工作的?

如上所述，Fluentd允许根据事件的标记路由事件。虽然您可以指定要匹配的确切标记(如<filter app.log>)，但您可以使用许多技术来更有效地管理数据流。

#### 通配符、扩展和其他提示

下面的匹配模式可以用于<match>和<filter>标签:

- *匹配单个标记部分。
  - 例如，模式a.*匹配a.b，但不匹配a或a.b.c

- **匹配零个或多个标记部分。
  - 例如，模式a.**匹配a, a.b, a.b.c

- {X,Y,Z}匹配X,Y或Z，其中X,Y和Z是匹配模式。
  - 例如，模式{a,b}匹配a和b，但不匹配c
  - 这可以与*或**模式结合使用。例子:包括a.{b,c}.\* 和a.{b,c.\*\*}
- /regular expression/用于复杂模式
  - 例如，模式 `/(?!a\.).*` 匹配非a.开头的标签,比如b.xxx
  - 从fluentd v1.11.2开始就支持该特性
- \#{…}以Ruby表达式的形式计算括号内的字符串。(参见下面嵌入Ruby表达式一节)。
- 当多个模式被列在单个**tag** (由一个或多个空白分隔)内时，它将匹配所列的任何模式。例如:
  - 模式<match a b>匹配a和b。
  - <match a.\**  b.*> match a, a.b, a.b.c(第一个模式)和b.d(第二个模式)

### 11. 注意匹配顺序

Fluentd试图按照标记在配置文件中出现的顺序匹配它们。所以，如果你有以下配置:

```shell
# ** matches all tags. Bad :(
<match **>
  @type blackhole_plugin
</match>

<match myapp.access>
  @type file
  path /var/log/fluent/access
</match>
```

然后myapp.access从不匹配。更广泛的匹配模式应该定义再严格匹配之后。

```shell
<match myapp.access>
  @type file
  path /var/log/fluent/access
</match>

# Capture all unmatched tags. Good :)
<match **>
  @type blackhole_plugin
</match>
```

当然，如果您使用两个相同的模式，第二个匹配将永远不会被匹配。如果你想把事件发送到多个输出，考虑out_copy插件。

常见的陷阱是当你在<match>后面放一个<filter>块时。由于上述原因，它永远不会工作，因为事件永远不会通过过滤器。

```shell
# You should NOT put this <filter> block after the <match> block below.
# If you do, Fluentd will just emit events without applying the filter.

<filter myapp.access>
  @type record_transformer
  ...
</filter>

<match myapp.access>
  @type file
  path /var/log/fluent/access
</match>
```

### 12. 嵌入Ruby表达式

由于Fluentd v1.4.0，您可以使用#{…}来将任意的Ruby代码嵌入匹配模式中。下面是一个例子:

```ruby
<match "app.#{ENV['FLUENTD_TAG']}">
  @type stdout
</match>
```

如果将环境变量FLUENTD_TAG设置为dev，则计算结果为app.dev。

### 13. Values的支持数据类型

每个Fluentd插件都有自己特定的参数集。例如，in_tail有**rotate_wait**和**pos_file**等参数。每个参数都有一个与之相关联的特定类型。类型定义如下:

- 字符串:该字段被解析为一个字符串。这是最通用的类型，每个插件决定如何处理字符串。

  - 字符串有三个字面值:不带引号的一行字符串，`'`单引号字符串和`“`双引号字符串。

- Integer:解析为整数。

- float:字段被解析为浮点型。

- Size:该字段被解析为字节数。有几个符号的变化:

  ```shell
  <INTEGER>k or <INTEGER>K: number of kilobytes
  <INTEGER>m or <INTEGER>M: number of megabytes
  <INTEGER>g or <INTEGER>G: number of gigabytes
  <INTEGER>t or <INTEGER>T: number of terabytes
  # 否则，该字段被解析为整数，该整数是字节数。
  ```

- Time:该字段被解析为一个时间持续时间。

```shell
<INTEGER>s: seconds
<INTEGER>m: minutes
<INTEGER>h: hours
<INTEGER>d: days
# 否则，该字段被解析为float，该float是秒数。此选项用于指定次秒持续时间，如0.1(0.1秒= 100毫秒)。
```

- array:该字段被解析为JSON数组。它还支持速记语法。这些是相同的值:

```shell
normal: ["key1", "key2"]
shorthand: key1,key2
```

- hash:该字段被解析为一个JSON对象。它还支持速记语法。这些是相同的值:

```shell
normal: {"key1": "value1", "key2": "value2"}
shorthand: key1:value1,key2:value2
```

**array**和**hash**类型是JSON，因为几乎所有编程语言和基础设施工具都可以轻松生成JSON值，而不是其他任何不寻常的格式。

### 14. 常见的插件参数

这些参数被保留，并以@符号作为前缀:

- @type:插件类型

- @id:插件id. `In_monitor_agent`将此值用于plugin_id领域

- @label:标签符号。
- @log_level:每个插件的日志级别

### 15. 检查配置文件

配置文件可以在不启动插件的情况下使用——dry-run选项进行验证:

```shell
fluentd --dry-run -c fluent.conf
```

### 16. 介绍配置文件的一些有用特性

多行支持“引用字符串、数组和散列值”

```shell
str_param "foo  # Converts to "foo\nbar". NL is kept in the parameter
bar"
array_param [
  "a", "b"
]
hash_param {
  "k": "v",
  "k1": 10
}
```

fluentd假定\[或\{开头的为数组或哈希，因此，如果想设置\[或\{开头,但不是json的格式，请使用`'`,`“`

比如：

**mail plugin**

```shell
<match **>
  @type mail
  subject "[CRITICAL] foo's alert system"
</match>
```

**map plugin**

```shell
<match tag>
  @type map
  map '[["code." + tag, time, { "code" => record["code"].to_i}], ["time." + tag, time, { "time" => record["time"].to_i}]]'
  multi true
</match>
```

这个限制将随着配置解析器的改进而消除。

**嵌入Ruby代码**

你可以用"引号字符串"中的#{}来计算Ruby代码。这对于设置机器信息很有用，例如主机名。

```shell
host_param  "#{Socket.gethostname}" # host_param is actual hostname like `webserver1`.
env_param   "foo-#{ENV["FOO_BAR"]}" # NOTE that foo-"#{ENV["FOO_BAR"]}" doesn't work.
```

从v1.1.0开始，主机名和worker_id快捷方式可用:

```shell
host_param  "#{hostname}"  # This is same with Socket.gethostname
@id         "out_foo#{worker_id}" # This is same with ENV["SERVERENGINE_WORKER_ID"]
```

worker_id快捷方式在多个worker下是有用的。例如，对于单独的插件id，添加worker_id来存储s3中的路径，以避免文件冲突。

从1.8.0版本开始，helper方法use_nil和use_default是可用的:

```shell
some_param  "#{ENV["FOOBAR"] || use_nil}"     # Replace with nil if ENV["FOOBAR"] isn't set
some_param  "#{ENV["FOOBAR"] || use_default}" # Replace with the default value if ENV["FOOBAR"] isn't set
```

注意，这些方法不仅会将内嵌的Ruby代码替换掉，还会将整个字符串替换为nil或默认值。

```shell
some_path   "#{use_nil}/some/path" # some_path is nil, not "/some/path"
```

config-xxx mixins使用“${}”，而不是“#{}”。这些嵌入式配置是两个不同的东西。

**在双引号字符串字面量中，\是转义字符**

正斜杠\被解释为转义字符。你需要“设置”,“r”,“n”,“t”或一些双引号字符串字面量中的字符。

```shell
str_param   "foo\nbar" # \n is interpreted as actual LF character
```

## 二：Routing Examples

本文展示了典型路由场景的配置示例。

### 简单：Input -> Filter -> Output

```shell
<source>
  @type forward  #(收集tcp数据包)
</source>

<filter app.**>
  @type record_transformer
  <record>
    hostname "#{Socket.gethostname}"
  </record>
</filter>

<match app.**>
  @type file
  # ...
</match>
```

### 两个输入：forward and tail

```shell
<source>
  @type forward
</source>

<source>
  @type tail
  tag system.logs
  # ...
</source>

<filter app.**>
  @type record_transformer
  <record>
    hostname "#{Socket.gethostname}"
  </record>
</filter>

<match {app.**,system.logs}>
  @type file
  # ...
</match>
```

如果要分隔每个源的数据管道，请使用Label。

### 带Label: Input -> Filter -> Output

标签通过分离数据管道减少了复杂的标签处理。

```shell
<source>
  @type forward
</source>

<source>
  @type dstat
  @label @METRICS # dstat events are routed to <label @METRICS>
  # ...
</source>

<filter app.**>
  @type record_transformer
  <record>
    # ...
  </record>
</filter>

<match app.**>
  @type file
  # ...
</match>

<label @METRICS>
  <match **>
    @type elasticsearch
    # ...
  </match>
</label>
```

### 通过Tag重新路由事件

使用fluent-plugin-route插件。这个插件重写标签并重新发出事件到其他match或Label。

```shell
<match worker.**>
  @type route
  remove_tag_prefix worker
  add_tag_prefix metrics.event

  <route **>
    copy # For fall-through. Without copy, routing is stopped here. 
  </route>
  <route **>
    copy
    @label @BACKUP
  </route>
</match>

<match metrics.event.**>
  @type stdout
</match>

<label @BACKUP>
  <match metrics.event.**>
    @type file
    path /var/log/fluent/backup
  </match>
</label>
```

### 根据记录内容重新路由事件

使用 [fluent-plugin-rewrite-tag-filter](https://github.com/fluent/fluent-plugin-rewrite-tag-filter)

```shell
<source>
  @type forward
</source>

# event example: app.logs {"message":"[info]: ..."}
<match app.**>
  @type rewrite_tag_filter
  <rule>
    key message
    pattern ^\[(\w+)\]
    tag $1.${tag}
  </rule>
  # more rules
</match>

# send mail when receives alert level logs
<match alert.app.**>
  @type mail
  # ...
</match>

# other logs are stored into a file
<match *.app.**>
  @type file
  # ...
</match>
```

See also: [out_rewrite_tag_filter]()

### 重新路由事件到其他标签

使用**out_relabel**插件。这个插件只是向Label发送事件，而不需要重写tag

```shell
<source>
  @type forward
</source>

<match app.**>
  @type copy
  <store>
    @type forward
    # ...
  </store>
  <store>
    @type relabel
    @label @NOTIFICATION
  </store>
</match>

<label @NOTIFICATION>
  <filter app.**>
    @type grep
    regexp1 message ERROR
  </filter>

  <match app.**>
    @type mail
  </match>
</label>
```

## 三：配置:常见的参数

一些常用参数可用于所有或部分Fluentd插件。本页面描述这些参数。

### 所有插件的参数

`@type`

@type参数指定插件的类型。

```shell
<source>
  @type my_plugin_type
</source>

<filter>
  @type my_filter
</filter>
```

`@id`

@id参数为配置指定一个唯一的名称。它被用作缓冲区、存储、日志记录和其他用途的路径。

```shell
<match>
  @type file
  @id service_www_accesslog
  path /path/to/my/access.log
  # ...
</match>
```

应该为所有插件指定这个参数，以便全局启用root_dir和workers特性。

See also: [System Configuration]()

`@log_level`

此参数指定插件特定的日志级别。默认日志级别为info。全局日志级别可以通过在<system> section 中设置log_level或使用-v/-q命令行参数来指定。@log_level参数只覆盖指定插件实例的日志级别。

```shell
<system>
  log_level info
</system>

<source>
  # ...
  @log_level debug # shows debug log only for this plugin
</source>
```

该参数的主要用途是:

1. 为插件抑制过多的日志;

2. 显示调试日志以帮助调试过程;

Please see the [logging article]() for further details.

### 触发事件的插件参数

`@label`

@label参数将输入事件路由到<label>部分，即<filter>和<match>一组的子部分在<label>下的集合。

```shell
<source>
  @type ...
  @label @access_logs
  # ...
</source>

<source>
  @type ...
  @label @system_metrics
  # ...
</source>

<label @access_log>
  <match **>
    @type file
    path ...
  </match>
</label>

<label @system_metrics>
  <match **>
    @type file
    path ...
  </match>
</label>
```

注意:@label参数的值必须以@字符开头。

强烈建议使用@label将事件路由到任何插件，而无需修改标签。它有助于使复杂的配置模块化和简单化。

## 四：配置:解析部分

一些Fluentd插件支持<parse>部分来指定如何解析原始数据。

### 解析部分概述

parse 语句可以位于<source>， <match>或<filter> 语句下。它为支持解析器插件特性的插件启用。

```shell
<source>
  @type tail
  # ...
  <parse>
    # ...
  </parse>
</source>
```

### 解析器插件类型

<parse>部分的@type参数指定了解析器插件的类型。Fluentd核心捆绑了一些有用的解析器插件。

```shell
<parse>
  @type apache2
</parse>
```

### 参数

`@type`

@type参数指定解析器插件的类型。

```shell
<parse>
  @type regexp
  # ...
</parse>
```

以下是内置解析器插件的列表:

- `regexp`
- `apache2`
- `apache_error`
- `nginx`
- `syslog`
- `csv`
- `tsv`
- `ltsv`
- `json`
- `multiline`
- `none`

### 解析参数

以下参数的默认值将被各个解析器插件覆盖:

- `types` (hash)(可选):指定将字段转换为另一个字段的类型

  - Default: ==nil==

  - 基于字符串的哈希: ==field1:type, field2:type, field3:type:option,==

    ==field4:type:option==

  - JSON格式:  =={"field1":"type", "field2":"type", "field3":"type:option",==

    =="field4":"type:option"}==

  - example:  types user_id:integer,paid:bool,paid_usd_amount:float

- `time_key ` (string)(可选):指定事件时间字段。如果事件没有此字段，使用当前时间。
  - Default： ==nil==
  - 注意: json,ltsv和regexp覆盖此值的默认值参数，并默认设置为time
- `null_value_pattern` (string)(可选):指定空值模式。
  
  - Default: ==nil==
- `null_empty_string` (bool)(可选):如果为真，则空字符串字段被替换为nil
  
- default: false
  
- `estimate_current_event` (bool)(可选): 当time_key被指定时，如果为true，使用Fluent::EventTime.now(current time)作为时间戳
  
  - Default: true
- `keep_time_key`(bool)(可选):如果为true，则keep time字段在record里。
  
  - Default:false
- `timeout`  (time)(可选):指定解析处理超时。这主要用于检测错误的regexp模式。
  
  - Default：nil

### types 参数

对于types参数，支持以下类型:

- string: 将字段转换为字符串类型。这使用to_s方法进行转换。

- bool: 将字符串"true"， "yes"或"1"转换为true。否则,false。

- integer: (not int):将字段转换为Integer类型。这使用to_i方法进行转换。例如，字符串“1000”可以转换为1000。

- float: 将字段转换为Float类型。这使用to_f方法进行转换。例如，字符串"7.45"可以转换成7.45。

- time: 将字段转换为Fluent::EventTime类型。它使用Fluentd时间解析器进行转换。对于time类型，第三个字段指定的时间格式类似于time_format。

  ```shell
    date:time:%d/%b/%Y:%H:%M:%S %z # for string with time format
    date:time:unixtime             # for integer time
    date:time:float                # for float time
  ```

- array: 将字符串字段转换为Array类型。对于Array类型，第三个字段指定分隔符(默认为逗号“，”)。例如，如果字段item_ids包含值“3,4,5”，则将item_ids:array解析为["3"，"4"，"5"]。或者，如果值为"Adam|Alice|Bob"，则types item_ids:array:| 将其解析为["Adam"， "Alice"， "Bob"]。

### Time 参数

- time_type: (enum)(可选):根据此类型解析/格式化该值
  - Default: `float`
  - Available values: `float`, `unixtime`, `string`, `mixed`
- time_format：(string)(可选):根据指定的格式处理值。只有time_type为字符串时才可用。
  - Default: `nil`
- localtime： (bool)(可选):如果为true，则使用本地时间。否则,使用UTC。这与utc是独占的。
- utc： 如果为true，使用UTC。否则,使用local time。这与本地时间是独占的。
- timezone： (string)(可选):使用指定的时区。一个人可以将时间值解析为指定的时区格式。
  - Default: `nil`
- time_format_fallbacks： (可选):以指定的顺序使用指定的时间格式作为回退。您可以使用time_format_fallbacks解析未确定的时间格式。当混合使用time_type时启用此选项。
  - Default: `nil`

```shell
 time_type mixed
 time_format unixtime
 time_format_fallbacks %iso8601
```

在上面的用例中，时间戳首先被解析为**unixtime**，如果它失败，那么它被解析为**%iso8601**备用。注意，time_format_fallbacks是解析混合时间戳格式的最后一种方法。这会造成性能损失(通常，在time_format_fallbacks中指定了N个回退，如果使用最后指定的格式作为回退，在最坏的情况下会慢N倍)。

## 配置: Buffer部分

Buffer部分位于<match>部分之下。它为那些支持`被缓冲输出`特性的输出插件启用。

```shell
<match tag.*>
  @type file
  # ...
  <buffer>
    # ...
  </buffer>

  # <buffer> section can only be configured once!
</match>
```

### Buffer 插件类型

<buffer>的@type参数指定了缓冲区插件的类型:

```shell
<buffer>
  @type file
</buffer>
```

Fluentd核心包 **file**和**memory** buffer 插件，即:

- buf_file
- buf_memory

也可以安装和配置第三方插件。

但是，@type参数不是必需的。如果省略，默认情况下，使用输出插件指定的缓冲区插件(如果可能的话)。否则，使用内存缓冲区插件。

对于通常的workload，建议使用文件缓冲区插件。对于一般用例，它更持久。

### Chunk Keys

输出插件将事件分组到chunks中。Chunk keys，指定为<buffer> section的参数，控制如何将事件分组到Chunk中。

```shell
<buffer ARGUMENT_CHUNK_KEYS>
  # ...
</buffer>
```

如果指定，chunk key参数必须是逗号分隔的字符串。

### Blank Chunk Keys

在没有或空白chunk key的情况下，输出插件会将所有匹配的事件写到一个块中，直到超过块的大小，前提是输出插件本身没有指定任何默认的块键。

```shell
<match tag.**>
  # ...
  <buffer>      # <--- No chunk key specified as argument
    # ...
  </buffer>
</match>

# No chunk keys: All events will be appended into the same chunk.

11:59:30 web.access {"key1":"yay","key2":100}  --|
                                                 |
12:00:01 web.access {"key1":"foo","key2":200}  --|---> CHUNK_A
                                                 |
12:00:25 ssh.login  {"key1":"yay","key2":100}  --|
```

### Tag

如果Tag被指定为块键，输出插件将事件写入按标记分组的块中。具有不同标记的事件将被写入不同的块中。

```shell
<match tag.**>
  # ...
  <buffer tag>
    # ...
  </buffer>
</match>

# Tag chunk key: The events will be grouped into chunks by tag.

11:59:30 web.access {"key1":"yay","key2":100}  --|
                                                 |---> CHUNK_A
12:00:01 web.access {"key1":"foo","key2":200}  --|

12:00:25 ssh.login  {"key1":"yay","key2":100}  ------> CHUNK_B
```

### Time

如果指定了参数time和参数timekey(必需的)，输出插件将事件写入到按时间键分组的块中。

时间键是这样计算的:

```shell
time (unix time) / timekey (seconds)
```

For example:

- timekey 60: `["12:00:00", ..., "12:00:59"]`, `["12:01:00", ..., "12:01:59"]`,

  ...

- timekey 180: `["12:00:00", ..., "12:02:59"]`, `["12:03:00", ..., "12:05:59"]`,

  ...

- timekey 3600: `["12:00:00", ..., "12:59:59"]`, `["13:00:00", ...,"13:59:59"]`,

  ...

这些事件将根据其时间范围被分组成块。它们将在时间键范围过期后被输出插件刷新。

```shell
<match tag.**>
  # ...
  <buffer time>
    timekey      1h # chunks per hours ("3600" also available)
    timekey_wait 5m # 5mins delay for flush ("300" also available)
  </buffer>
</match>

# Time chunk key: The events will be grouped by timekey with timekey_wait delay.

11:59:30 web.access {"key1":"yay","key2":100}  ------> CHUNK_A

12:00:01 web.access {"key1":"foo","key2":200}  --|
                                                 |---> CHUNK_B
12:00:25 ssh.login  {"key1":"yay","key2":100}  --|
```

**timekey_wait**参数配置事件的刷新延迟。默认值是600(10分钟)。

事件时间通常是从当前时间戳开始的延迟时间。Fluentd将等待为延迟事件刷新缓冲的块。例如，下图显示了块(timekey: 3600)实际刷新的时间，用于示例timekey_wait值:

```shell
 timekey: 3600
 -------------------------------------------------------
 time range for chunk | timekey_wait | actual flush time
  12:00:00 - 12:59:59 |           0s |          13:00:00
  12:00:00 - 12:59:59 |     60s (1m) |          13:01:00
  12:00:00 - 12:59:59 |   600s (10m) |          13:10:00
```

### Other Keys

其他键(非时间/非标记)作为记录的字段名处理。输出插件将根据这些字段的值将事件分组为块。

```shell
<match tag.**>
  # ...
  <buffer key1>
    # ...
  </buffer>
</match>

# Chunk keys: The events will be grouped by values of "key1".

11:59:30 web.access {"key1":"yay","key2":100}  --|---> CHUNK_A
                                                 |
12:00:01 web.access {"key1":"foo","key2":200}  --|---> CHUNK_B
                                                 |
12:00:25 ssh.login  {"key1":"yay","key2":100}  --|---> CHUNK_A
```

### Nested Field Support

嵌套字段可以使用record_accessor语法指定。

Example:

```shell
<match tag.**>
  # ...
  <buffer $.nest.field> # access record['nest']['field']
    # ...
  </buffer>
</match>
```

### Combination of Chunk Keys

两个或多个块键可以组合在一起。事件将根据这些组合块键的值组合成块。

```shell
# <buffer tag,time>

11:58:01 ssh.login  {"key1":"yay","key2":100}  ------> CHUNK_A

11:59:13 web.access {"key1":"yay","key2":100}  --|
                                                 |---> CHUNK_B
11:59:30 web.access {"key1":"yay","key2":100}  --|

12:00:01 web.access {"key1":"foo","key2":200}  ------> CHUNK_C

12:00:25 ssh.login  {"key1":"yay","key2":100}  ------> CHUNK_D
```

注意:对于块键的总数没有硬性限制。但是，过多的块键可能会降低I/O性能和/或增加总资源利用率。

### Empty Keys

可以使用[]作为Buffer section参数来指定缓冲区块键为空。

```shell
<match tag.**>
  # ...
  <buffer []>
    # ...
  </buffer>
</match>
```

当输出插件有自己的默认块键并且需要禁用它们时，这特别有用。

### Placeholders(占位符)

当指定了块键时，可以从配置参数值中提取这些值。这取决于插件是否对配置值应用了一个方法(extract_placeholders)。

下面的配置显示了在路径上应用extract_placeholder的文件输出插件:

```shell
# chunk_key: tag
# ${tag} will be replaced with actual tag string
<match log.*>
  @type file
  path /data/${tag}/access.log  #=> "/data/log.map/access.log"
  <buffer tag>
    # ...
  </buffer>
</match>
```

可以使用strptime占位符提取缓冲区块键中的timekey值。提取的时间值是时间键范围的第一秒。

Example:

```shell
# chunk_key: tag and time
# ${tag[1]} will be replaced with 2nd part of tag ("map" of "log.map"), zero-origin index
# %Y, %m, %d, %H, %M, %S: strptime placeholder are available when "time" chunk key specified

<match log.*>
  @type file
  path /data/${tag[1]}/access.%Y-%m-%d.%H%M.log #=> "/data/map/access.2017-02-28.20:48.log"

  <buffer tag,time>
    timekey 1m
  </buffer>
</match>
```

任何键都可以作为块键。如果引用了块键中未指定的键，Fluentd将引发配置错误。

```shell
<match log.*>
  @type file
  path /data/${tag}/access.${key1}.log #=> "/data/log.map/access.yay.log"
  <buffer tag,key1>
    # ...
  </buffer>
</match>
```

### Chunk ID

${chunk_id}将被替换为内部块id。无需在块键中指定chunk_id。

```shell
<match test.**>
  @type file
  path /path/to/app_${tag}_${chunk_id}
  append true
  <buffer tag>
    flush_interval 5s
  </buffer>
</match>
```

结果: **test.foo** tag 如下:

```shell
# 5b35967b2d6c93cb19735b7f7d19100c is chunk id
/path/to/app_test.foo_5b35967b2d6c93cb19735b7f7d19100c.log
```

这个占位符用于标识块，例如secondary_file, s3等。

### Nested Field Support

块键也是一样:

```shell
<match log.*>
  @type file
  path /data/${tag}/access.${$.nest.field}.log #=> "/data/log.map/access.nested_yay.log"
  <buffer tag,$.nest.field> # access record['nest']['field']
    # ...
  </buffer>
</match>
```

### Parameters

#### Argument

它是一个块键数组，必须是逗号分隔的字符串列表。它也可以保持空白。

```shell
<buffer> # blank
  # ...
</buffer>

<buffer tag, time, key1> # keys
  # ...
</buffer>
```

注意:tag和time块键为标签和时间保留，不能用于记录字段。

随time变化，有以下参数:

[Parameters]([Config: Buffer Section - Fluentd](https://docs.fluentd.org/configuration/buffer-section))

#### @type

@type参数指定缓冲区插件的类型。裸输出插件的默认类型是内存，但它可能会被输出插件实现覆盖。

例如，文件输出插件的默认值是file buffer plugin:

```shell
<buffer>
  @type file
  # ...
</buffer>
```

Buffering Parameters:

[Parameters]([Config: Buffer Section - Fluentd](https://docs.fluentd.org/configuration/buffer-section))

## 配置:格式部分

有些Fluentd插件支持<format>节来指定如何格式化记录。

### 格式部分概述

format部分可以在<match>或<filter>部分下。

```shell
<match tag.*>
  @type file
  # ...
  <format>
    # ...
  </format>
</match>
```

**格式化程序插件类型**

<format>的@type参数指定了格式化程序插件的类型。Fluentd核心绑定了一些有用的格式化程序插件。

```shell
<format>
  @type json
</format>
```

以下是内置格式化程序插件的列表:

- out_file
- json
- ltsv
- csv
- megpack
- hash
- single_value

也可以安装和配置第三方插件。

**Parameters**

`@type`

@type参数指定格式化程序插件的类型。

```shell
<format>
  @type csv
  # ...
</format>
```

**Time Parameters**

- time_type
- time_format
- localtime
- utc
- timezone

## 配置:提取部分

### 提取部分概述

提取部分可以在<match>， <source>，或<filter> sections下。对于支持从事件记录中提取值的插件，例如exec，它是启用的。

```shell
<source>
  @type exec
  # ...
  <extract>
    # ...
  </extract>
</source>
```

**提取部分参数**

提取参数

- tag_key
- keep_tag_key
- time_key
- keep_time_key

[Parameters]([Config: Extract Section - Fluentd](https://docs.fluentd.org/configuration/extract-section))

**Time Parameters**

- time_type
- time_format
- localtime
- utc
- timezone

## 配置:注入部分

### 注入部分概述

inject区段可以在<match>或<filter>区段下。对于支持向事件记录注入值的插件，它是启用的。

```shell
<match>
  @type file
  # ...
  <inject>
    # ...
  </inject>
</match>
```

Example

配置:

```shell
<inject>
  time_key fluentd_time
  time_type string
  time_format %Y-%m-%dT%H:%M:%S.%NZ
  tag_key fluentd_tag
</inject>
```

Event Record:

```shell
tag: test
time: 1547575563.952259
record: {"message":"hello"}
```

Injected Record:

```shell
{"message":"hello","fluentd_tag":"test","fluentd_time":"2019-01-15T18:06:03.952259000Z"}
```

**Inject Section Parameter**

Inject Parameters

- hostname_key
- hostname
- worker_id_key
- tag_key
- time_key

[Parameters](https://docs.fluentd.org/configuration/inject-section)

Time Parameters

- time_type
- time_format
- localtime
- utc
- timezone

## 配置:运输部分

一些使用server/http_server插件助手的Fluentd输入、输出和筛选插件也支持<transport>部分来指定如何处理连接。

### 运输部分概述

传输区段必须位于<match>、<source>和<filter>区段之下。它指定传输协议、版本和证书。

```shell
# tcp
<transport tcp>
</transport>

# udp
<transport udp>
</transport>

# tls
<transport tls>
  cert_path /path/to/fluentd.crt
  private_key_path /path/to/fluentd.key
  private_key_passphrase YOUR_PASSPHRASE
  # ...
</transport>
```

Parameters

- protocol [nnum枚举：tcp/udp/tls]
  - Default: tcp

[Config: Transport Section - Fluentd](https://docs.fluentd.org/configuration/transport-section)

**TLS Setting**

- version
- min_version
- max_version
- ciphers
- insecure

如果您想接受多个TLS协议，请使用min_version/max_version而不是version。为了支持旧的样式，fluentd接受TLS1_1和TLSv1_1值。

**Public CA签名参数说明**

对于<transport tls>

- ca_path
- cert_path
- private_key_path
- private_key_passphrase
- client_cert_auth
- cert_verifier

**由私有CA生成并签名参数**

对于<transport tls>

- ca_cert_path
- ca_private_key_path
- ca_private_key_passphrase

**由私有CA证书或自签名参数生成和签名**

- generate_private_key_length
- generate_cert_country
- generate_cert_state
- generate_cert_locality
- generate_cert_common_name
- generate_cert_expiration

**Cert摘要算法参数**

对于<transport tls>

- generate_cert_digest： [enum: `sha1`/`sha256`/`sha384`/`sha512`]



## 配置:存储部分

一些Fluentd插件支持<storage>部分来指定如何处理插件的内部状态。

```shell
<source>
  @type windows_eventlog
  # ...
  <storage>
    # ...
  </storage>
</source>
```

### 存储插件类型

<storage>的@type参数指定了存储插件的类型。Fluentd核心捆绑了一个有用的存储插件。

```shell
<storage>
  @type local
</storage>
```

一些存储插件可能在<storage>区段中有参数:

```shell
<storage awesome_path>
  @type local
</storage>
```

也可以安装和配置第三方插件。

## 配置:服务发现部分

一些Fluentd插件支持<service_discovery>部分来动态设置目标节点。

### 服务发现组概述

service_discovery部分位于<match>下

```shell
<match tag.*>
  @type forward
  # ...
  <service_discovery>
    # ...
  </service_discovery>
</match>
```

**服务发现插件类型**

<service_discovery>小节的@type参数指定插件的类型。Fluentd核心捆绑了一些有用的服务发现插件，例如file。

```shell
<service_discovery>
  @type file
  # ...
</service_discovery>
```

以下是内置的服务发现插件列表:

- static
- file
- srv

**@type**

@type参数指定服务发现插件的类型。

```shell
<service_discovery>
  @type static
  # ...
</service_discovery>
```

## 部署

### System Configuration

[System Configuration - Fluentd](https://docs.fluentd.org/deployment/system-config)

本文描述了Fluentd的<system>部分的系统配置和命令行选项。

系统配置是设置系统范围的配置(如启用RPC、多个工作者等)的一种方法。

## Logging

[Logging - Fluentd](https://docs.fluentd.org/deployment/logging)

本文描述了Fluentd日志机制。

Fluentd有两个日志层:全局日志层和每个插件日志层。可以为全局日志和插件级别日志设置不同的日志级别。

**Log Level**

- fatal
- error
- warn
- info
- debug
- trace

日志级别默认为info, Fluentd默认输出info、warn、error和fatal日志。

## Signals

[Signals - Fluentd](https://docs.fluentd.org/deployment/signals)

本文解释了fluentd如何处理UNIX信号。

**Process Model**

当启动Fluentd时，它将创建两个进程:supervisor和worker。主管进程控制工作进程的生命周期。确保只向主管流程发送信号。

## RPC

HTTP RPC使您能够通过HTTP端点管理Fluentd实例。您可以将此特性用作Unix信号的替代品。

它对于信号不被很好支持的环境特别有用，例如Windows。这需要Fluentd开始时不使用——no-supervisor命令行选项。

## High Availability Config

对于高流量网站，我们建议使用Fluentd的高可用性配置。

请参见分布式日志和容器，以了解部署模式的优缺点。

### 消息传递语义

Fluentd主要是为事件日志传递系统设计的。

在这种系统中，有几种交付保证是可能的:

- 最多一次:消息立即传递。如果传输成功，消息不会再发送出去。然而,许多失败场景会导致消息丢失(例如不再写入能力)。

- 至少一次:每条消息至少传递一次。以失败告终,在这种情况下，消息可能被传递两次。

- 恰好一次:每条消息传递一次且仅一次。这是最令人向往的。

如果系统“不能丢失单个事件”，并且必须传输“恰好一次”，那么系统必须在耗尽写容量时停止摄取事件。正确的方法是使用同步日志记录，并在事件不能被接受时返回错误。

这就是为什么Fluentd提供了“最多一次”和“至少一次”的传输。为了在不影响应用程序性能的情况下收集大量数据，数据记录器必须异步传输数据。这以潜在的交付失败为代价提高了性能。

然而，大多数失败场景是可以避免的。以下部分描述如何设置Fluentd的拓扑以实现高可用性。

### Network Topology

要将Fluentd配置为高可用性，我们假定您的网络由日志转发器和日志聚合器组成。![img](https://gblobscdn.gitbook.com/assets%2F-LR7OsqPORtP86IQxs6E%2F-LR7PDOnAgulIFNQiUNJ%2F-LR7PRYlS49XoqJurybc%2Ffluentd_ha.png?alt=media)

“日志转发器”通常安装在每个节点上以接收本地事件。一旦接收到事件，它们就通过网络将其转发到“日志聚合器”。对于日志转发器，fluent-bit也是轻量级处理的良好候选。

“日志聚合器”是连续地从日志转发器接收事件的守护进程。它们缓冲事件，并定期将数据上传到云中。

Fluentd可以充当日志转发器或日志聚合器，具体取决于其配置。下一节将描述各自的设置。我们假设活动日志聚合器的IP为192.168.0.1，备份的IP为192.168.0.2。

### 日志传送装置配置

使用以下配置配置日志转发器将日志发送到日志聚合器:

```shell
# TCP input
<source>
  @type forward
  port 24224
</source>

# HTTP input
<source>
  @type http
  port 8888
</source>

# Log Forwarding
<match mytag.**>
  @type forward

  # primary host
  <server>
    host 192.168.0.1
    port 24224
  </server>
  # use secondary host
  <server>
    host 192.168.0.2
    port 24224
    standby
  </server>

  # use longer flush_interval to reduce CPU usage.
  # note that this is a trade-off against latency.
  <buffer>
    flush_interval 60s
  </buffer>
</match>
```

当活动聚合器(192.168.0.1)失效时，日志将被发送到备份聚合器(192.168.0.2)。如果两个服务器都死了，日志将在相应的转发器节点的磁盘上进行缓冲。

### 日志聚合器配置(Log Aggregator Configuration)

日志汇聚器使用以下配置将日志传输的输入源配置为TCP:

```shell
# Input
<source>
  @type forward
  port 24224
</source>

# Output
<match mytag.**>
  # ...
</match>
```

对传入的日志进行缓冲，然后定期上传到云。如果上传失败，日志将保存在本地磁盘上，直到重传成功。

**故障情况下**

当日志转发器从应用程序接收到事件时，事件首先被写入磁盘缓冲区(由<buffer>'s path指定)。在每个flush_interval之后，缓冲的数据被转发到聚合器。

这个过程对于数据丢失具有内在的健壮性。如果日志转发器的fluentd进程终止，那么在其重新启动时，缓冲的数据将被正确地传输到其聚合器。如果转发器和聚合器之间的网络中断，则自动重试数据传输。

然而，可能的消息丢失场景确实存在:

- 进程在接收到事件后立即终止，但在写入之前终止他们进入缓冲区。

- 转发器的磁盘损坏，文件缓冲区丢失。

**聚合器故障**

当日志聚合器从日志转发器接收事件时，事件首先被写入磁盘缓冲区(由<buffer>'s path指定)。在每个flush_interval之后，将缓冲的数据上传到云。



这个过程对于数据丢失具有内在的健壮性。如果日志聚合器的fluentd进程终止，那么在其重新启动时，将正确地重新传输来自日志转发器的数据。如果聚合器和云之间的网络中断，数据传输将自动重试。



然而，可能的消息丢失场景确实存在:

- 进程在接收到事件后立即终止，但在写入之前终止他们进入缓冲区。

- 聚合器的磁盘损坏，文件缓冲区丢失。

### Troubleshooting

“没有可用节点”

请确保您不仅可以使用TCP，还可以使用UDP与端口24224进行通信。这些命令对检查网络配置很有用:

```shell
$ telnet host 24224
$ nmap -p 24224 -sU host
```

请注意，有一个已知的问题，VMware偶尔会丢失用于心跳的小UDP包。

### Performance Tuning

[Performance Tuning - Fluentd](https://docs.fluentd.org/deployment/performance-tuning-single-process)

本文介绍如何在单个进程中优化Fluentd性能。如果您的流量达到每秒钟5,000条消息，以下技术应该足够了。

随着流量的增加，Fluentd倾向于更多的CPU限制。在这种情况下，考虑使用多工作者特性。

### Multi Process Workers

本文描述了如何使用Fluentd的多进程工作者特性来实现高流量。这个特性启动两个或更多的熟练工人来利用多个CPU能力。这个特性可以简单地替代fluent-plugin-multiprocess。

### Failure Scenarios

本文描述了各种Fluentd故障场景。我们将假设您已经为高可用性配置了Fluentd，以便每个应用程序节点都有其本地转发器，并且所有日志都被聚合到多个聚合器中。

### Plugin Management

本文解释了如何管理Fluentd插件，包括添加第三方插件。

fluent-gem

fluent-gem命令用于安装Fluentd插件。这是gem命令的包装器。

Example:

```shell
fluent-gem install fluent-plugin-grep
```

Ruby不保证其主要版本之间的C扩展API兼容性。如果你更新了Fluentd的Ruby版本，你应该重新安装依赖于C扩展的插件。

### Trouble Shooting

Look at Logs

如果事情没有像预期的那样发生，请首先查看您的日志。对于td-agent (rpm/deb)，日志位于“/var/log/td-agent/td-agent.log”。

### Fluentd UI

fluentd-ui是基于浏览器的fluentd和td-agent管理器，支持以下操作:

- 安装、卸载和升级Fluentd插件
- 启动/停止/启动fluentd过程

- 配置Fluentd设置，如配置文件、pid文件路径等。

- 使用简单的错误查看器查看Fluentd日志

fluentd-ui不适用于fluentd v1，而td-agent 3和4不包含它。这个内容目前是针对v0.12的。

### Linux Capability

本文展示了在Fluentd核心上启用Linux功能模块的配置和相关的gem安装说明。

### Command Line Option

本文描述了fluentd项目中的命令行工具及其选项。

## Input Plugins

Fluentd有八(8)种插件:

- input
- parser
- filter
- output
- formatter
- storage
- service discovery
- buffer



### Overview

输入插件扩展Fluentd以从外部源检索和提取事件日志。输入插件通常创建线程、套接字和侦听套接字。还可以编写它以定期地从数据源提取数据。

输入插件列表:

- in_tail
- in_forward
- in_udp
- in_tcp
- in_unix
- in_http
- in_syslog
- in_exec
- in_sample
- in_windows_eventlog

**其他输入插件**

请参考以下可用插件列表，以了解其他输入插件:

Fluentd插件

### tail

![img](https://gblobscdn.gitbook.com/assets%2F-LR7OsqPORtP86IQxs6E%2F-LWyDIvT7Z7M06Hdqznf%2F-LWNPOu-hh314qQErrMR%2Ftail.png?alt=media)

`in_tail` Input插件允许Fluentd从文本文件的尾部读取事件。它的行为类似于`tail -F`命令。

它包含在Fluentd的核心中。

**示例配置**

```shell
<source>
  @type tail
  path /var/log/httpd-access.log
  pos_file /var/log/td-agent/httpd-access.log.pos
  tag apache.access
  <parse>
    @type apache2
  </parse>
</source>
```

关于[configure](https://docs.fluentd.org/configuration/config-file)的基本结构和语法，请参阅配置文件文章。有关<parse>，请参见[parse](https://docs.fluentd.org/configuration/parse-section)部分。

**它是如何工作的**

当Fluentd首次配置in_tail时，它将从该日志的尾部开始读取，而不是从开头。循环日志后，Fluentd开始从头读取新文件。它跟踪当前inode号。

如果td-agent重启，它将从重启前的最后一个位置继续读取。这个位置记录在pos_file参数指定的位置文件中。

**Linux Capability**

从v1.12.0开始，如果Fluentd的Linux能力处理模块被启用，in_tail将处理以下Linux能力:

`CAP_DAC_READ_SEARCH` (`:dac_read_search` on `in_tail` code.)

`CAP_DAC_OVERRIDE` (`:dac_override` on `in_tail` code.)

Plugin Helpers:

- timer
- event_loop
- parser
- compat_parameters

[tail - Fluentd](https://docs.fluentd.org/input/tail)

## Output Plugins

### Overview

Fluentd v1.0输出插件有三(3)种缓冲和刷新模式:

- **非缓冲模式**不缓冲数据并立即输出结果。
- **同步缓冲模式**有“暂存的”缓冲块(块是一个事件集合)和块队列，其行为可以是由<buffer>部分控制(见下图)。
- **异步缓冲模式**也有“阶段”和“队列”，但是输出插件不会在方法中提交写入块同步，但稍后提交。![img](https://gblobscdn.gitbook.com/assets%2F-LR7OsqPORtP86IQxs6E%2F-LeVWL3walFMKmir5umf%2F-LeVWO3_U0_H75zRg1PM%2Ffluentd-v0.14-plugin-api-overview.png?alt=media)

输出插件可以支持所有模式，但可能只支持其中一种模式。如果配置中没有<buffer>节，Fluentd会自动选择适当的模式。如果用户为不支持缓冲的输出插件指定<buffer>部分，Fluentd将引发配置错误。

v1中的输出插件可以通过配置动态地控制缓冲区分块的键。用户可以将缓冲块键配置为时间(用户指定的任何单位)、标签和记录的任何键名。输出插件将事件拆分为块:块中的事件具有相同的块键值。输出插件的缓冲区行为(如果有的话)是由一个单独的buffer插件定义的。可以为每个输出插件选择不同的缓冲区插件。

### 输出插件列表

- out_copy
- out_null
- out_roundrobin
- out_stdout
- out_exec_filter
- out_forward
- out_mongo/out_mongo_replset
- out_exec
- out_file
- out_s3
- out_webhdfs

其他插件

查看可用插件列表以了解更多关于其他输出插件的信息:

http://fluentd.org/plugin/