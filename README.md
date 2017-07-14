<!--
Licensed to the Apache Software Foundation (ASF) under one
or more contributor license agreements.  See the NOTICE file
distributed with this work for additional information
regarding copyright ownership.  The ASF licenses this file
to you under the Apache License, Version 2.0 (the
"License"); you may not use this file except in compliance
with the License.  You may obtain a copy of the License at

  http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing,
software distributed under the License is distributed on an
"AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
KIND, either express or implied.  See the License for the
specific language governing permissions and limitations
under the License.
-->

# flume版本说明!

flume目前最高版本1.7.0，默认支持elasticsearch版本0.90.1。想与2.4.4版本整合，难度很大。各种百度google，
发现不好使，于是对源码进行了研究，发现直接修改下flume-ng-elasticsearch-sink模板，就OK了。
不兼容的问题主要是因为 elasticsearch client连接方式，不同的版本不同的写法。因此网上各种说删jar，替换都是瞎扯蛋。

# 为什么一定要flume与elasticsearch整合呢？
我觉得elasticsearch+kibana组合完美，统计直接出图，分分钟解决老板各种出数据出表问题。当然elasticsearch也只是保留7天内数据，
（全部原始数据放到hdfs里去啊，以后用hive做离线分析用。）kibana出的统计数据可以精确到分钟级别。这样就可以使用kibana的插件sentinl，做数据监控，监控基本数据指标，也就鼠标点点，也就搞定。
比如，监控的北京地区客户端支付协议延迟超5s比例过5%，马上发短信报警，先出个拼图，再用sentinl设置下，报警马上接上。so easy！

# 为什么不用logstash而用flume呢？
1、logstash拿到数据本来没做数据缓存，需要agent->[redis或kafka等],然后从redis或kafka拉数据到elasticsearch。如果redis或kafka
内存爆了，logstash就假死了。虽然logstash使用非常方便，但是这块做的不好。但是flume不一样，分布式采集，容错都不错。而且channel提供的file
和memory，非常完美。尤其是memory，直接跟flume整合一起，同一个进程，memory channel使用的是guava cacha实现，速度也是非常快。
2、flume事务提交的，有二段重试和故障转移功能，flume而且还是java写的，看看就明白了，哈哈

# flume+es的架构
![image](https://raw.githubusercontent.com/moueimei/flume-release-1.7.0/master/jgt.png)

# 上面都是废话，如何使用呢？
1、下载整个工程到本地，maven编译。
2、将flume-ng-core-1.7.0.jar和flume-ng-elasticsearch-sink-1.7.0.jar放到 FLUME_HOME/lib 替换原来的。
3、将elasticsearch-2.4.4.jar和她依赖的jsr166e-1.1.0.jar lucene-core-5.5.2.jar t-digest-3.0.jar jackson-dataformat-cbor-2.8.1.jar jackson-core-2.8.1.jar
hppc-0.7.1.jar guava-18.0.jar compress-lzf-1.0.2.jar放到FLUME_HOME/lib下。东西有点多，不怕，搞一个工程，把elasticsearch2.4.4 maven引入
下，打包成war包，里面的lib目录都有这些，不用傻乎乎一个个去找。不知道flume不想升级elasticsearch2.4.4原因是不是因为要加这么多包的原因。
4、当然把jackson-core之前的老版本删除下。就可以愉快的玩耍了。

# 如果证明好使呢
写一个flume的http的handlel类org.pq.client.HTTPSourceHandler吧，类似如下：
[code]

public class HTTPSourceHandler implements org.apache.flume.source.http.HTTPSourceHandler {
    private static final Logger LOG = LoggerFactory.getLogger(HTTPSourceHandler.class);
    private boolean insertTimestamp;

    public List<Event> getEvents(HttpServletRequest request) throws HTTPBadRequestException, Exception{
         List<Event> events=new ArrayList<>();
        try {
            String content = request.getParameter("content");
            if(StringUtils.isEmpty(content)){
                return Collections.emptyList();
            }
            String ip = GETIP.getIpAddr(request);
            JSONObject json = JSONObject.parseObject(content);
            Map<String, String> eventHeaders
                    = new HashMap<String, String>();
            long now = DateTimeUtils.currentTimeMillis();
            eventHeaders.put("@timestamp", now+"");

            events.add(EventBuilder.withBody(content, Charset.defaultCharset(),
                    eventHeaders));
        } catch (Exception ex) {
            throw new HTTPBadRequestException(
                    "Request is not in expected format. " +
                            "Please refer documentation for expected format.", ex);
        }
        return events;
    }
    /**
    * flume 内部headers不能传null，一定要""这样，反正是小问题
    */
    private String ifNullDefault(String value) {
        return StringUtils.defaultIfEmpty(value,BLANK);
    }

    public void configure(Context context) {
        insertTimestamp = context.getBoolean(CONF_INSERT_TIMESTAMP,false);
    }

    把这个类打成jar包，在FLUME_HOME/下，mkdir -p plugins.d/mytest/lib mkdir -p plugins.d/mytest/libext mkdir -p plugins.d/mytest/native
    把jar包放到plugins.d/mytest/lib下面，另外如果你用到其他的jar包，请放到plugins.d/mytest/libext下，比如用到fastjson.jar native你懂的，我也不清楚。


    写个config吧
    如下：
    a50000.sources=r1
    a50000.sinks=k1
    a50000.channels=c1

    a50000.sources.r1.type=http
    a50000.sources.r1.bind=10.xx.xxx.140
    a50000.sources.r1.port=50000
    a50000.sources.r1.channels=c1
    a50000.sources.r1.handler=org.pq.client.HTTPSourceHandler
    a50000.sources.r1.insertTimestamp=true

    # a50000.sinks.k1.type=logger
    # a50000.sinks.k1.channel=c1

    a50000.sinks.k1.type=org.apache.flume.sink.elasticsearch.ElasticSearchSink
    a50000.sinks.k1.batchSize=5
    a50000.sinks.k1.hostNames=10.xx.xxx.xxx,10.xxx.100.xxx,10.xxx.100.xx
    a50000.sinks.k1.indexType=log
    a50000.sinks.k1.indexName=arrow-client
    a50000.sinks.k1.clusterName=cmplay-es-cluster
    a50000.sinks.k1.serializer=org.apache.flume.sink.elasticsearch.ElasticSearchDynamicSerializer
    a50000.sinks.k1.indexNameBuilder=org.apache.flume.sink.elasticsearch.TimeBasedIndexNameBuilder
    a50000.sinks.k1.channel=c1

    a50000.channels.c1.type=memory
    a50000.channels.c1.capacity=1000
    a50000.channels.c1.transactionCapacity=500

    启动下 cd FLUME_HOME运行：
    bin/flume-ng agent -c conf -f http_50000.conf  -n a50000 -Dflume.root.logger=INFO,console

    启动好了，在浏览器请求下，就可以测试了。
    为什么我要说到这个呢，因为，我不去确定，整合的时候，是不是还有什么包需要替换和删除的，通过这个方式，你就可以慢慢调试了，报错了，发现NODEFINE CLASSS
    之类的报错，你还不知道缺啥少啥吗？
# 稳定性怎么样？
    可以这么说，现在看起来修改flume框架代码core和sink，其实直接去修改，就发现改的就几行代码，因为jar包升级了，之前低版本的写法变了变，还是原来的flume，
 目前我线上跑着，每天数据杠杆的，没任何异常。

 不要怕改框架源码，走出去了，才发现万变不离其宗。神马都是浮云~~~~~~~~

