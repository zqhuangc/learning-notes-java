## RBAC（Role-Based Access Control ）基于角色的访问控制。  
用户-角色-权限-资源

用户
组织
组织角色
角色
角色功能
功能
组织管理的功能（分级授权用）

模块
角色
组织
分级授权

RBAC支持公认的安全原则：最小特权原则、责任分离原则和数据抽象原则


RBAC96是一个模型族，其中包括RBAC0~RBAC3四个概念性模型。

1、基本模型RBAC0定义了完全支持RBAC概念的任何系统的最低需求。

2、RBAC1和RBAC2两者都包含RBAC0，但各自都增加了独立的特点，它们被称为高级模型。

RBAC1中增加了角色分级的概念，一个角色可以从另一个角色继承许可权。

RBAC2中增加了一些限制，强调在RBAC的不同组件中在配置方面的一些限制。

3、RBAC3称为统一模型，它包含了RBAC1和RBAC2，利用传递性，也把RBAC0包括在内。这些模型构成了RBAC96模型族。

RBAC0的模型中包括用户（U）、角色（R）和许可权（P）等3类实体集合。