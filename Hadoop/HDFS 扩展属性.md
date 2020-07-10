# HDFS的扩展属性
## 概述
扩展属性（Extended attribute）允许对文件添加与之相关的额外的元数据。和系统层级的元数据（文件访问权限，修改时间等）不同，扩展属性是被应用添加存储的，可以是文件的额外描述。HDFS的扩展属性，是引申自linux文件系统的扩展属性，是一种name-value的存储结构，name是string类型，value是binary。在HDFS中，扩展属性的name需要用namespace来做前缀，例如"user.encodingType"，一个文件可以有多个扩展属性。HDFS的扩展属性是存储在namenode中，并且随着文件的删除被移除。</br>

## Namespace和permission
上面我们说到，扩展属性的name必须是以namespace作为前缀，在HDFS中有五个可用的namespace：user, trusted, system, security, raw。不同的namespace有不同的权限准则。
1. user: 扩展属性的权限和文件的权限相同
2. trusted: HDFS超级用户使用
3. system: HDFS内部保留使用，用来实现HDFS的功能
4. security: HDFS内部保留使用, 有一个特殊用法，“security.hdfs.unreadable.by.superuser”属性，可以组织超级用户读取文件内容，这个属性被设置之后就不能被移除，并且不设置value只。
5. raw: 内部系统需要暴露的的一些属性使用

## 扩展属性的设置读取
### Hadoop shell
+ getfattr
```
hadoop fs -getfattr [-R] -n name | -d [-e en] <path>
```
| | |
|--|--|
|-R|递归列出所有文件文件夹的属性|
|-n name|获取对应name属性的value|
|-d|获取改path下的所有扩展属性|
|-e|value编码设定|
|path|文件或文件夹|

+ setfattr
```
hadoop fs -setfattr -n name [-v value] | -x name <path>
```
| | |
|--|--|
|-n name|扩展属性name|
|-v value|扩展属性value|
|-x name|删除扩展属性|
|path|文件或文件夹|

### Filesystem API
+ getXattr

| return type|method |
|--|--|
|byte[]	|getXAttr(Path path, String name)</br>Get an xattr name and value for a file or directory.|
|Map\<String,byte[]\>|	getXAttrs(Path path)</br>Get all of the xattr name/value pairs for a file or directory.|
|Map\<String,byte[]\>|	getXAttrs(Path path, List<String> names)</br>Get all of the xattrs name/value pairs for a file or directory.|

+ setXattr

| return type|method |
|--|--|
|void|setXAttr(Path path, String name, byte[] value)</br>Set an xattr of a file or directory.|
|void|setXAttr(Path path, String name, byte[] value, EnumSet<XAttrSetFlag> flag)</br>Set an xattr of a file or directory.|

``` scala 
    val xattrs = fileSystem.getXAttrs(path)
    if (xattrs.containsKey("user.name")) {
        fileSystem.setXAttr(path, "user.name",
        "first".getBytes(),
        util.EnumSet.of(XAttrSetFlag.REPLACE))
    } else {
        fileSystem.setXAttr(path, "user.name",
        "secondary".getBytes(),
        util.EnumSet.of(XAttrSetFlag.CREATE))
    }
```

### WebHdfs

| | |
|---|---|
|Set XAttr|curl -i -X PUT "http://\<HOST\>:\<PORT\>/webhdfs/v1/\<PATH\>?op=SETXATTR&xattr.name=\<XATTRNAME\>&xattr.value=\<XATTRVALUE\>&flag=\<FLAG\>"|
|Remove XAttr|curl -i -X PUT "http://\<HOST\>:\<PORT\>/webhdfs/v1/\<PATH\>?op=REMOVEXATTR&xattr.name=\<XATTRNAME\>"|
|Get an XAttr|curl -i "http://\<HOST\>:\<PORT\>/webhdfs/v1/\<PATH\>?op=GETXATTRS&xattr.name=\<XATTRNAME\>&encoding=\<ENCODING\>"|
|Get multiple XAttrs|curl -i "http://\<HOST\>:\<PORT\>/webhdfs/v1/\<PATH\>?op=GETXATTRS&xattr.name=\<XATTRNAME1\>&xattr.name=\<XATTRNAME2\>&encoding=\<ENCODING\>"|