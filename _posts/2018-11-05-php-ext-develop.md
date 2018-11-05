---
layout:     post
title:      "PHP扩展开发"
subtitle:   " \"PHP扩展开发\""
date:       2018-11-05 20:01:00
author:     "ZhuDong"
catalog: true
tags:
    - 技术
---

### 下载PHP源码，生成扩展框架

1、cd ~/php-7.0.9/ext

2、./ext_skel --extname=testext

>Creating directory testext

>Creating basic files: config.m4 config.w32 .gitignore testext.c php_testext.h CREDITS EXPERIMENTAL tests/001.phpt testext.php [done].

>To use your new extension, you will have to execute the following steps:

>	1.  $ cd ..
>	2.  $ vi ext/testext/config.m4
>	3.  $ ./buildconf
>	4.  $ ./configure --[with|enable]-testext
>	5.  $ make
>	6.  $ ./sapi/cli/php -f ext/testext/testext.php
>	7.  $ vi ext/testext/testext.c
>	8.  $ make

>Repeat steps 3-6 until you are satisfied with ext/testext/config.m4 and
step 6 confirms that your module is compiled into PHP. Then, start writing
code and repeat the last two steps as often as necessary.

输出如上，说明创建代码骨架成功。而且告知了接下来的操作步骤。

1）进入到 ext/testext 目录

2）修改 config.m4 文件

3）执行./buildconf

4)  执行 ./configure --enable-testext

5）make 

6)  测试是否编译成功

7）进入ext/testext/testext.c 文件，进行开发。

8）开发完了，就进行重新make

重复7、8步骤直到扩展满足自己的需求。

--

### 实例

扩展作用：将Content-Type为application/json的POST请求的json格式Body数据解析为数组形式，并写入到 \$\_POST中。

分析：
听过上篇文章对 \$\_POST的分析，我们知道，对于application/www-xxx-urlencoded类型的普通请求， application/json类型的请求和它的不同在于没有注册post\_entry， 并且没有合适的post_handler将JSON数据解析成数组，并写入到PG(http\_globals)[0] 中。 由此，我们可以想到，可以在MINIT阶段构造一个application/json类型的post\_entry注册到PHP已知的content\_types中去，同时自定义post\_handler。在post\_hander中解析json数据为数组，并写入到PG(http\_globals)[0] 中，同时注册到EG(symbol\_table)， 这样我们应该就可以通过 \$\_POST获取到JOSN类型的k-v数据了。


开发：按照步骤1~6， 进入步骤7进行开发。我们定义我们的扩展叫mjson。 mjson.c文件主要修改的是MINIT方法，有几个关键的操作如下：

1）定义post\_entry 和 post\_handler 处理函数

```
SAPI_API SAPI_POST_HANDLER_FUNC(php_std_mjson_handler);
static sapi_post_entry php_mjson_entries[] = {
	{ DEFAULT_JSON_CONTENT_TYPE, sizeof(DEFAULT_JSON_CONTENT_TYPE)-1, sapi_read_standard_form_data,	php_std_mjson_handler },
	{ NULL, 0, NULL, NULL }
};
```

2) 在MINIT方法中注册post\_entry

```
PHP_MINIT_FUNCTION(mjson)
{
	/* If you have INI entries, uncomment these lines
	REGISTER_INI_ENTRIES();
	*/
	sapi_register_post_entries(php_mjson_entries);
	return SUCCESS;
}
```

3) 实现php\_std\_mjson\_handler 方法：

```
SAPI_API SAPI_POST_HANDLER_FUNC(php_std_mjson_handler) {
	zend_long maxlen = (ssize_t) PHP_STREAM_COPY_ALL;
	zend_string *result;
	php_stream *src;
	src = SG(request_info).request_body;
	result = php_stream_copy_to_mem(src, maxlen, 0);
	php_json_decode_ex(&PG(http_globals)[TRACK_VARS_POST],ZSTR_VAL(result),ZSTR_LEN(result),PHP_JSON_OBJECT_AS_ARRAY,512);
}
```

至此， 这个扩展的功能就实现了。我们再次make & make install

最后修改php.ini 文件，将mjson.so 扩展载入，重启FPM，编写一个测试脚本如下：

```
<?php
var_dump($_POST);
var_dump(file_get_contents("php://input"));
```
最后发送一个curl请求，请求以上test.php。 \$\_POST中包含了json数据解析而成的数组。

curl "http://127.0.0.1:8080/mis/test.php" -d '{"test":1}' -H "Content-Type:application/json"

返回
```PHP
array(1) {
  ["test"]=>
  int(1)
}
string(10) "{"test":1}"
```