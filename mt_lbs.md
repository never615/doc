
# MTLBS 室内定位sdk

墨兔将会提供室内定位sdk、地图sdk、导航sdk，本文是关于室内定位sdk的开发计划。

## 功能
---

1. 基本的位置数据（经纬坐标、所在商城、分区（东西座、ABC区等）、楼层、店铺的id（如果beacon是店铺beacon的话））
2. 支持离线定位,启用离线模式后，会优先使用本地数据计算位置。
3. 提供综合定位，通过ibeacon+gps+传感器实现高精度的综合定位。
4. *【暂不实现】* 所在poi描述（可以设置围栏返回范围的所有poi或按数量设置返回[id、name、type、dist])
5. *【暂不实现】* 反地理编码+位置语义（如：海上世界B区2楼海底捞附近）
6. *【暂不实现】* 精准推荐，根据位置和用户数据提供个性化、智能化广告推送和运营支持。
7. *【暂不实现】* 电子围栏、位置提醒

*4、5两点支持离线缓存和设置缓存策略*


## 总体设计
---
![定位模块](https://raw.githubusercontent.com/never615/respository/master/%E5%A2%A8%E5%85%94%E4%BD%8D%E7%BD%AE%E6%9C%8D%E5%8A%A1.png)

**设计目标：可以支持扩展多个商城。** 

### 云端数据管理

1. 为了便于管理，云端需要对各个商城的数据分开管理更新
2. app请求下载数据库时，云端需要根据请求的商城id数组将数据打包到一个数据库

  
*考虑到现在只有一个商城，请求数据的接口暂不实现上述逻辑（动态生成数据库），使用临时方案。  
**临时方案：**
当app端携带**商城id**和**数据版本号**请求数据时，如果是海上世界id，则根据版本号，决定是否返回最新数据库（和现有数据升级方案类似，只是数据内容不一样）。如果是其他id则返回对应错误码。*

### 定位大概流程

![定位流程图](https://raw.githubusercontent.com/never615/respository/master/%E5%AE%9A%E4%BD%8D%E6%B5%81%E7%A8%8B.png)


### 定位情况各种情况分析
app启动后，定位当前位置，判断当前所在商城。
如果定位不到：回调给上层处理。判断当前位置必须联网。



### app端数据方案
1. 建立本地config表，用来管理各商场的数据版本情况
2. 数据支持自动更新和预置两种方案，支持配置详细的升级参数
3. 数据升级采用事务管理
4. *【暂不实现】* 数据升级支持自定义第三方通道 


### 其他
* sdk加密
* 上传jcenter（android）
*  *【暂不实现】*app  key注册和验证机制，只有在墨兔注册的app才能使用数据升级服务


## 详细设计
---
### 数据库设计

表名     | 说明
-------- | ---
Mall     | 保存商圈/区域信息
Block    | 一个商圈的不同区域，如：海岸城东座、西座
Bloor    | 商场中一个区域 
Floor    | 楼层信息
Beacon   | iBeacon设备信息
CoordinateMapping	|gps 坐标和mt坐标的映射表
Config	|保存数据版本信息（单独数据库保存）

#### 1. 商城表（Mall）


字段    | 类型      |  约束     |  说明
---|---|---|---
id      | Integer   | PK        | 自动增长
name    | Text      | NOT NULL  | 商城名称
gpsLng  | Real      | NOT NULL  | 商城坐标gps经度
gpsLat  | Real      | NOT NULL  | 商城坐标gps纬度
bdLng   | Real      | NOT NULL  | 商城坐标bd09ll经度
bdLat   | Real      | NOT NULL  | 商城坐标bd09ll纬度
lef	  | Real		   | 	| 商城范围经度1 默认值 -1000
right		| Real		| 	| 商城范围经度2 默认值 -1000
top		| Real		| 	| 商城范围纬度1 默认值 -1000
down		| Real 		| 	| 商城范围纬度2 默认值 -1000


#### *【暂不使用】*2. 分区表（Block）
字段    | 类型      |  约束     |  说明
---|---|---|---
id      | Integer   | PK        | 自动增长
name    | Text      | NOT NULL  | 商场分区的名字
mallId  | Integer   | NOT NULL  | 商场id

#### *【暂不使用】*3. 楼层表（Floor）
字段    | 类型      |  约束     |  说明
---|---|---|---
id      | Integer   | PK        | 自动增长
name    | Text      | NOT NULL  | 楼层名字
mallId  | Integer   | NOT NULL  | 商城标识

#### *【暂不使用】*4. 自定义区域表（Bloor）
字段    | 类型      | 约束      | 说明
---|---|---|---
id      | Integer   | PK        | 自动增长
blockId | Integer   | NOT NULL  | 商城分区标识
floorId | Integer   | NOT NULL  | 楼层分区标识
type    | Integer   |           | 区域类型（暂时用不到） 

#### 5. beacon表（Beacon）
区域类型（regionType） 1:公共区域；2:商场；3:停车场；4:店铺

字段    | 类型      | 约束      | 说明
---|---|---|---
id      | Integer   | PK        | 自动增长
major   | Integer   | NOT NULL  | beacon携带的参数
minor   | Integer   | NOT NULL  | beacon携带的参数
longitude|Real      | NOT NULL  | 墨兔坐标经度
latitude |Real      | NOT NULL  | 墨兔坐标纬度
regionType|Integer  | NOT NULL  | 所属区域类型id
shopId  | Integer   | NOT NULL  | 店铺id，非店铺beacon，则为0
bloorId | Integer   | NOT NULL  | 墨兔自定义分区的id
floorId | Integer   | NOT NULL  | 楼层id
blockId | Integer   | NOT NULL  | 商城分区id
mallId  | Integer   | NOT NULL  | 商城id

#### 6. gps坐标和墨兔坐标映射表(MappingCoordinate)

字段	| 类型			| 约束			| 说明
---|---|---|---
id		| Integer	| PK			| 自动增长
gpsLat| Real		| NOT NULL	| gps(wgs84)纬度坐标
gpsLng| Real		| NOT NULL	| gps经度坐标
bdLat|Real		| NOT NULL	| 百度(bd09ll)坐标纬度
bdLng|Real		| NOT NULL	| 百度(bd09ll)坐标经度
mtLat|Real		| NOT NULL	| 墨兔坐标纬度
mtLng|Real		| NOT NULL	| 墨兔坐标经度
bloorId|Integer| NOT NULL	| 墨兔自定义区域id
floorId|Integer|NOT NULL	| 楼层id
blockId|Integer|NOT NULL	| 分区id
mallId|Integer	| NOT NULL	| 商城id

#### 7. 本地保存信息的表（config）本地表
字段|类型|约束|说明
---|---|---|---
id|Integer|PK|自动增长
mallId|Integer|NOT NULL|商城id
dataVerision|Integer|NOT NULL|商城所对应的数据版本

### 接口设计
#### 定位api
1. 初始化MtLocationClient及相关参数的配置
    1. 设置是否在stop的时候杀死这个进程，默认不杀死。配置定位SDK内部是一个SERVICE，并放到了独立进程，  
    2. 配置前台扫描频率
    3. 配置后台扫描频率
    4. 配置是否启用自动更新
    5. 配置关闭sdk内部实现的gps定位（可以调用者自己实现）
    6. 是否启用gps辅助定位（5、6点建议一起开关）
    7. 配置uuid
2. 启动定位
3. 关闭定位
4. 定位结果监听
5. 设置当前gps位置，sdk根据此判断当前所在mall（如果不设置此方法，sdk内部有默认实现gps获取的实现）
6. 检查更新（要更新的mall的id数组（空的话更新全部），是否强制更新）
7. 自动更新参数的配置
    1. 检查周期
    2. 网络条件
    3. 是否充电
8. 配置数据库地址（默认是：）




### 约束




## 开发者使用说明
---
1. 使用室内定位服务前要保证蓝牙开启
2. sdk支持到minSdkVersion 9，但是使用定位功能至少需要android4.3以上和蓝牙4.0
3. 本地没有数据的情况下，云端定位受网络环境影响
4. 如果无网且本地也没有数据，则无法定位。本地有数据，使用本地数据计算，否则请求云端接口。
5. 内部使用了百度的定位sdk，所以需要申请秘钥，[点此申请](http://lbsyun.baidu.com/index.php?title=android-locsdk/guide/key)


**申请key之后在application标签中加入：**

	<meta-data
            android:name="com.baidu.lbsapi.API_KEY"
            android:value="key" />       //key:开发者申请的key



**声明权限**
 
	 <!-- 这个权限用于进行网络定位-->
    <uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION"/>
    <!-- 这个权限用于访问GPS定位-->
    <uses-permission android:name="android.permission.ACCESS_FINE_LOCATION"/>
    <!-- 用于访问wifi网络信息，wifi信息会用于进行网络定位-->
    <uses-permission android:name="android.permission.ACCESS_WIFI_STATE"/>
    <!-- 获取运营商信息，用于支持提供运营商信息相关的接口-->
    <uses-permission android:name="android.permission.ACCESS_NETWORK_STATE"/>
    <!-- 这个权限用于获取wifi的获取权限，wifi信息会用来进行网络定位-->
    <uses-permission android:name="android.permission.CHANGE_WIFI_STATE"/>
    <!-- 用于读取手机当前的状态-->
    <uses-permission android:name="android.permission.READ_PHONE_STATE"/>
    <!-- 写入扩展存储，向扩展卡写入数据，用于写入离线定位数据-->
    <uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE"/>
    <!-- 访问网络，网络定位需要上网-->
    <uses-permission android:name="android.permission.INTERNET"/>
    <!-- SD卡读取权限，用户写入离线定位数据-->
    <uses-permission android:name="android.permission.MOUNT_UNMOUNT_FILESYSTEMS"/>
	<!-- 蓝牙权限，用于获取蓝牙状态 -->
	<uses-permission android:name="android.permission.BLUETOOTH"/>
	<!-- 蓝牙权限，用于控制蓝牙开关 -->
	<uses-permission android:name="android.permission.BLUETOOTH_ADMIN"/>
	<!-- 声明应用需要使用设备的蓝牙BLE -->
	<uses-feature
        android:name="android.hardware.bluetooth_le"
        android:required="true"/>
   
**声明服务**

	在application标签中声明service组件,每个app拥有自己单独的定位service
	<service
            android:name="com.baidu.location.f"
            android:enabled="true"
            android:process=":remote"/>
	<service
            android:name="com.aprilbrother.aprilbrothersdk.service.BeaconService"
            android:exported="false"/>

### 使用示例
    MtLocationManager mtLocationManager = new MtLocationManager
            .Builder(this)
            .locationListener(new MtLocationManager.MtLocationListener() {
                @Override
                public void notFoundMall() {
                    Logger.t(TAG).e("not found mall");
                }

                @Override
                public void onLocationChange(MtLocation mtLocation) {
                    Logger.t(TAG).e(mtLocation.toString());
                }
            })
            .build();
     mtLocationManager.startLocate();       

## 开发者需要注意的问题
---
1. 建议提前预置数据库
2. 数据库需要放到手机上应用包目录下的DATABASES目录中

