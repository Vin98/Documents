> 此文翻译自[Apple官方文档](https://developer.apple.com/library/archive/documentation/NetworkingInternet/Conceptual/RemoteNotificationsPG/APNSOverview.html#//apple_ref/doc/uid/TP40008194-CH8-SW1)

# APNs
**APNs**(Apple Push Notification service)是Apple远程通知服务的核心，开发者可以通过APNS向iOS、tvOS、macOS终端发送通知

## Provider To APNs
provider 本质上是一个服务器，由开发者部署配置，用来向APNs发送通知，APNs接收信息后，向目标应用发送对应的通知信息，在终端设备接收到通知后，进而将信息发给应用，并且管理用户与通知的交互。如下：

	Providers ------->  APNs ------->   Terminal  ------->  Client APP
- 当终端接收到通知的时候，应用并不处于运行状态，系统还是会正常的显示通知。
- 如果APNs发送通知的时候，终端设备关机，APNs就会保留信息，并在一段时间后重试

## APNs To Client APP
应用在用户设备上运行的时候会在用户设备和APNS之间建立一个安全的数据交互连接，应用通过该连接接收通知

## Provider Responsibilities
> provider与APNs沟通连接时的责任
>> 1. 通过APNS获取APP的全球唯一的设备验证码以及其他一些数据，允许provider获取APP的运行情况
>> 2. 根据APP的需求来决定通知推送的时间
>> 3. 建立通知请求，并发送到APNs，APNs再将其通知到设备

> 对于每一个通知请求，provider需要:
>> 1. 组建一个JSON数据，其中包含通知信息
>> 2. 添加device token和通知信息到一个HTTP/2请求
>> 3. 通过一个永久安全的线路(Security Architecture)发送包含证书的HTTP/2请求到APNs

## Using Multiple Providers 
对于多provider，每一个都有其自己的与APNs的安全长连接，任意一个持有device token的provider都可以向APNs发送通知请求  
![multiple provides](https://developer.apple.com/library/archive/documentation/NetworkingInternet/Conceptual/RemoteNotificationsPG/Art/remote_notif_multiple_2x.png)

## Quality of Service, Store-and-Forward, and Coalesced Notifications 
APNs 包括一个Quality of Service(QoS)组件来实现存储转发功能，当APNs试图向一个离线设备发送通知的时候，APNs将保存通知一段时间并在设备上线之后重新递送此通知，但是只保存每个设备的每个APP的最后一个通知，之前的通知将丢弃，并且如果设备长期离线，APNs将丢弃所有通知   
在通知请求中加入collapse id 可以合并相似通知。通常情况下当设备在线时，发送给APNs的每一个通知都将递送给设备，但是如果HTTP/2请求头中包含了apns-collapse-id，APNs将合并key值相同的通知请求

## Security Architecture
APNs强制实施端到端的，加密验证以及双层认证机制:连接信任和device token信任  
**连接信任工作于providers与APNs以及APNs与设备之间**   
> 1. providers与APNs之间的信任确保只有与Apple达成一致的公司拥有的经过认证的provider才能推送通知
> 2. APNs与device之间的信任确保只有经过认证的设备能连接到APNs并接收通知。APNs自动实施与设备的连接信任来确保设备的合法性    

provider要与APNs通信，必须拥有一个有效的key certificate(基于token的连接信任)或者SSL certificate(基于certificate的连接信任)，不管选择哪一种类型的证书，provider的连接信任都是provider向APNs发送通知请求的必备条件

**设备token信任作用于每一个远程通知的端到端，它确保通知只从provider发出并只由device接收**  
device token是一个无法辨认 NSData实例，包含一个由Apple分配给一个具体的设备上的一个具体的APP的唯一id
