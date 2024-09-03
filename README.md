

# 一、问题概述


客户的生产环境突然在近期间歇式的收到了Kafka CRC的相关异常，异常内容如下



```
Record batch for partition skywalking-traces-0 at offset 292107075 is invalid, cause: Record is corrupt (stored crc = 1016021496, compute crc = 1981017560)JAVA 复制 全屏
```

报错没有规律性，有可能半天都不出现一次，也有可能一小时出现2、3次


而这个报错会导致Kafka的Consumer hang死，即无法继续消费后续的消息，只能手动重启Consumer才能继续，是非常严厉的报错，导致生产不可用



简单解释一下这个异常的原因，Kafka会在每个Batch的header中存储持久化下来的消息体的CRC，所谓CRC可以简单理解为TCP的checkSum，而后Consumer在收到这条消息后，将Batch header中存储的CRC取出来，然后再根据统一的CRC算法计算收到的消息体的CRC值，而后拿上这两个值做一下比对，如果一样，说明消息没有被篡改，如果不一样，就会扔出CRC异常



由于是公司的重保客户，又出现如此严重的惊天bug，一时间公司的高层领导均高度重视此事，每天晚上的夕会（均是领导）也都要单独询问解决进度，一时间，压力山大


# 二、分析定位


## 2\.1、问题分析


在公司另外一个项目中（简称A客户，我们当前要处理的简称B客户），这个报错并不陌生，且报错的内容一模一样，那这两者会不会是同一个bug呢？答案是否定的


* **A客户**：确实会报CRC异常，但稳定复现，只要出现了，不论Consumer重启多少次，也不论多少个Consumer来消费，均是会报CRC异常（最终定位是A客户的磁盘出现的坏点，导致数据异常，更换磁盘后，问题消失）
* **B客户**：问题只会出现一次，如果手动重启Consumer，该问题便会消失，如果换个Consumer来消费，问题也会消失，好像它只是昙花一现，且再也无法复现，如同鬼魅


那这个问题不是常规问题，就比较棘手了


## 2\.2、Kafka自身bug ？


笔者长期活跃在Kafka社区，是Kafka社区的Contributer，且在阿里的公有云工作多年，但这个报错在之前却是一次没有遇到过


在我印象中，CRC错误已经好久没见到了，而后我排查了社区所有CRC有关的issue list，发现只有在0\.10之前的低版本中出现过类似异常，而我们的B客户生产环境使用的是2\.8\.2，这是整个2版本中的最后一个版本，而Kafka 2\.0以后的版本却很少出现这个异常


而Kafka自身计算CRC的逻辑却非常简洁，只有几十行java代码，综上，笔者认为Kafka自身出bug的概率相对较小，不能把重要精力放在这块


## 2\.3、报错倾向分析


接下来我们便对这个异常的报错倾向开始分析，经过各种排查，它具备以下特质：


* 间歇性、概率性、不确定性；有时候可能半天都不出现一次，也有可能一小时出现2、3次
* 无法复现；出现问题的消息，重新消费，CRC异常会消失
* B客户现场有8套环境，现只有一套环境存在此问题，对比其他7套环境，粗略看在磁盘型号、CPU型号等硬件设备是保持一致的
* 查看出问题的Broker，其不具备统一性，也没有落在同一台宿主机上
* 怀疑是内存条错误，排查对应的日志没有发现异常信息
* 与操作系统的同学确认，咱们现在虽然使用的是我们自研的OS，但TCP协议这块却没有改动过，使用的是Linux原生的
* 在家里的环境模拟网络丢包、错包的场景，没有复现CRC异常


 


至此，好像均一切正常，粗略排查后，没有发现有价值的线索。因此我们便将重心转移至“网络”\+“磁盘”这两块看似不太可能出现问题的基础能力上


# 三、网络排查


## 3\.1、埋点


进行网络排查的话，我们就要执行TCP抓包，即执行tcpdump命令，但抓包也不是简单在机器上执行一个命令那么简单，当前的部署环境是3台Broker、3台Consumer，异常会暴露在Consumer中，但单台Consumer可能会与多台Broker建连


![](https://img2024.cnblogs.com/blog/2109301/202409/2109301-20240902180357118-1782858057.png)


因此抓包的思路是在3台Broker上均创建监听，且在某一台consumer也创建监听，而后观察TCP发送出去的数据与接收到的数据是否一致


首先在3台Broker上执行dump命令，由于出现问题的概率不高，可能导致网络dump包非常大，因此在命令中将TCP包拆成500M大小的文件，且只保留最近的5个文件，3台Broker的IP如下：


* 10\.0\.0\.70
* 10\.0\.0\.71
* 10\.0\.0\.72


而3台Consumer的IP为：


* 10\.0\.0\.18
* 10\.0\.0\.19
* 10\.0\.0\.20


 


首先在1台consumer上执行如下命令：



```
sudo nuhup tcpdump -i vethfe2axxxx -nve host 10.0.0.70 or host 10.0.0.71 or host 10.0.0.72 -w broker_18.pcap -C 500 -W 5 -Z ccadmin &
sudo nuhup tcpdump -i vethfe2axxxx -nve host 10.0.0.70 or host 10.0.0.71 or host 10.0.0.72 -w broker_19.pcap -C 500 -W 5 -Z ccadmin &
sudo nuhup tcpdump -i vethfe2axxxx -nve host 10.0.0.70 or host 10.0.0.71 or host 10.0.0.72 -w broker_20.pcap -C 500 -W 5 -Z ccadmin &
```

其次在3台Broker上分别执行如下：



```
sudo nuhup tcpdump -i xxxxxxxxx1 -nve host 10.0.0.18 -w broker_18.pcap -C 500 -W 5 -Z ccadmin &
sudo nuhup tcpdump -i xxxxxxxxx2 -nve host 10.0.0.19 -w broker_19.pcap -C 500 -W 5 -Z ccadmin &
sudo nuhup tcpdump -i xxxxxxxxx3 -nve host 10.0.0.20 -w broker_20.pcap -C 500 -W 5 -Z ccadmin &
```

简单解释一下命令含义：


* \-i vethfe2axxxx 后面添加的是网卡名称
* \-nve host 需要添加的是需要监听的IP
* \-w broker\_18\.pcap 会将记录生成到broker\_18\.pcap的文件中
* \-C 500 每隔500M生成一个文件
* \-W 5 保留最近生成的5个文件
* \-Z ccadmin 指定用户名


## 3\.2、异常浮现


而后我们在屏幕前等待了漫长的近2个小时后，终于捕获到了一条异常信息，因此马不停蹄地将数据下载下来，进行对照分析。首先看到的便是`Wireshark`帮助解析生成的前后报文


![](https://img2024.cnblogs.com/blog/2109301/202409/2109301-20240902180426195-1044873775.png)


报文总长度为2712个字节，约3K的数据，但我们惊讶地发现，其中有15个字节的数据出现了偏差


![](https://img2024.cnblogs.com/blog/2109301/202409/2109301-20240902180452720-377895502.png)


至此，定位可能就是网络传输出现了问题，不过我们还需要做更多的校验工作。`Wireshark`工具虽然好用，但却不是万能的，它只能解析Kafka的传输协议，无法对内容进行校验与还原


 


（我们在样本中随机取了3个正常交互的消息进行对比，发现在网络发送前与发送后的数据完全一致）


## 3\.3、Kafka协议解析


Kafka的协议分为传输协议及存储协议，笔者对它们相对比较熟悉，因此开始直接对前后传输的报文进行了解析，首先是Request



```
import org.apache.kafka.common.requests.FetchRequest;
import org.apache.kafka.common.requests.RequestHeader;

import javax.xml.bind.DatatypeConverter;
import java.nio.ByteBuffer;

public class MyTest {
    public static void main(String[] args) {

        // 08-30 15:06:40.820028
        String hexString =
            "0e 1f 48 2d 7e 32 06 82 25 e9 a3 d9 08 00 45 00 " +
                "00 ab 3f b3 40 00 40 06 35 ed 62 02 00 55 62 02 " +
                "00 54 eb 2e 23 85 68 57 32 37 b5 08 3f b2 80 18 " +
                "7d 2c c5 4a 00 00 01 01 08 0a b7 d1 e6 5c 26 30 " +
                "b9 11 00 00 00 73 00 01 00 0c 11 85 6f bf 00 15 " +
                "62 72 6f 6b 65 72 2d 31 30 30 32 2d 66 65 74 63 " +
                "68 65 72 2d 30 00 00 00 03 ea 00 00 01 f4 00 00 " +
                "00 01 00 a0 00 00 00 72 52 11 e0 11 85 6f bf 02 " +
                "13 5f 5f 63 6f 6e 73 75 6d 65 72 5f 6f 66 66 73 " +
                "65 74 73 02 00 00 00 15 00 00 00 03 00 00 00 00 " +
                "13 36 67 3a 00 00 00 03 00 00 00 00 00 00 00 00 " +
                "00 10 00 00 00 00 01 01 00";
        // FetchRequestData(clusterId=null, replicaId=1002, maxWaitMs=500, minBytes=1, maxBytes=10485760,
        // isolationLevel=0, sessionId=1917981152, sessionEpoch=293957567,
        // topics=[FetchTopic(topic='__consumer_offsets', partitions=[FetchPartition(partition=21, currentLeaderEpoch=3,
        // fetchOffset=322332474, lastFetchedEpoch=3, logStartOffset=0, partitionMaxBytes=1048576)])], forgottenTopicsData=[], rackId='')



        StringBuilder sb = new StringBuilder();
        char[] charArray = hexString.toCharArray();
        for (char c : charArray) {
            if (c != ' ') {
                sb.append(c);
            }
        }
        hexString = sb.toString();
        byte[] byteArray = DatatypeConverter.parseHexBinary(hexString);
        ByteBuffer byteBuffer = ByteBuffer.wrap(byteArray);
        System.out.println("len is : " + byteBuffer.array().length);
        byteBuffer.get(new byte[66]);
        byteBuffer.getInt();

        RequestHeader header = RequestHeader.parse(byteBuffer);
        System.out.println(header);
        FetchRequest fetchRequest = FetchRequest.parse(byteBuffer, (short) 12);
        System.out.println(fetchRequest);
    }
}
```

最终输出如下



```
len is : 185
RequestHeader(apiKey=FETCH, apiVersion=12, clientId=broker-1002-fetcher-0, correlationId=293957567)
FetchRequestData(clusterId=null, replicaId=1002, maxWaitMs=500, minBytes=1, maxBytes=10485760, isolationLevel=0, sessionId=1917981152, sessionEpoch=293957567, topics=[FetchTopic(topic='__consumer_offsets', partitions=[FetchPartition(partition=21, currentLeaderEpoch=3, fetchOffset=322332474, lastFetchedEpoch=3, logStartOffset=0, partitionMaxBytes=1048576)])], forgottenTopicsData=[], rackId='')
```

且TCP传输前后的内容一致，因此问题没有出现在Request上


 


接下来就需要进行Response的二进制协议解析，代码如下



```
import org.apache.kafka.common.TopicPartition;
import org.apache.kafka.common.record.MemoryRecords;
import org.apache.kafka.common.requests.FetchResponse;
import org.apache.kafka.common.requests.ResponseHeader;

import javax.xml.bind.DatatypeConverter;
import java.nio.ByteBuffer;
import java.util.LinkedHashMap;
import java.util.Map;

public class MyTest2 {
    public static void main(String[] args) {

        08-30 15:06:41.853611
               String hexString =
                   "0e 1f 48 2d 7e 32 06 82 25 e9 a3 d9 08 00 45 00 " +
                       "00 ab 3f b3 40 00 40 06 35 ed 62 02 00 55 62 02 " +
                       "00 54 eb 2e 23 85 68 57 32 37 b5 08 3f b2 80 18 " +
                       "7d 2c c5 4a 00 00 01 01 08 0a b7 d1 e6 5c 26 30 " +
                       "b9 11 00 00 00 f9 11 85 6f c1 00 00 00 00 00 00 " +
                       "00 72 52 11 e0 02 13 5f 5f 63 6f 6e 73 75 6d 65 " +
                       "72 5f 6f 66 66 73 65 74 73 03 00 00 00 03 00 00 " +
                       "00 00 00 00 15 09 0c 9c 00 00 00 00 15 09 0c 9c " +
                       "00 00 00 00 00 00 00 00 00 ff ff ff ff 89 01 " +
        
                       "      00 00 00 00 15 09 0c 9c 00 00 00 7c 00 00 " +
                       "00 03 02 08 16 d8 10 00 00 00 00 00 00 00 00 01 " +
                       "91 a2 1b 8f 3c 00 00 01 91 a2 1b 8f 3c ff ff ff " +
                       "ff ff ff ff ff ff ff 00 00 00 00 00 00 00 01 92 " +
                       "01 00 00 00 56 00 01 00 0b 43 43 4d 2d 65 6e 63 " +
                       "72 79 70 74 00 16 43 53 53 2d 49 49 53 2d 53 69 " +
                       "74 75 61 74 69 6f 6e 51 75 65 72 79 00 00 00 00 " +
                       "30 00 03 00 00 00 00 00 00 00 00 ff ff ff ff 00 " +
                       "00 00 00 01 91 a2 1b 8f 3b 00 " +
        
        
        
                       "      00 00 00 00 15 00 00 00 00 00 00 13 36 67 " +
                       "47 00 00 00 00 13 36 67 47 00 00 00 00 00 00 00 " +
                       "00 00 ff ff ff ff 01 00 00 00 ";


//        String hexString =
//                "0e 1f 48 2d 7e 32 06 82 25 e9 a3 d9 08 00 45 00 " +
//                "00 ab 3f b3 40 00 40 06 35 ed 62 02 00 55 62 02 " +
//                "00 54 eb 2e 23 85 68 57 32 37 b5 08 3f b2 80 18 " +
//                "7d 2c c5 4a 00 00 01 01 08 0a b7 d1 e6 5c 26 30 " +
//                "b9 11 00 00 04 65 11 85 6f c0 00 00 00 00 00 00 " +
//                "00 72 52 11 e0 03 13 5f 5f 63 6f 6e 73 75 6d 65 " +
//                "72 5f 6f 66 66 73 65 74 73 02 00 00 00 15 00 00 " +
//                "00 00 00 00 13 36 67 3b 00 00 00 00 13 36 67 3b " +
//                "00 00 00 00 00 00 00 00 00 ff ff ff ff da 07 " +
//
//                "      00 00 00 00 13 36 67 3b 00 00 03 cd 00 00 " +
//                "00 03 02 5c 37 79 85 00 00 00 00 00 0b 00 00 01 " +
//                "91 a2 1b 8f 3c 00 00 01 91 a2 1b 8f 37 ff ff ff " +
//                "ff ff ff ff ff ff ff 00 00 00 00 00 00 00 0c 96 " +
//                "01 00 00 00 5a 00 01 00 0b 43 43 4d 2d 65 6e 63 " +
//                "72 79 70 74 00 16 43 53 53 2d 49 49 53 2d 53 69 " +
//                "74 75 61 74 69 6f 6e 51 75 65 72 79 00 00 00 00 " +
//                "30 00 03 00 00 00 00 00 00 00 00 ff ff ff ff 00 " +
//                "00 00 00 01 91 a2 1b 8f 3b 00 " +
//
//
//
//                "      00 00 00 00 15 00 00 00 00 00 00 13 36 67 " +
//                "47 00 00 00 00 13 36 67 47 00 00 00 00 00 00 00 " +
//                "00 00 ff ff ff ff 01 00 00 00 ";


        StringBuilder sb = new StringBuilder();
        char[] charArray = hexString.toCharArray();
        for (char c : charArray) {
            if (c != ' ') {
                sb.append(c);
            }
        }
        hexString = sb.toString();
        byte[] byteArray = DatatypeConverter.parseHexBinary(hexString);
        ByteBuffer byteBuffer = ByteBuffer.wrap(byteArray);
        System.out.println("len is : " + byteBuffer.array().length);
        byteBuffer.get(new byte[66]);
        byteBuffer.getInt();


        ResponseHeader responseHeader = ResponseHeader.parse(byteBuffer, (short) 0);
        System.out.println("responseHeader is " + responseHeader);
        FetchResponse fetchResponse = FetchResponse.parse(byteBuffer, (short) 11);
        System.out.println(fetchResponse);
        LinkedHashMap> map = fetchResponse.responseData();
        System.out.println("map size is : " + map.size());
        System.out.println("map is : " + map);
        for (Map.Entry> entry : map.entrySet()) {
            System.out.println();
            System.out.println();
            System.out.println();
            System.out.println();
            System.out.println("TP is : " + entry.getKey());
            FetchResponse.PartitionData value = entry.getValue();
            MemoryRecords records = value.records();
            records.batches().forEach(batch -> {
                System.out.println("isValid: " + batch.isValid());
                System.out.println("crc : " + batch.checksum());
                System.out.println("baseOffset : " + batch.baseOffset());
            });
        }
    }
}
```

其中关键的部分是查看2个CRC的逻辑：


* 一块是从协议体中获取到的，即Broker计算的CRC内容，这个对比传输前后均一致
* 一块是根据消息内容动态计算的，消息传输前，动态计算的CRC内容能与自身存储的对齐，但消息接收后，动态计算的CRC便发生了偏差


下图是真实生产业务日志中爆出的异常


![](https://img2024.cnblogs.com/blog/2109301/202409/2109301-20240902180528990-90494422.png)


其中发现计算出来的CRC是2481280076。而后将我们dump下来的二进制进行协议解码，执行上述代码后发现结果与生产环境的一致


![](https://img2024.cnblogs.com/blog/2109301/202409/2109301-20240902180547426-323643955.png)


# 四、磁盘排查


后续我们对发送出去的内容与磁盘的内容做了对比，内容一致，因此bug不是因磁盘导致


# 五、结论


由于出现问题的消息在TCP发送前后出现了diff，而正常收到的消息，在TCP发送前后均一致，因此


**定位为网络出现了问题**


网络的同学也认为这个现象不符合预期，已经开始介入了排查


 


总结：其实根据这些年处理Kafka异常的经验，这个bug第一时间拿到的时候，就感觉不像是Kafka自己的问题，关键是异常报错在Kafka的Broker端，我们需要不断提升自证清白的能力



 本博客参考[楚门加速器p](https://tianchuang88.com)。转载请注明出处！
