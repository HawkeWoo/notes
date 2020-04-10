# Maxcompute 实战

[TOC]

## 安装客户端

1. 下载[Maxcompute](http://docs-aliyun.cn-hangzhou.oss.aliyun-inc.com/assets/attach/119118/cn_zh/1559120057202/odpscmd_public_May.zip?spm=a2c4g.11186623.2.17.1a185c235KBu1Z&file=odpscmd_public_May.zip)客户端

2. 解压下载的安装文件，得到**bin**、**conf**、**lib**、**plugins**四个文件夹。

3. 编辑客户端配置文件([endpoint配置](https://help.aliyun.com/document_detail/34951.html?spm=a2c4g.11186623.6.585.71345c23PZl3J8))

   ```ini
   $ cat ./conf/odps_config.ini
   project_name=<project name>
   access_id=<your access id>
   access_key=<your access key>
   
   # end_point=http://service.odps.aliyun.com/api
   # 根据实际情况定
   end_point=http://service.cn-hongkong.maxcompute.aliyun.com/api
   log_view_host=http://logview.odps.aliyun.com
   https_check=true
   # confirm threshold for query input size(unit: GB)
   data_size_confirm=100.0
   # this url is for odpscmd update
   update_url=http://repo.aliyun.com/odpscmd
   # download sql results by instance tunnel
   use_instance_tunnel=true
   # the max records when download sql results by instance tunnel
   instance_tunnel_max_record=10000
   # IMPORTANT:
   # If leaving tunnel_endpoint untouched, console will try to automatically get one from odps service, which might charge networking fees in some cases.
   # Please refer to https://help.aliyun.com/document_detail/34951.html
   # 根据实际情况定
   tunnel_endpoint=http://dt.cn-hongkong.maxcompute.aliyun.com
   ```

4. 执行bin下的odpscmd即可。

5. 为了更方便地使用maxcompute客户端，可以把odps添加到环境变量中。

   ```shell
   # 我用的是zsh
   $ vim ~/.zshrc
   ```

   ```shell
   # 在最后添加
   export ODPS_HOME=/Your Dir/odpscmd_public_May
   export PATH=$PATH:$ODPS_HOME/bin
   ```

   ```shell
   # 然后执行下面命令使其生效
   $ source ~/.zshrc
   # 执行命令行就可以执行maxcompute客户端
   $ odpscmd
   ```

## 建表

进入DataStudio, 新建业务流程，在这里我们新建了`market集成`。

选择`market集成`这一业务流程，在表选项新建我们所需要的数据表，如下（通过DDL模式填写建表语句，然后提交到生产环境即可）：

### ODS层

event表

```sql
CREATE TABLE IF NOT EXISTS event
(
	iuid STRING COMMENT 'Aqumon Instrument Code',
	event_type INT COMMENT 'Event type, example: splitting, divident, etc.',
  value DECIMAL COMMENT 'Event value',
	ex_time BIGINT COMMENT 'Execution time',
	pay_time BIGINT COMMENT 'Pay time',
  extra_data STRING COMMENT 'Extra data'
)PARTITIONED BY (dt STRING);
```

historical_data表

```sql
CREATE TABLE IF NOT EXISTS historical_data
(
  iuid STRING COMMENT 'Aqumon Instrument Code',
  value DECIMAL COMMENT 'Quote value',
  ts BIGINT COMMENT 'Timestamp in ms'
)PARTITIONED BY (dt STRING, region STRING);
```

### CDM层

dw_historical_data表

```sql
CREATE TABLE IF NOT EXISTS dw_historical_data
(
  iuid STRING COMMENT 'Aqumon Instrument Code',
  original_value DECIMAL COMMENT 'Original Quote value',
  forward_value DECIMAL COMMENT 'Forward Quote value',
  backward_value DECIMAL COMMENT 'Backward Quote value',
  ts BIGINT COMMENT 'Timestamp in ms'
)PARTITIONED BY (dt STRING, region STRING);
```

### ADS层

backtesting表

```sql
CREATE TABLE IF NOT EXISTS backtesting
(
	portfolio_id BIGINT COMMENT 'Portfolio id',
  composition STRING COMMENT 'Portfolio composition, iuid and weight'
  date STRING COMMENT 'Date',
  value DECIMAL COMMENT 'Backtesting value'
);
```

## 数据集成

