---
layout:     post
title:      "Alibaba-fastjson-Object-parse"
subtitle:   "Json解析"
date:       2019-11-15 16:00:00
author:     "Autu"
header-img: "img/post-bg-js-module.jpg"
header-mask: 0.3
catalog:    true
tags:
    - Json
    - 通用配置
---

### 关于 <code>com.alibaba.fastjson.JSON</code> 中json解析详解

简单介绍一下这3个实体类之间的关系: 

- Host: 物理主机类, 主要用来描述云平台中物理主机的相关信息
- DataStore: 数据存储详细信息类, Host中包含多个DataStore列表
- StorageUnits: 存储单元类, DataStore包含多个存储单元



#### 1. 先把实体类粘到这里

##### Host实体类

```java
import com.fasterxml.jackson.annotation.JsonInclude;
import io.swagger.annotations.ApiModel;
import org.springframework.data.annotation.Id;
import org.springframework.data.mongodb.core.mapping.Document;

import java.io.Serializable;
import java.util.List;

/**
 * host平台信息实体类
 */
@ApiModel(value = "host平台信息", description = "定时更新来自平台的host信息")
@Document(collection = "host_cloud")
@JsonInclude(JsonInclude.Include.NON_NULL)
public class HostCloud implements Serializable {
    private static final long serialVersionUID = -7640376594563631583L;

    @Id
    private String id;
    /**host urn*/
    private String urn;
    /**主机中已经挂载光驱的虚拟机标识列表若无虚拟机使用主机光驱则此字段为[]*/
    private List<String> attachedISOVMs;
    /**bmc的IP地址*/
    private String bmcIp;
    /**bmc账号*/
    private String bmcUserName;
    /**主机所在集群是否开启 "本地内存盘" */
    private Boolean	clusterEnableIOTailor;
    /**集群名*/
    private String clusterName;
    /**集群ID, 空时表明主机直接位于站点下*/
    private String clusterUrn;
    /**查询该主机计算资源的uri地址*/
    private String computeResourceStatics;
    /**主机的CPU主频大小*/
    private Integer	cpuMHz;
    /**CPU数量*/
    private int cpuQuantity;
    /**数据存储列表*/
    private List<DataStore> dsList;
    /**主机重启后图形桌面虚拟机用于发送的共享内存大小*/
    private int gdvmMemoryReboot;
    /**GPU共享虚拟机数量*/
    private int gpuCapacity;
    /**主机重启后GPU共享虚拟机数量*/
    private int gpuCapacityReboot;
    /**主机重启后图形桌面虚拟机用于接收的共享内存大小*/
    private int gsvmMemoryReboot;
    /**主机实际生效的多路径类型属性*/
    private String hostMultiPathMode;
    /**OS主机名称*/
    private String hostRealName;
    /**主机IP*/
    private String ip;
    /**是否被指定为故障切换主机, 默认false, 不指定*/
    private Boolean isFailOverHost;
    /**主机是否是维护状态*/
    private Boolean isMaintaining;
    /**主机CPU最大能够支持的IMC模式*/
    private String maxImcSetting;
    /**主机内存总大小*/
    private Integer	memQuantityMB;
    /**用户设置的多路径类型属性*/
    private String multiPathMode;
    /**主机名称*/
    private String name;
    /**网卡数量*/
    private int nicQuantity;
    /**物理CPU数量*/
    private int physicalCpuQuantity;
    /**数据中心信息*/
    private Site site;
    /**主机状态*/
    private String status;
    /**访问主机的uri*/
    private String uri;

    // 这里省去set get toString方法
}

```

##### DataStore实体类

```java
import com.fasterxml.jackson.annotation.JsonFormat;
import com.fasterxml.jackson.annotation.JsonInclude;
import io.swagger.annotations.ApiModel;

import java.io.Serializable;
import java.util.Date;
import java.util.List;

/** 
 * 数据存储相关信息类
 */
@ApiModel(value="dataStore",description="存储详细信息")
@JsonInclude(JsonInclude.Include.NON_NULL)
public class DataStore  implements Serializable{
    private static final long serialVersionUID = -1172069639991209788L;

    /**唯一标识数据存储的urn*/
    private String urn;
    /**唯一标识数据存储的uri*/
    private String uri;
    /**datastore所关联的存储设备Urn*/
    private String suUrn;
    /**datastore所关联的存储设备名称*/
    private String suName;
    /**存储类型*/
    private String storageType;
    /**簇大小, 虚拟化文件系统的簇大小*/
    private Integer clusterSize;
    /**可读的数据存储的名字*/
    private String name;
    /**数据存储状态*/
    private String status;
    /**数据存储的最大可用容量*/
    private Integer capacityGB;
    /**数据存储的已经使用空间*/
    private Integer usedSizeGB;
    /**数据存储的空闲空间*/
    private Integer freeSizeGB;
    /**数据存储挂接的主机URN*/
    private List<String> hosts;
    /**是否支持精简配置*/
    private Boolean isThin;
    /**超分配比率, 在100到300之间*/
    private Integer thinRate;
    /**实际总空间, 非精简配置情况下和capacityGB相同*/
    private Integer actualCapacityGB;
    /**实际剩余空间, 非精简配置情况下和freeSizeGB相同*/
    private Integer actualFreeSizeGB;
    /**上次刷新时间, UTC时间*/
    @JsonFormat(pattern = "yyyy-MM-dd HH:mm:ss",timezone = "GMT+8")
    private Date refreshTime;
    /**数据存储对应的存储设备列表, 其中第一个存储设备为主LUN*/
    private List suIdList;
    /**存储设备列表*/
    private List<StorageUnits> storageUnits;

}

```

##### StorageUnits实体类

```java
import com.fasterxml.jackson.annotation.JsonInclude;
import io.swagger.annotations.ApiModel;

import java.io.Serializable;

/**
 * 存储类
 */
@ApiModel(value="StorageUnits",description="存储单元")
@JsonInclude(JsonInclude.Include.NON_NULL)
public class StorageUnits  implements Serializable {
    private static final long serialVersionUID = -1768233895565362278L;

    /**唯一标识存储设备的urn*/
    private String urn;
    /**存储设备名称*/
    private String sdName;
    /**关联的存储资源名称*/
    private String suName;

}
```



#### 2. 将json解析为实体类

关于 <code>JSON.parseObject(String text, Class<T> clazz)</code>  的作用是将Json解析为实体对象

<code>JSON.parseObject(hostCloudStr, HostCloud.class)</code>  , 此方法是将json转化为HostCloud实体类, hostCloudStr已经粘在了下面

```json
{
    "attachedISOVMs":[

    ],
    "bmcIp":"192.168.112.253",
    "bmcUserName":"root",
    "clusterEnableIOTailor":false,
    "clusterName":"ManagementCluster",
    "clusterUrn":"urn:sites:4EE5091A:clusters:115",
    "computeResourceStatics":"/service/sites/4EE5091A/hosts/170/computeResourceStatics",
    "cpuMHz":2800,
    "cpuQuantity":36,
    "dsList":[
        {
            "actualCapacityGB":1032,
            "actualFreeSizeGB":258,
            "capacityGB":1032,
            "clusterSize":1024,
            "freeSizeGB":258,
            "hosts":[
                "urn:sites:4EE5091A:hosts:170"
            ],
            "name":"autoDS_CNA01",
            "refreshTime":1573769026000,
            "status":"CONNECTED",
            "storageType":"LOCALPOME",
            "storageUnits":[
                {
                    "sdName":"LOCAL",
                    "suName":"36c81f660dc81370022cfb63d0bb0c55a",
                    "urn":"7041688F143F42FE9B1E9C3DAC3F9D5C"
                }
            ],
            "suIdList":[

            ],
            "suName":"36c81f660dc81370022cfb63d0bb0c55a",
            "suUrn":"urn:sites:4EE5091A:storageunits:7041688F143F42FE9B1E9C3DAC3F9D5C",
            "thin":true,
            "thinRate":100,
            "uri":"/service/sites/4EE5091A/datastores/1",
            "urn":"urn:sites:4EE5091A:datastores:1",
            "usedSizeGB":1260
        }
    ],
    "failOverHost":false,
    "gdvmMemoryReboot":128,
    "gpuCapacity":1,
    "gpuCapacityReboot":1,
    "gsvmMemoryReboot":128,
    "hostMultiPathMode":"CURRENCY",
    "hostRealName":"CNA01",
    "ip":"172.22.50.11",
    "maintaining":false,
    "maxImcSetting":"IvyBridge",
    "memQuantityMB":118898,
    "multiPathMode":"CURRENCY",
    "name":"CNA01",
    "nicQuantity":4,
    "physicalCpuQuantity":2,
    "site":{
        "dc":false,
        "ip":"172.22.51.10",
        "name":"site",
        "self":true,
        "status":"normal",
        "uri":"/service/sites/4EE5091A",
        "urn":"urn:sites:4EE5091A"
    },
    "status":"normal",
    "uri":"/service/sites/4EE5091A/hosts/170",
    "urn":"urn:sites:4EE5091A:hosts:170"
}
```



#### 3. 将json解析为实体类集合

<code>public static <T> List<T> parseArray(String text, Class<T> clazz)</code>  , 该方法是将实体类集合对象的json转换为实体类List对象

```java
HwApiResult findHosts = apiCall("findHostsByUris", map);

Object result = findHosts.getResult();
// Object类型的数据不可以直接解析为实体类, 需要先转换为字符串
String lineArray = JSONArray.toJSONString(result);
// 解析json为HostCloud对象
List<HostCloud> hostClouds = JSON.parseArray(lineArray, HostCloud.class);
```

下面为代表实体类集合的json串

```json
[
    {
        "physicalCpuQuantity":2,
        "clusterUrn":"urn:sites:4EE5091A:clusters:115",
        "cpuMHz":2800,
        "clusterEnableIOTailor":false,
        "cpuQuantity":36,
        "isFailOverHost":false,
        "dsList":[
            {
                "freeSizeGB":258,
                "actualCapacityGB":1032,
                "suUrn":"urn:sites:4EE5091A:storageunits:7041688F143F42FE9B1E9C3DAC3F9D5C",
                "hosts":[
                    "urn:sites:4EE5091A:hosts:170"
                ],
                "refreshTime":"2019-11-15 02:03:42",
                "isThin":true,
                "thinRate":100,
                "suIdList":[

                ],
                "uri":"/service/sites/4EE5091A/datastores/1",
                "usedSizeGB":1260,
                "clusterSize":1024,
                "urn":"urn:sites:4EE5091A:datastores:1",
                "capacityGB":1032,
                "name":"autoDS_CNA01",
                "storageType":"LOCALPOME",
                "storageUnits":[
                    {
                        "urn":"7041688F143F42FE9B1E9C3DAC3F9D5C",
                        "sdName":"LOCAL",
                        "suName":"36c81f660dc81370022cfb63d0bb0c55a"
                    }
                ],
                "actualFreeSizeGB":258,
                "suName":"36c81f660dc81370022cfb63d0bb0c55a",
                "status":"CONNECTED"
            }
        ],
        "hostMultiPathMode":"CURRENCY",
        "clusterName":"ManagementCluster",
        "maxImcSetting":"IvyBridge",
        "isMaintaining":false,
        "computeResourceStatics":"/service/sites/4EE5091A/hosts/170/computeResourceStatics",
        "memQuantityMB":118898,
        "ip":"172.22.50.11",
        "bmcIp":"192.168.112.253",
        "attachedISOVMs":[

        ],
        "gsvmMemoryReboot":128,
        "uri":"/service/sites/4EE5091A/hosts/170",
        "nicQuantity":4,
        "urn":"urn:sites:4EE5091A:hosts:170",
        "site":{
            "urn":"urn:sites:4EE5091A",
            "ip":"172.22.51.10",
            "name":"site",
            "isDC":false,
            "uri":"/service/sites/4EE5091A",
            "isSelf":true,
            "status":"normal"
        },
        "bmcUserName":"root",
        "gpuCapacityReboot":1,
        "hostRealName":"CNA01",
        "name":"CNA01",
        "multiPathMode":"CURRENCY",
        "gpuCapacity":1,
        "gdvmMemoryReboot":128,
        "status":"normal"
    },
    {
        "physicalCpuQuantity":2,
        "clusterUrn":"urn:sites:4EE5091A:clusters:115",
        "cpuMHz":2800,
        "clusterEnableIOTailor":false,
        "cpuQuantity":36,
        "isFailOverHost":false,
        "dsList":[
            {
                "freeSizeGB":869,
                "actualCapacityGB":1032,
                "suUrn":"urn:sites:4EE5091A:storageunits:5B0FC46B6AE14E0DA19F2F48A15612DA",
                "hosts":[
                    "urn:sites:4EE5091A:hosts:185"
                ],
                "refreshTime":"2019-09-16 07:05:19",
                "isThin":true,
                "thinRate":100,
                "suIdList":[

                ],
                "uri":"/service/sites/4EE5091A/datastores/2",
                "usedSizeGB":280,
                "clusterSize":1024,
                "urn":"urn:sites:4EE5091A:datastores:2",
                "capacityGB":1032,
                "name":"autoDS_CNA02",
                "storageType":"LOCALPOME",
                "storageUnits":[
                    {
                        "urn":"5B0FC46B6AE14E0DA19F2F48A15612DA",
                        "sdName":"LOCAL",
                        "suName":"36b083fe0c15e220022cf9c0cff18bde9"
                    }
                ],
                "actualFreeSizeGB":869,
                "suName":"36b083fe0c15e220022cf9c0cff18bde9",
                "status":"CONNECTED"
            }
        ],
        "hostMultiPathMode":"CURRENCY",
        "clusterName":"ManagementCluster",
        "maxImcSetting":"IvyBridge",
        "isMaintaining":true,
        "computeResourceStatics":"/service/sites/4EE5091A/hosts/185/computeResourceStatics",
        "memQuantityMB":103577,
        "ip":"172.22.50.12",
        "bmcIp":"192.168.112.254",
        "attachedISOVMs":[

        ],
        "gsvmMemoryReboot":128,
        "uri":"/service/sites/4EE5091A/hosts/185",
        "nicQuantity":4,
        "urn":"urn:sites:4EE5091A:hosts:185",
        "site":{
            "urn":"urn:sites:4EE5091A",
            "ip":"172.22.51.10",
            "name":"site",
            "isDC":false,
            "uri":"/service/sites/4EE5091A",
            "isSelf":true,
            "status":"normal"
        },
        "bmcUserName":"root",
        "gpuCapacityReboot":1,
        "hostRealName":"CNA02",
        "name":"CNA02",
        "multiPathMode":"CURRENCY",
        "gpuCapacity":1,
        "gdvmMemoryReboot":128,
        "status":"fault"
    }
]
```



#### 4. 将实体类转换为json

```java
// 将HostCloud实体类对象转换为json
JSONObject jsonObject = new JSONObject();
String hostCloudStr = jsonObject.toJSONString(hostCloud);
```