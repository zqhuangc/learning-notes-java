## 概念
DoS即Denial Of Service，拒绝服务的缩写。DoS攻击是指故意的攻击网络协议实现的缺陷或直接通过野蛮手段残忍地耗尽被攻击对象的资源，目的是让目标计算机或网络无法提供正常的服务或资源访问，使目标系统服务系统停止响应甚至崩溃，而在此攻击中并不包括侵入目标服务器或目标网络设备。  
这些服务资源包括网络带宽，文件系统空间容量，开放的进程或者允许的连接。这种攻击会导致资源的匮乏，无论计算机的处理速度多快、内存容量多大、网络带宽的速度多快都无法避免这种攻击带来的后果。  
利用软件实现的缺陷  
利用协议的漏洞 synflood攻击 发送SYN报文，而不返回ACK报文
攻击程序 

攻击流程
作用于TCP连接的三次握手
攻击手段
DoS具有代表性的攻击手段包括PingofDeath、TearDrop、UDPflood、SYNflood、LandAttack、IPSpoofingDoS等


## Hash Collision DoS（Hash碰撞的拒绝式服务攻击）
利用各语言的Hash算法的“非随机性”，hash碰撞，一次性录入多条hashcode相同的记录  
hash map
json转map的问题


防御措施    
要防守这样的攻击，可以尝试下面几招：   
打补丁，把hash算法改了。  
限制POST的参数个数，限制POST的请求长度。  
最好还有防火墙检测异常的请求。