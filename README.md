# FastKV
[![Maven Central](https://img.shields.io/maven-central/v/io.github.billywei01/fastkv-java)](https://search.maven.org/artifact/io.github.billywei01/fastkv-java)


## 1. 概述
FastKV是用Java编写的高效可靠的key-value存储库。<br>

备注：由于提供给Android平台的版本和纯JDK的版本的差异越来越多，所以分开仓库来维护。<br>
本仓库为纯JDK版本，Android的版本在：https://github.com/BillyWei01/FastKV
<br>

FastKV有以下特点：
1. 读写速度快
    - 二进制编码，编码后的体积相对XML等文本编码要小很多；
    - 增量编码：FastKV记录了各个key-value相对文件的偏移量，
      从而在更新数据时可以直接在指定的位置写入数据。
    - 默认用mmap的方式记录数据，更新数据时直接写入到内存即可，没有IO阻塞。
    - 超长字符串等做特殊处理，单独另起文件写入，不影响主文件的加载和更新。
2. 支持多种写入模式
   - 除了mmap这种非阻塞的写入方式，FastKV也支持常规的阻塞式写入方式，
     并且支持同步阻塞和异步阻塞（分别类似于SharePreferences的commit和apply)。
3. 支持多种类型
   - 支持常用的boolean/int/float/long/double/String等基础类型。
   - 支持ByteArray (byte[])。
   - 支持存储自定义对象。
   - 内置Set<String>的编码器 (兼容SharePreferences)。
4. 方便易用
   - FastKV提供了了丰富的API接口，开箱即用。
   - 提供的接口其中包括getAll()和putAll()方法，
     所以很方便迁移SharePreferences等框架的数据到FastKV, 当然，迁移FastKV的数据到其他框架也很简单。
5. 稳定可靠
   - 通过double-write等方法确保数据的完整性。
   - 在API抛IO异常时自动降级处理。
6. 代码精简
   - FastKV由纯Java实现，编译成jar包后体积只有数十K。

相对Android版本，JDK版本不支持多进程和内容加密。<br>
多进程一般用在Android客户端主进程和服务进程之间的公共存储，所以纯JDK版本就没这个环境了；
内容加密通常也是客户端的需求, 所以JDK版本暂时不加这个功能了，<br>
毕竟增加功能的同时会增加代码复杂度，如果后面确实需要再补充。

   
## 2. 使用方法

### 2.1 导入

```gradle
dependencies {
    implementation 'io.github.billywei01:fastkv-java:1.2.2'
}
```

### 2.2 初始化
```kotlin
    FastKVConfig.setLogger(FastKVLogger)
    FastKVConfig.setExecutor(Dispatchers.Default.asExecutor())
```
初始化可以按需设置日志接口和Executor。


### 2.3 数据读写
- 基本用法
```java
    FastKV kv = new FastKV.Builder(path, name).build();
    
    if(!kv.getBoolean("flag")){
        kv.putBoolean("flag" , true);
    }
    
    int count = kv.getInt("count");
    if(count < 10){
        kv.putInt("count" , count + 1);
    }
```

- 存储自定义对象

```java
    FastKV.Encoder<?>[] encoders = new FastKV.Encoder[]{LongListEncoder.INSTANCE};
    FastKV kv = new FastKV.Builder(path, name).encoder(encoders).build();
        
    List<Long> list = new ArrayList<>();
    list.add(100L);
    list.add(200L);
    list.add(300L);
    kv.putObject("long_list", list, LongListEncoder.INSTANCE);
    
    List<Long> list2 = kv.getObject("long_list");
```

除了支持基本类型外，FastKV还支持写入对象，只需在构建FastKV实例时传入对象的编码器即可。
编码器为实现FastKV.Encoder的对象。
比如上面的LongListEncoder的实现如下：

```java
public class LongListEncoder implements FastKV.Encoder<List<Long>> {
    public static final LongListEncoder INSTANCE = new LongListEncoder();

    @Override
    public String tag() {
        return "LongList";
    }

    @Override
    public byte[] encode(List<Long> obj) {
        return new PackEncoder().putLongList(0, obj).getBytes();
    }

    @Override
    public List<Long> decode(byte[] bytes, int offset, int length) {
        return PackDecoder.newInstance(bytes, offset, length).getLongList(0); 
    }
}
```

编码对象涉及序列化/反序列化。<br/>
这里推荐笔者的另外一个框架：https://github.com/BillyWei01/Packable


## 3. 性能测试
- 测试数据：搜集APP中的SharePreferences汇总的部份key-value数据（经过随机混淆）得到总共四百多个key-value。<br>
          由于日常使用过程中部分key-value访问多，部分访问少，所以构造了一个正态分布的访问序列。<br>
          此过程循环多遍，分别求其总耗时。
- 比较对象： SharePreferences/DataStore/MMKV。
- 测试机型：荣耀20S。

测试代码：[Benchmark](https://github.com/BillyWei01/FastKV/blob/main/app/src/main/java/io/fastkv/fastkvdemo/Benchmark.kt)

测试结果：

| | 写入(ms) |读取(ms) 
---|---|---
SharePreferences | 1182 | 2
DataStore | 33277 | 2
MMKV | 29 | 10
FastKV  | 19 | 1 

- SharePreferences提交用的是apply, 耗时依然不少。
- DataStore写入很慢。
- MMKV的读取比SharePreferences/DataStore要慢，写入则比之快不少。
- FastKV无论读取还是写入都比其他方式要快。

现实中通常情况下不会有此差距，因为一般而言不会几百个key-value写到同一个文件，此处仅为显示极端情况下的对比，读者可自行调整参数看对比数据。

## 4. 参考链接
https://juejin.cn/post/7018522454171582500

## License
See the [LICENSE](LICENSE) file for license rights and limitations.



