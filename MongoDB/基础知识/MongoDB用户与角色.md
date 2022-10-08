[TOC]

# MongoDB用户与角色

## 用户管理

在MongoDB启用AUTH认证后，需要通过用户名/密码以及认证数据库去连接实例，因此需要提前创建好相应的用户

**USER文档描述**

```
{
  user: "<name>",
  pwd: "<cleartext password>",
  customData: { <any information> },
  roles: [
    { role: "<role>", db: "<database>" } | "<role>",
    ...
  ],
  authenticationRestrictions: [
    {
       clientSource: ["<IP>" | "<CIDR range>", ...]
       serverAddress: ["<IP>" | "<CIDR range>", ...]
    },
    ...
  ],
  mechanisms: [ "<SCRAM-SHA-1|SCRAM-SHA-256>", ... ],
  passwordDigestor: "<server|client>"
}
```

**创建用户**

MongoDB可以通过db.createUser()函数来创建用户，创建用户的数据库是用户的身份验证数据库，虽然用户将对此数据库进行验证，但用户可以在其它数据库中拥有角色，即用户的验证数据库不影响用户的权限

```
> use test
> db.createUser( 
{ 
user:"test", 
pwd:"abcd123#", 
roles:[{role:"readWrite",db:"test"}],"mechanisms" : ["SCRAM-SHA-1"]})
```

**删除用户**

删除用户可以采用db.dropUser()，需要指定到用户所属的数据库下(低版本采用db.removeUser())，副本集环境建议添加majority参数。也可以直接通过db.dropAllUsers()删除当前数据库所有用户

```
> use test
> db.dropUser("test",{w:"majority",wtimeout:5000})
```

**修改用户名密码**

修改用户密码可以通过db.changeUserPassword(“username”,”password”)函数

```
> use test
> db.changeUserPassword("test","abcd123$")
```

**查看用户信息**

查看用户信息可以通过db.getUser(‘username’)查看用户名，数据库以及角色权限。也可以通过db.getUsers()查看当前所有用户信息

```
> use test
> db.getUser("test")
```

> 实例所有用户的信息都存放在admin数据库的system.users集合下，可以通过show user查看

**常用管理命令**

| 名称                     | 描述                         |
| :----------------------- | :--------------------------- |
| createUser               | 创建一个新用户。             |
| updateUser               | 更新用户的数据。             |
| dropUser                 | 删除一个用户。               |
| dropAllUsersFromDatabase | 删除与数据库关联的所有用户。 |
| grantRolesToUser         | 向用户授予角色及其特权。     |
| revokeRolesFromUser      | 从用户删除角色。             |
| usersInfo                | 返回有关指定用户的信息。     |



## 角色管理

MongoDB支持基于角色的访问控制来管理MongoDB用户访问的权限

**内置角色**

MongoDB提供了内置角色，可提供数据库系统不同的访问级别。每个数据库都存在内置的数据库用户角色和数据库管理角色

| 角色                 | 说明                                                         |
| :------------------- | :----------------------------------------------------------- |
| read                 | 允许用户读取指定数据库                                       |
| readWrite            | 允许用户读写指定数据库                                       |
| dbAdmin              | 允许用户在指定数据库中执行索引创建，统计信息收集等管理权限。此角色不授予用户和角色管理权限 |
| dbOwner              | 指定数据库的所有者，该角色可以对指定数据库执行任何管理操作，是由readWrite、dbAdmin和userAdmin组成 |
| userAdmin            | 允许用户在指定数据库上创建和修改角色和用户                   |
| clusterAdmin         | 作用于admin，允许用户对集群进行管理并且提供了dropDatabase的权限 |
| clusterManager       | 作用于admin，允许用户对集群进行管理和监视，访问分片和复制中的config和local数据库 |
| clusterMonitor       | 作用于admin，允许用户对监视工具只读访问权限                  |
| hostManager          | 作用于admin，允许用户监视和管理服务器                        |
| Backup/restore       | 作用于admin，允许用户通过mongodump和mongorestore执行备份恢复 |
| readAnyDatabase      | 作用于admin，允许用户只读访问除local和config以外的数据库     |
| readWriteAnyDatabase | 作用于admin，允许用户读写除local和config以外的数据库         |
| userAdminAnyDatabase | 作用于admin，允许用户拥有除local和config外的userAdmin权限    |
| dbAdminAnyDatabase   | 作用于admin，允许用户拥有除local和config以外的dbAdmin权限    |
| root                 | 超级管理员用户                                               |

具体内置角色信息，请参考官方链接：[built-in-roles](http://www.mongoing.com/docs/reference/built-in-roles.html#database-user-roles)

**自定义角色**

当需要授予用户大量的角色或权限，可以考虑创建一个角色来包含权限或继承其它角色

```
use admin
db.createRole(
{
  role: "myClusterwideAdmin",
  privileges: [
    { resource: { cluster: true }, actions: [ "addShard" ] },
    { resource: { db: "config", collection: "" }, actions: [ "find", "update" ] },
    { resource: { db: "users", collection: "usersCollection" }, actions:  [ "update" ] },
    { resource: { db: "", collection: "" }, actions: [ "find" ] }
  ],
  roles: [
    { role: "read", db: "admin" }
  ]
},
{ w: "majority" , wtimeout: 5000 })
```

**删除角色**

删除角色可以使用db.dropRole()函数，也可以使用db.dropAllRoles()删除当前数据库所有角色

```
>use admin
>db.dropRole( "myClusterwideAdmin", { w: "majority" } )
```

**向角色添加权限**

```
> use test
> db.grantPrivilegesToRole(
  "inventoryCntrl01",
  [
    {
       resource: { db: "test", collection: "" },
       actions: [ "insert" ]
    },
    {
       resource: { db: "test", collection: "system.js" },
       actions: [ "find" ]
    }
  ],
  { w: "majority" }
)
```

**回收角色权限**

```
> use test
> db.revokePrivilegesFromRole(
  "accountRole",
  [
    {
      resource : {
      db : "products",
      collection : "gadgets"
    },
    actions : [ "find","update" ]}
  ]
)
```

[Privilege Actions]:https://docs.mongodb.com/manual/reference/privilege-actions/#std-label-security-user-actions



**角色继承**

```
> use test
> db.grantRolesToRole(
  "productsReaderWriter",
  [ "productsReader" ],
  { w: "majority" , wtimeout: 5000 }
)
```

**授予角色给用户**

在用户创建完成后，可以通过db.grantRoleToUser()函数将角色授予用户

```
> use test
> db.grantRolesToUser(
  "test",
  [ "readWrite" , { role: "read", db: "stock" } ],
  { w: "majority" , wtimeout: 4000 }
)
```

**回收用户角色**

```
> use test
> db.revokeRolesFromUser( "test",
  [ { role: "read", db: "stock" }, "readWrite" ],
  { w: "majority" }
)
```

**角色管理命令**

| 名称                     | 描述                                                         |
| :----------------------- | :----------------------------------------------------------- |
| createRole               | 创建一个角色并指定其特权。                                   |
| updateRole               | 更新用户定义的角色。                                         |
| dropRole                 | 删除用户定义的角色。                                         |
| dropAllRolesFromDatabase | 从数据库中删除所有用户定义的角色。                           |
| grantPrivilegesToRole    | 将特权分配给用户定义的角色。                                 |
| revokePrivilegesFromRole | 从用户定义的角色中删除指定的特权。                           |
| grantRolesToRole         | 指定角色，用户定义的角色将从这些角色继承特权。               |
| revokeRolesFromRole      | 从用户定义的角色中删除指定的继承角色。                       |
| rolesInfo                | 返回指定角色的信息。db.runCommand({rolesInfo: { role: "associate", db: "products" },showPrivileges: true}) |
| invalidateUserCache      | 刷新用户信息的内存缓存，包括凭据和角色。                     |

