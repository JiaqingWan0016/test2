# Portal代码架构分析

通过对当前目录下Portal代码的分析，可以看出这是一个实现Portal认证功能的系统，包含内核模块和用户空间应用程序。Portal认证是一种常见的网络接入认证方式，通常用于无线网络、酒店网络等场景。下面从整体架构、目录结构、核心功能模块等方面进行分析。

## 1. 整体架构

Portal系统采用了典型的内核态与用户态协同工作的架构：

1. **内核模块**：负责网络数据包的拦截、处理和转发
2. **用户态程序**：负责认证逻辑、与外部Portal服务器通信、配置管理等

这种架构的优势在于：
- 内核模块可以高效处理网络数据包
- 用户态程序可以实现复杂的业务逻辑和配置管理
- 两者通过proc文件系统和socket通信

## 2. 目录结构

```
/portal
├── Makefile                  # 顶层Makefile
├── kernel_modules/           # 内核模块相关代码
│   ├── Makefile
│   ├── portal_kernel.c       # 内核模块主要实现
│   └── portal_kernel.h       # 内核模块头文件
├── script/                   # 脚本文件
│   └── se_portal             # Portal服务启停脚本
└── user/                     # 用户态程序
    ├── Makefile
    ├── app/                  # 应用程序
    │   └── portal_proccess.c # 主程序实现
    ├── config/               # 配置文件
    ├── include/              # 头文件
    │   ├── portal_proccess.h # 处理相关头文件
    │   └── portal_server.h   # 服务器相关头文件
    └── src/                  # 源代码
```

## 3. 核心功能模块

### 3.1 内核模块 (portal_kernel.c)

内核模块是系统的核心部分，主要功能包括：

1. **会话管理**：通过`TS_HANDLE`管理用户会话
2. **认证策略匹配**：`portal_auth_policy_match`函数用于匹配认证策略
3. **proc接口**：提供`/proc/tos/portal_local_switch`和`/proc/tos/portal_outside_switch`接口与用户态交互
4. **钩子函数**：注册网络钩子函数拦截数据包
5. **私有数据处理**：通过`tos_portal_ts_priv_handle`管理会话私有数据

关键数据结构：
```c
struct outside_portal_template {
    int   server_id;
    char  server_name[50];
    char  redirect_url[100];
    char  push_switch[5];
    char  userip_name[50];
    char  manufaturingno_name[100];
    char  server_ip[32];
    int   peer_port;
    int   show_flag;
    char  device_ip_name[100];
    char  user_access_addr_name[100];
}
```

### 3.2 用户态程序 (portal_proccess.c)

用户态程序负责实现Portal认证的业务逻辑，主要功能包括：

1. **守护进程管理**：通过`setup_daemon`、`check_pid`、`write_pid`等函数实现守护进程
2. **配置管理**：读取和解析配置文件
3. **控制接口**：`Control_Handle_portal`函数处理控制命令
4. **与内核通信**：通过proc文件系统与内核模块通信
5. **外部Portal服务器通信**：实现与外部Portal服务器的通信协议

关键数据结构：
```c
struct outside_portal_template {
    int   server_id;          // portal服务器id号，最大支持三个
    char  server_name[50];
    char  redirect_url[100];  // 重定向url
    char  push_switch[5];     // 推送开关
    char  userip_name[50];
    char  manufaturingno_name[100]; // 设备序列号名称
    char  server_ip[LEN32];   // 外部服务器ip地址
    int   peer_port;          // 外部服务器portal端口号
    int   show_flag;          // 显示标记
    char  device_ip_name[100];
    char  user_access_addr_name[100];
}

struct local_portal {
    char  localserver_ip[LEN32]; // 本地服务器ip地址
    int   jump_flag;             // 跳转标记
    char  jump_url[100];         // 跳转url
    char  switch_flag;
}
```

### 3.3 启动脚本 (se_portal)

启动脚本负责Portal服务的启动、停止和重启：

```bash
#!/bin/ash
. /tos/bin/functions

case $1 in
    start)
        # 加载内核模块并启动用户态程序
        insmod -f /tos/lib/portal.ko
        portal_proccess
        ;;
    stop)
        # 停止用户态程序并卸载内核模块
        killall portal_proccess
        rmmod portal
        ;;
    restart)
        # 重启服务
        $0 stop
        $0 start
        ;;
    *)
        echo "Usage: se_portal {start|stop|restart}"
        exit
esac
```

## 4. 通信机制

### 4.1 内核态与用户态通信

1. **proc文件系统**：
   - `/proc/tos/portal_local_switch`：控制本地Portal开关
   - `/proc/tos/portal_outside_switch`：控制外部Portal开关

2. **Socket通信**：用户态程序通过socket与内核模块交换数据

### 4.2 外部通信

1. **与外部Portal服务器通信**：
   - 支持多个外部Portal服务器（最多三个）
   - 通过HTTP协议实现重定向和认证

2. **心跳机制**：
   - `hearbeat_timeout_interval`参数控制心跳超时时间

## 5. 认证流程

根据代码分析，Portal认证流程大致如下：

1. 未认证用户访问网络资源
2. 内核模块拦截数据包，发现用户未认证
3. 将用户重定向到Portal认证页面
4. 用户在Portal页面输入认证信息
5. Portal服务器验证用户信息
6. 认证成功后，用户态程序通知内核模块放行该用户流量
7. 用户可以正常访问网络资源

## 6. 总结

Portal系统采用内核态与用户态协同工作的架构，实现了网络接入认证功能。系统支持本地Portal和外部Portal服务器，可以灵活配置认证策略和重定向规则。通过proc文件系统和socket通信实现内核态与用户态的数据交换，保证了系统的高效性和灵活性。

该系统的设计思路清晰，功能模块划分合理，是一个典型的网络安全认证系统实现。

# Portal功能用户态详细配置项分析

通过对Portal代码的分析，可以看出Portal功能在用户态主要有两类配置：本地Portal配置和外部Portal配置。这些配置项控制着Portal认证的行为和与外部服务器的交互方式。下面详细分析这些配置项。

## 1. 本地Portal配置项

本地Portal配置主要定义在`struct local_portal`结构体中，用于控制本地Portal服务器的行为：

```c
struct local_portal{
    char  localserver_ip[LEN32]; // 本地服务器ip地址
    int   jump_flag;             // 跳转标记，0表示不跳转，1表示跳转到之前的页面，2表示跳转到配置的url
    char  jump_url[100];         // 跳转url
    char  switch_flag;           // 开关标志
}
```

### 配置项详解：

1. **localserver_ip**：
   - 本地Portal服务器的IP地址
   - 用于接收未认证用户的HTTP请求并提供认证页面

2. **jump_flag**：
   - 认证成功后的跳转行为控制
   - 0：不跳转，保持在认证成功页面
   - 1：跳转到用户之前访问的页面
   - 2：跳转到指定的URL（由jump_url指定）

3. **jump_url**：
   - 当jump_flag为2时，认证成功后跳转的目标URL
   - 最大长度为100字符

4. **switch_flag**：
   - 本地Portal功能的开关标志
   - 控制是否启用本地Portal认证

### 配置修改枚举：

```c
enum CFG_LOCAL_PORTAL_ADD_MODIFY
{
    CFG_LOCAL_PORTAL_SERVER_IP    = 0,
    CFG_LOCAL_PORTAL_JUMP_FLAG    = 1,
    CFG_LOCAL_PORTAL_JUMP_URL     = 2,
    CFG_LOCAL_PORTAL_SWITCH       = 3,
    CFG_LOCAL_PORTAL_MAX          = 4
}
```

## 2. 外部Portal配置项

外部Portal配置分为两部分：模板配置和协议配置。

### 2.1 外部Portal模板配置

模板配置定义在`struct outside_portal_template`结构体中，用于定义与外部Portal服务器交互的参数：

```c
struct outside_portal_template{
    int   server_id;                  // portal服务器id号，最大支持三个
    char  server_name[50];            // 服务器名称
    char  redirect_url[100];          // 重定向url
    char  push_switch[5];             // 推送开关
    char  userip_name[50];            // 用户IP参数名称
    char  manufaturingno_name[100];   // 设备序列号名称
    char  server_ip[LEN32];           // 外部服务器ip地址
    int   peer_port;                  // 外部服务器portal端口号
    int   show_flag;                  // 显示标记
    char  device_ip_name[100];        // 设备IP参数名称
    char  user_access_addr_name[100]; // 用户接入地址参数名称
}
```

#### 配置项详解：

1. **server_id**：
   - 外部Portal服务器的ID号
   - 系统最多支持三个外部Portal服务器

2. **server_name**：
   - 外部Portal服务器的名称
   - 用于标识不同的Portal服务器

3. **redirect_url**：
   - 重定向URL模板
   - 用于构建重定向用户到外部Portal服务器的URL

4. **push_switch**：
   - 推送开关
   - 控制是否向外部Portal服务器推送用户信息

5. **userip_name**：
   - 用户IP参数的名称
   - 在URL中传递用户IP时使用的参数名

6. **manufaturingno_name**：
   - 设备序列号参数的名称
   - 在URL中传递设备序列号时使用的参数名

7. **server_ip**：
   - 外部Portal服务器的IP地址
   - 用于建立与外部Portal服务器的连接

8. **peer_port**：
   - 外部Portal服务器的端口号
   - 默认为2000端口

9. **show_flag**：
   - 显示标记
   - 控制是否在Web界面显示该服务器配置

10. **device_ip_name**：
    - 设备IP参数的名称
    - 在URL中传递设备IP时使用的参数名

11. **user_access_addr_name**：
    - 用户接入地址参数的名称
    - 在URL中传递用户接入地址时使用的参数名

#### 模板配置修改枚举：

```c
enum CFG_OUTSIDE_PORTAL_TEMPLATE_ADD_MODIFY
{
    CFG_OUTSIDE_PORTAL_SERVER_NAME         = 0,
    CFG_OUTSIDE_PORTAL_REDIRECT_URL        = 1,
    CFG_OUTSIDE_PORTAL_PUSH_SWITCH         = 2,
    CFG_OUTSIDE_PORTAL_USER_IP_NAME        = 3,
    CFG_OUTSIDE_PORTAL_AC_NUMBER           = 4,
    CFG_OUTSIDE_PORTAL_DEV_IP_NAME         = 5,
    CFG_OUTSIDE_PORTAL_USER_ACC_ADDR_NAME  = 6,
    CFG_OUTSIDE_PORTAL_TEMPLATE_MAX        = 7
}
```

### 2.2 外部Portal协议配置

协议配置定义在`struct outside_portal_protocol`结构体中，用于控制与外部Portal服务器的通信协议：

```c
struct outside_portal_protocol{
    char  server_ip[LEN32];            // 外部服务器ip地址
    int   peer_port;                   // 外部服务器portal端口号
    int   hearbeat_timeout_interval;   // 心跳超时间隔
    char  dev_ip[LEN32];               // 设备IP地址
    char  switch_sta;                  // 开关状态
}
```

#### 配置项详解：

1. **server_ip**：
   - 外部Portal服务器的IP地址
   - 用于建立与外部Portal服务器的通信

2. **peer_port**：
   - 外部Portal服务器的端口号
   - 用于建立与外部Portal服务器的通信

3. **hearbeat_timeout_interval**：
   - 心跳超时间隔
   - 控制与外部Portal服务器的心跳检测频率
   - 用于检测外部Portal服务器的可用性

4. **dev_ip**：
   - 设备IP地址
   - 用于向外部Portal服务器标识本设备

5. **switch_sta**：
   - 开关状态
   - 控制是否启用外部Portal协议

#### 协议配置修改枚举：

```c
enum CFG_OUTSIDE_PORTAL_PROTOCOL_MODIFY
{
    CFG_OUTSIDE_PORTAL_SERVER_IP           = 0,
    CFG_OUTSIDE_PORTAL_PEER_PORT           = 1,
    CFG_OUTSIDE_PORTAL_HEARTBEAT_INTERVAL  = 2,
    CFG_OUTSIDE_PORTAL_DEV_IP              = 3,
    CFG_OUTSIDE_PORTAL_SWTICH              = 4,
    CFG_OUTSIDE_PORTAL_PROTOCOL_MAX        = 5
}
```

## 3. 配置管理机制

Portal系统通过以下机制管理配置：

### 3.1 配置获取函数

```c
// 获取本地Portal配置
int get_local_portal_default(struct local_portal *localportal);

// 获取外部Portal模板配置
int get_outside_portal_template_default(struct outside_portal_template *outsideportal);

// 获取外部Portal协议配置
int get_outside_portal_protocol_default(struct outside_portal_protocol *outsideportal);
```

### 3.2 配置应用函数

```c
// 设置Portal Nginx配置文件
int set_portal_nginx_file();

// 应用Portal配置
void apply_portal_config(struct portal_send_msg *portal_msg_tmp);
```

### 3.3 配置控制接口

通过Unix域套接字实现的控制接口，允许其他进程修改Portal配置：

```c
int Control_Handle_portal()
{
    // 接收控制消息
    ret = read(sockfd, (unsigned char *)&portal_msg_tmp, sizeof(struct portal_send_msg));
    
    // 应用配置
    apply_portal_config(&portal_msg_tmp);
    
    // 返回处理结果
    write(sockfd, (unsigned char *)&portal_msg_tmp, sizeof(struct portal_send_msg));
}
```

## 4. 配置示例

### 4.1 本地Portal配置示例

```
localportalserverip=192.168.1.1
jumpflag=1
jumpurl=http://www.example.com
switch=1
```

### 4.2 外部Portal配置示例

```
servername=ExternalPortal
redirecturl=http://{server_ip}:{port}/portal?userip={userip}&acname={acname}
pushswitch=1
useripname=userip
acmunber=AC123456
server_ip=203.0.113.10
peer_port=2000
hearbeat_timeout_interval=60
```

## 5. 总结

Portal功能的用户态配置项主要分为本地Portal配置和外部Portal配置两大类。本地Portal配置控制内置Portal服务器的行为，而外部Portal配置则定义了与外部Portal服务器的交互方式。

这些配置项通过结构体定义，并使用枚举类型标识不同的配置修改操作。系统提供了获取默认配置、应用配置和控制接口等机制，使得Portal功能可以灵活配置，满足不同网络环境的需求。

通过这些详细的配置项，Portal系统可以实现多种认证场景，包括本地认证和与外部Portal服务器的集成，为网络接入控制提供了强大的支持。

# Portal内核态详细配置项分析

通过对Portal内核模块代码的分析，可以看出Portal功能在内核态主要有以下几类配置项，这些配置项控制着Portal认证的行为和网络流量处理方式。

## 1. 开关配置项

内核态Portal模块提供了两个主要的开关配置项，通过proc文件系统暴露给用户态：

### 1.1 本地Portal开关

```c
static int local_portal_switch; // 本地Portal开关状态

// 读取本地Portal开关状态
static int read_tos_local_portal_switch(char *buffer, char **start, off_t offset, int length, int *eof,
    void *data)
{
    int len = 0;
    off_t begin = 0;

    len = sprintf(buffer, "%02x\n", local_portal_switch);

    *start = buffer + (offset - begin);    
    len -= (offset - begin);    
    if (len > length)
        len = length;
    return len;
}

// 设置本地Portal开关状态
static int write_tos_local_portal_switch(struct file *file, const char *buf,
                        unsigned long count, void *data)
{    
    char tmp = 0;
    int ret = count;
    tmp = simple_strtol(buf,NULL,10);
    local_portal_switch = tmp;
    
    return ret; 
}
```

### 1.2 外部Portal开关

```c
static int outside_portal_switch; // 外部Portal开关状态

// 读取外部Portal开关状态
static int read_tos_outeside_portal_switch(char *buffer, char **start, off_t offset, int length, int *eof,
    void *data)
{
    int len = 0;
    off_t begin = 0;

    len = sprintf(buffer, "%02x\n", outside_portal_switch);

    *start = buffer + (offset - begin); 
    len -= (offset - begin);    
    if (len > length)
        len = length;
    return len;
}

// 设置外部Portal开关状态
static int write_tos_outeside_portal_switch(struct file *file, const char *buf,
                        unsigned long count, void *data)
{    
    char tmp = 0;
    int ret = count;
    tmp = simple_strtol(buf,NULL,10);
    outside_portal_switch = tmp;
    
    return ret; 
}
```

## 2. 会话私有数据配置

内核态Portal模块为每个会话维护私有数据，用于控制认证状态和行为：

```c
struct tos_portal_session_private {
    int portal;       // Portal类型标识
    int permit_flag;  // 许可标志，表示是否通过认证
    int nginx_port;   // Nginx端口号，用于重定向
}
```

这些私有数据通过会话管理接口进行注册和管理：

```c
// 注册会话私有数据处理函数
portal_ts_priv_id = TS_register_priv_fun(tos_portal_ts_priv_handle);

// 会话私有数据处理函数
void *tos_portal_ts_priv_handle(TS_HANDLE h)
{
    struct tos_portal_session_private *p = NULL;
    
    TS_get_private(h, portal_ts_priv_id, (void **)&p);
    TS_detach_private(h, portal_ts_priv_id);
    
    if(!p) 
        return NULL;

    if(p)
        TOS_FREE(p);
    
    return NULL;
}
```

## 3. 外部Portal模板配置

内核态也维护了外部Portal模板配置，与用户态的配置结构相对应：

```c
struct outside_portal_template{
    int   server_id;                  // portal服务器id号
    char  server_name[50];            // 服务器名称
    char  redirect_url[100];          // 重定向url
    char  push_switch[5];             // 推送开关
    char  userip_name[50];            // 用户IP参数名称
    char  manufaturingno_name[100];   // 设备序列号名称
    char  server_ip[32];              // 外部服务器ip地址
    int   peer_port;                  // 外部服务器portal端口号
    int   show_flag;                  // 显示标记
    char  device_ip_name[100];        // 设备IP参数名称
    char  user_access_addr_name[100]; // 用户接入地址参数名称
}
```

内核提供了根据服务器ID获取配置的函数：

```c
struct outside_portal_template * get_outside_portal_config_by_serverid(int server_id)
{
    struct tos_obj_head *obj = NULL;
    struct list_head *list, *head;
    struct outside_portal_template *outside_portal_tmp = NULL;
    int num = 0;

    num = get_obj_count_typeA(OBJ_TYPE_OUTSIDE_TEMPLATE_PORTAL);

    if (num == 0)
            return NULL;

    if ((head = get_objlist_by_type(OBJ_TYPE_OUTSIDE_TEMPLATE_PORTAL)) == NULL)
        return NULL;
    list_for_each(list, head) {
        obj = (struct tos_obj_head *)list;
        outside_portal_tmp = (struct outside_portal_template *)obj->data;
        if(server_id == outside_portal_tmp->server_id)
        {
            return outside_portal_tmp;
        }
    }
    return NULL;
}
```

## 4. 认证策略配置

内核态Portal模块实现了认证策略匹配功能，用于决定哪些流量需要进行Portal认证：

```c
__u32 portal_auth_policy_match(void *obj, void *cb)
{ 
    struct tos_obj_head *pobj = obj;
    struct auth_policy  *auth_policy = pobj->data;
    struct tos_obj_head *refer_obj;
    struct tos_buff *tb = cb;
    struct net_device *in = tb->idev;
    int         match_ret = 0;
    int         i = 0;

    // 匹配区域
    if(GET_REFER_OBJ_NUMBER_PORTAL(SAREA, auth_policy) > 0){
        for(i=0; i < GET_REFER_OBJ_NUMBER_PORTAL(SAREA, auth_policy); i++){        
            refer_obj = (struct tos_obj_head *)pobj->refer_block[auth_policy->b_sarea+i];
            if(refer_obj->type == OBJ_TYPE_AREA){
                // 区域匹配逻辑
            }
        }
    }
    
    // 其他匹配条件...
    
    return match_ret;
}
```

## 5. 会话处理配置

内核态Portal模块通过注册会话处理钩子函数，实现对网络流量的拦截和处理：

```c
// 会话处理操作结构
static struct se_ops portal_postsession_ops = {
    .name = PORTAL_POSTSESSION_NAME,
    .priority = 0,
    .hook = portal_postsession_hook,
};

// 注册会话处理钩子
ret = se_register(&portal_postsession_ops);
```

## 6. proc文件系统配置

内核态Portal模块通过proc文件系统提供了配置接口：

```c
// 创建本地Portal开关proc文件
local_proc_entry =
    create_proc_entry("portal_local_switch", S_IFREG | S_IRUGO | S_IWUGO,
              proc_tos);
if (!local_proc_entry) {
    printk("create porc file  portal_switch fail\n");
    goto clean;
}

local_proc_entry->owner = THIS_MODULE;
local_proc_entry->read_proc = read_tos_local_portal_switch;
local_proc_entry->write_proc = write_tos_local_portal_switch;

// 创建外部Portal开关proc文件
outside_proc_entry =
    create_proc_entry("portal_outside_switch", S_IFREG | S_IRUGO | S_IWUGO,
              proc_tos);
if (!outside_proc_entry) {
    printk("create porc file  portal_outside_switch fail\n");
    goto clean;
}

outside_proc_entry->owner = THIS_MODULE;
outside_proc_entry->read_proc = read_tos_outeside_portal_switch;
outside_proc_entry->write_proc = write_tos_outeside_portal_switch;
```

## 7. 初始化和清理配置

内核态Portal模块在初始化和清理时进行相关配置：

```c
// 初始化函数
static int __init portal_init_module(void)
{
    // 注册会话处理钩子
    ret = se_register(&portal_postsession_ops);
    
    // 注册认证策略匹配函数
    ret = tos_register_func(OBJ_TYPE_AUTH_POLICY, CMD_TYPE_MATCH, (conf_func_t *)portal_auth_policy_match);

    // 注册会话私有数据处理函数
    portal_ts_priv_id = TS_register_priv_fun(tos_portal_ts_priv_handle);
    
    // 创建proc文件
    // ...
    
    // 记录日志
    tos_sys_log(LOG_INFO,"PORTAL","Load PORTAL module ok!",RESULT_SUCC,"NULL");
    
    return 0;
}

// 清理函数
static void __exit portal_cleanup_module(void)
{
    // 移除proc文件
    remove_proc_entry("tos/portal_outside_switch", NULL);
    remove_proc_entry("tos/portal_local_switch", NULL);
    
    // 注销会话处理钩子
    se_unregister(&portal_postsession_ops);
    
    // 注销认证策略匹配函数
    tos_unregister_func(OBJ_TYPE_AUTH_POLICY, CMD_TYPE_MATCH);
    
    // 注销会话私有数据处理函数
    TS_unregister_priv_fun(portal_ts_priv_id);

    // 记录日志
    tos_sys_log(LOG_INFO,"PORTAL","Remove PORTAL module ok!",RESULT_SUCC,"NULL");
}
```

## 8. 辅助函数配置

内核态Portal模块还提供了一些辅助函数用于配置处理：

```c
// IP地址转换为点分十进制字符串
static __inline__ char *u_to_dot(__u32 ip)
{
    static char buf[32];
    unsigned char *p=(unsigned char *)&ip;
    sprintf(buf,"%d.%d.%d.%d",p[0],p[1],p[2],p[3]);

    return buf;
}

// MAC地址转换为字符串
static __inline__ int mac2str(unsigned char *mac, char *str)
{
    sprintf(str,"%02X:%02X:%02X:%02X:%02X:%02X",*mac,
                        *(mac+sizeof(unsigned char)),
                        *(mac+2*sizeof(unsigned char)),
                        *(mac+3*sizeof(unsigned char)),
                        *(mac+4*sizeof(unsigned char)),
                        *(mac+5*sizeof(unsigned char)));
    return 0;
}
```

## 总结

Portal内核态配置项主要包括：

1. **开关配置**：通过proc文件系统控制本地Portal和外部Portal的启用状态
2. **会话私有数据配置**：维护每个会话的Portal认证状态
3. **外部Portal模板配置**：存储外部Portal服务器的配置信息
4. **认证策略配置**：决定哪些流量需要进行Portal认证
5. **会话处理配置**：通过钩子函数拦截和处理网络流量
6. **proc文件系统配置**：提供用户态与内核态交互的接口
7. **初始化和清理配置**：模块加载和卸载时的配置处理
8. **辅助函数配置**：提供配置处理所需的辅助功能

这些配置项共同构成了Portal内核态模块的完整功能，实现了网络流量的拦截、认证状态的维护以及与用户态的交互。

# Portal内核模块的匹配和认证流程详细分析

通过对Portal内核模块代码的分析，我们可以详细了解Portal认证的匹配和认证流程。Portal作为一种网络接入认证机制，其内核模块主要负责对网络流量进行拦截、匹配和认证处理。

## 1. Portal内核模块的整体架构

Portal内核模块主要通过以下几个关键组件实现匹配和认证流程：

1. 会话处理钩子（Session Hook）
2. 认证策略匹配（Auth Policy Match）
3. 会话私有数据管理（Session Private Data）
4. 重定向处理（Redirection）

## 2. 认证策略匹配流程

### 2.1 认证策略结构

认证策略在内核中通过`struct auth_policy`结构体定义：

```c
struct auth_policy{
    __u16 b_sarea;      // 区域引用块起始位置
    __u16 b_src;        // 源地址引用块起始位置
    __u16 refer;        // 引用计数
    int portal_num;     // Portal服务器编号
    int hit_count;      // 命中计数
}
```

### 2.2 认证策略匹配函数

认证策略匹配通过`portal_auth_policy_match`函数实现：

```c
__u32 portal_auth_policy_match(void *obj, void *cb)
{ 
    struct tos_obj_head *pobj = obj;
    struct auth_policy *auth_policy = pobj->data;
    struct tos_obj_head *refer_obj;
    struct tos_buff *tb = cb;
    struct net_device *in = tb->idev;
    int match_ret = 0;
    int i = 0;

    // 1. 匹配区域
    if(GET_REFER_OBJ_NUMBER_PORTAL(SAREA, auth_policy) > 0){
        for(i=0; i < GET_REFER_OBJ_NUMBER_PORTAL(SAREA, auth_policy); i++){        
            refer_obj = (struct tos_obj_head *)pobj->refer_block[auth_policy->b_sarea+i];
            if(refer_obj->type == OBJ_TYPE_AREA){
                match_ret = is_dev_in_area(in, refer_obj);
            }else match_ret = 0;

            if(match_ret) break;
        }

        if(!match_ret) return 0;
    }

    // 2. 匹配源地址
    if(GET_REFER_OBJ_NUMBER_PORTAL(SRC, auth_policy) > 0){
        for(i=0; i<GET_REFER_OBJ_NUMBER_PORTAL(SRC, auth_policy); i++){        
            refer_obj = (struct tos_obj_head *)pobj->refer_block[auth_policy->b_src+i];
            if(refer_obj->type == OBJ_TYPE_ADDRESS){
                match_ret = tos_match_address(refer_obj, cb, DIR_SRC, SRC_IP(tb));
            }else match_ret = 0;

            if(match_ret) break;
        }

        if(!match_ret) return 0;
    }

    // 3. 匹配成功，返回Portal服务器编号
    return auth_policy->portal_num;
}
```

### 2.3 地址匹配函数

地址匹配支持三种类型：主机地址、子网地址和地址范围，分别通过以下函数实现：

```c
// 主机地址匹配
static __u32 tos_match_address_host(void *obj, void *cb, int direction, __u32 ip)
{
    struct tos_address *pa = obj;
    struct tos_buff *tb = cb;
    __u32 addr = 0;
    __u8 any_mac[ETH_ALEN] = {0x00,0x00,0x00,0x00,0x00,0x00};
    int i = 0;
    
    // 获取地址
    if (direction == DIR_SRC) {
        addr = ntohl(ip);
    }
    if (direction == DIR_DST) {
        addr = ntohl(ip);
    }

    // MAC地址匹配
    if (pa->n_addr == 0) {
        if (memcmp(pa->mac, any_mac, ETH_ALEN)) {
            if(memcmp(SRC_MAC(tb), pa->mac, ETH_ALEN)) {
                return 1;
            }
        }
    } else {
        // IP地址匹配
        for (i = 0; i < pa->n_addr; i++) {
            if (addr == pa->addr[i]) {
                return 1;
            }
        }
    }
    return 0;
}

// 子网地址匹配
static __u32 tos_match_address_subnet(void *obj, void *cb, int direction, __u32 ip)
{
    struct tos_address *pa = obj;
    int i = 0;
    __u32 addr = 0;
    
    // 获取地址
    if (direction == DIR_SRC) {
        addr = ntohl(ip);
    }
    if (direction == DIR_DST) {
        addr = ntohl(ip);
    }
    
    // 子网匹配
    if ((pa->ip1 & pa->ip2) != (addr & pa->ip2)) {
        return 0;
    }

    // 排除例外地址
    for (i = 0; i < pa->n_addr; i++) {
        if (addr == pa->addr[i]) {
            return 0;
        }
    }    
    return 1;
}

// 地址范围匹配
static __u32 tos_match_address_range(void *obj, void *cb, int direction, __u32 ip)
{
    struct tos_address *pa = obj;
    int i = 0;
    __u32 addr = 0;
    
    // 获取地址
    if (direction == DIR_SRC) {
        addr = ntohl(ip);
    }
    if (direction == DIR_DST) {
        addr = ntohl(ip);
    }

    // 范围匹配
    if ((pa->ip1 > addr) || (pa->ip2 < addr)) {
        return 0;
    }

    // 排除例外地址
    for (i = 0; i < pa->n_addr; i++) {
        if (addr == pa->addr[i]) {
            return 0;
        }
    }
    return 1;
}
```

## 3. 认证处理流程

### 3.1 会话处理函数

Portal认证处理主要通过`portal_auth_policy_handle`函数实现：

```c
int portal_auth_policy_handle(struct tos_buff **ptb) {
    struct tos_obj_head * obj = NULL;    
    struct auth_policy * auth_policy = NULL;
    char protocol[10];
    __u32 sport = 0, dport = 0;
    struct tos_buff *tb = *ptb;
    unsigned char v4_addr[64] = {0};
    unsigned char v4_addr_peer[64] = {0};
    __u32 id = 0;
    int ret = 0;
    int policy_num = 0;
    struct outside_portal_template *outside_portal = NULL;
    struct tos_portal_session_private *data;
    struct tos_portal_session_private *data1;
    
    // 1. 检查是否有认证策略
    policy_num = get_obj_count_typeA(OBJ_TYPE_AUTH_POLICY);
    if (policy_num == 0)
        return TOS_ACCEPT;    
    
    // 2. 协议检查和端口提取
    if(IP_PROTO(tb) != IPPROTO_TCP && IP_PROTO(tb) != IPPROTO_ICMP) {
        if (IP_PROTO(tb) == IPPROTO_UDP) {
            if (tb->l4h.udph) {
                sport = ntohs(SRC_UPORT(tb));
                dport = ntohs(DST_UPORT(tb));
            } else {
                return TOS_ACCEPT;
            }
            
            // DNS请求放行
            if (dport == 53 || sport == 53)
                return TOS_ACCEPT;
        } else {
            return TOS_ACCEPT;
        }
    }
    
    // 3. 获取会话私有数据
    data = TS_get_or_create_private(tb->ts, portal_ts_priv_id, sizeof(struct tos_portal_session_private));
    if (!data) {
        return TOS_ACCEPT;
    }
    
    // 4. 检查是否已认证
    if (data->permit_flag == 1) {
        return TOS_ACCEPT;
    }
    
    // 5. 匹配认证策略
    id = tos_match_obj(OBJ_TYPE_AUTH_POLICY, CMD_TYPE_MATCH, tb);
    if (id == 0) {
        return TOS_ACCEPT;
    }
    
    // 6. 获取认证策略对象
    obj = get_obj_by_id(id);
    if (!obj) {
        return TOS_ACCEPT;
    }
    auth_policy = (struct auth_policy *)obj->data;
    
    // 7. 处理HTTP请求
    if (IP_PROTO(tb) == IPPROTO_TCP && tb->l4h.tcph) {
        sport = ntohs(SRC_TPORT(tb));
        dport = ntohs(DST_TPORT(tb));
        
        // 处理HTTP请求（80端口）
        if (dport == 80) {
            // 本地Portal处理
            if (auth_policy->portal_num == 0 && local_portal_switch) {
                data->portal = 0;
                data->permit_flag = 0;
                data->nginx_port = 8080;
                
                // 重定向到本地Portal服务器
                return TOS_REDIRECT;
            }
            
            // 外部Portal处理
            if (auth_policy->portal_num > 0 && outside_portal_switch) {
                outside_portal = get_outside_portal_config_by_serverid(auth_policy->portal_num);
                if (outside_portal) {
                    data->portal = auth_policy->portal_num;
                    data->permit_flag = 0;
                    
                    // 重定向到外部Portal服务器
                    return TOS_REDIRECT;
                }
            }
        }
    }
    
    // 8. 处理HTTPS请求
    if (IP_PROTO(tb) == IPPROTO_TCP && tb->l4h.tcph) {
        if (dport == 443) {
            // HTTPS请求处理
            return TOS_DROP;
        }
    }
    
    // 9. 默认处理
    return TOS_DROP;
}
```

### 3.2 会话私有数据结构

会话私有数据通过`struct tos_portal_session_private`结构体定义：

```c
struct tos_portal_session_private {
    int portal;       // Portal类型标识（0表示本地Portal，>0表示外部Portal服务器ID）
    int permit_flag;  // 许可标志（0表示未认证，1表示已认证）
    int nginx_port;   // Nginx端口号，用于本地Portal重定向
}
```

### 3.3 会话私有数据处理函数

会话私有数据处理通过`tos_portal_ts_priv_handle`函数实现：

```c
void *tos_portal_ts_priv_handle(TS_HANDLE h)
{
    struct tos_portal_session_private *p = NULL;
    
    TS_get_private(h, portal_ts_priv_id, (void **)&p);
    TS_detach_private(h, portal_ts_priv_id);
    
    if(!p) 
        return NULL;

    if(p)
        TOS_FREE(p);
    
    return NULL;
}
```

## 4. 外部Portal配置获取

外部Portal配置通过`get_outside_portal_config_by_serverid`函数获取：

```c
struct outside_portal_template * get_outside_portal_config_by_serverid(int server_id)
{
    struct tos_obj_head *obj = NULL;
    struct list_head *list, *head;
    struct outside_portal_template *outside_portal_tmp = NULL;
    int num = 0;

    num = get_obj_count_typeA(OBJ_TYPE_OUTSIDE_TEMPLATE_PORTAL);

    if (num == 0)
            return NULL;

    if ((head = get_objlist_by_type(OBJ_TYPE_OUTSIDE_TEMPLATE_PORTAL)) == NULL)
        return NULL;
    list_for_each(list, head) {
        obj = (struct tos_obj_head *)list;
        outside_portal_tmp = (struct outside_portal_template *)obj->data;
        if(server_id == outside_portal_tmp->server_id)
        {
            return outside_portal_tmp;
        }
    }
    return NULL;
}
```

## 5. 匹配和认证流程图

```
┌─────────────────────────────────┐
│ 数据包进入内核                    │
└───────────────┬─────────────────┘
                ▼
┌─────────────────────────────────┐
│ 会话处理钩子捕获数据包             │
└───────────────┬─────────────────┘
                ▼
┌─────────────────────────────────┐     ┌─────────────────┐
│ 检查是否有认证策略                 ├─否──► 放行数据包       │
└───────────────┬─────────────────┘     └─────────────────┘
                │ 是
                ▼
┌─────────────────────────────────┐     ┌─────────────────┐
│ 检查协议类型                      ├─非TCP/UDP► 放行数据包   │
└───────────────┬─────────────────┘     └─────────────────┘
                │ TCP/UDP
                ▼
┌─────────────────────────────────┐     ┌─────────────────┐
│ 获取会话私有数据                   ├─失败► 放行数据包       │
└───────────────┬─────────────────┘     └─────────────────┘
                │ 成功
                ▼
┌─────────────────────────────────┐     ┌─────────────────┐
│ 检查是否已认证                    ├─已认证► 放行数据包      │
└───────────────┬─────────────────┘     └─────────────────┘
                │ 未认证
                ▼
┌─────────────────────────────────┐     ┌─────────────────┐
│ 匹配认证策略                      ├─不匹配► 放行数据包      │
└───────────────┬─────────────────┘     └─────────────────┘
                │ 匹配
                ▼
┌─────────────────────────────────┐
│ 获取认证策略对象                   │
└───────────────┬─────────────────┘
                │
                ▼
┌─────────────────────────────────┐     ┌─────────────────┐
│ 检查是否为HTTP请求(80端口)         ├─是──► 处理HTTP请求     │
└───────────────┬─────────────────┘     │                 │
                │ 否                     │ ┌─────────────┐ │
                │                       │ │本地Portal处理│ │
                │                       │ └──────┬──────┘ │
                │                       │        │        │
                │                       │ ┌─────────────┐ │
                │                       │ │外部Portal处理│ │
                │                       │ └──────┬──────┘ │
                │                       │        │        │
                │                       │ ┌─────────────┐ │
                │                       └─│  重定向请求  │◄┘
                │                         └─────────────┘
                ▼
┌─────────────────────────────────┐     ┌─────────────────┐
│ 检查是否为HTTPS请求(443端口)       ├─是──► 丢弃数据包       │
└───────────────┬─────────────────┘     └─────────────────┘
                │ 否
                ▼
┌─────────────────────────────────┐
│ 默认处理：丢弃数据包               │
└─────────────────────────────────┘
```

## 6. 认证策略匹配流程图

```
┌─────────────────────────────────┐
│ 开始认证策略匹配                   │
└───────────────┬─────────────────┘
                ▼
┌─────────────────────────────────┐     ┌─────────────────┐
│ 检查是否有区域匹配条件              ├─否──► 跳过区域匹配     │
└───────────────┬─────────────────┘     └────────┬────────┘
                │ 是                              │
                ▼                                 │
┌─────────────────────────────────┐              │
│ 遍历区域引用块                    │              │
└───────────────┬─────────────────┘              │
                │                                 │
                ▼                                 │
┌─────────────────────────────────┐              │
│ 检查设备是否在区域内               │              │
└───────────────┬─────────────────┘              │
                │                                 │
                ▼                                 │
┌─────────────────────────────────┐     ┌────────▼────────┐
│ 区域匹配结果                      ├─不匹配► 返回不匹配      │
└───────────────┬─────────────────┘     └─────────────────┘
                │ 匹配
                ▼
┌─────────────────────────────────┐     ┌─────────────────┐
│ 检查是否有源地址匹配条件            ├─否──► 跳过源地址匹配   │
└───────────────┬─────────────────┘     └────────┬────────┘
                │ 是                              │
                ▼                                 │
┌─────────────────────────────────┐              │
│ 遍历源地址引用块                   │              │
└───────────────┬─────────────────┘              │
                │                                 │
                ▼                                 │
┌─────────────────────────────────┐              │
│ 根据地址类型进行匹配               │              │
│ - 主机地址匹配                    │              │
│ - 子网地址匹配                    │              │
│ - 地址范围匹配                    │              │
└───────────────┬─────────────────┘              │
                │                                 │
                ▼                                 │
┌─────────────────────────────────┐     ┌────────▼────────┐
│ 源地址匹配结果                    ├─不匹配► 返回不匹配      │
└───────────────┬─────────────────┘     └─────────────────┘
                │ 匹配
                ▼
┌─────────────────────────────────┐
│ 返回Portal服务器编号               │
└─────────────────────────────────┘
```

## 7. 总结

Portal内核模块的匹配和认证流程主要包括以下几个步骤：

1. **数据包捕获**：通过会话处理钩子捕获网络数据包
2. **认证状态检查**：检查会话私有数据中的认证状态
3. **策略匹配**：根据区域和源地址等条件匹配认证策略
4. **认证处理**：
   - 对于HTTP请求，根据Portal类型（本地或外部）进行重定向
   - 对于HTTPS请求，直接丢弃
   - 对于其他请求，根据配置决定是放行还是丢弃
5. **认证完成后**：更新会话私有数据中的认证状态

通过这种机制，Portal内核模块能够有效地拦截未认证用户的网络访问请求，并将其重定向到认证页面，实现网络接入控制。

# Portal内核层代码错误分析

通过对Portal内核模块代码的分析，我发现了以下几个潜在的错误和问题：

## 1. 内存泄漏问题

在`tos_portal_ts_priv_handle`函数中存在潜在的内存泄漏：

```c
void *tos_portal_ts_priv_handle(TS_HANDLE h)
{
    struct tos_portal_session_private *p = NULL;
    
    TS_get_private(h, portal_ts_priv_id, (void **)&p);
    TS_detach_private(h, portal_ts_priv_id);
    
    if(!p) 
        return NULL;

    if(p)
        TOS_FREE(p);
    
    return NULL;
}
```

这里的问题是先进行了`if(!p) return NULL;`的判断，然后又进行了`if(p) TOS_FREE(p);`的判断。第一个判断后如果p为NULL就直接返回了，所以第二个判断是多余的，应该直接释放内存。

## 2. 错误的地址匹配逻辑

在`tos_match_address_subnet`函数中：

```c
static __u32 tos_match_address_subnet(void *obj, void *cb, int direction, __u32 ip)
{
    struct tos_address *pa = obj;
    int i = 0;
    __u32 addr = 0;
    
    if (direction == DIR_SRC) {
        addr = ntohl(ip);
    }
    if (direction == DIR_DST) {
        addr = ntohl(ip);
    }
    
    if ((pa->ip1 & pa->ip2) != (addr & pa->ip2)) {
        return 0;
    }

    for (i = 0; i < pa->n_addr; i++) {
        if (addr == pa->addr[i]) {
            return 0;
        }
    }    
    return 1;
}
```

这里的问题是对于源地址和目标地址的处理完全相同，没有区分不同方向的处理逻辑。应该使用`if...else if...`结构而不是两个独立的`if`语句。

## 3. 资源清理不完整

在`portal_init_module`函数的错误处理部分：

```c
clean:
    se_unregister(&portal_postsession_ops);     
    tos_unregister_func(OBJ_TYPE_AUTH_POLICY, CMD_TYPE_MATCH);
    TS_unregister_priv_fun(portal_ts_priv_id);
    if (outside_proc_entry)
        remove_proc_entry("tos/portal_outside_switch", NULL);
    if (local_proc_entry)
        remove_proc_entry("tos/portal_local_switch", NULL);
    return -1;
}
```

这里的问题是proc文件路径不正确。应该是`"portal_outside_switch"`和`"portal_local_switch"`，而不是带有`"tos/"`前缀的路径，因为在创建proc文件时已经指定了父目录为`proc_tos`。

## 4. DNS处理逻辑不完善

在`portal_auth_policy_handle`函数中：

```c
if (IP_PROTO(tb) == IPPROTO_UDP) {
    if (tb->l4h.udph) {
        sport = ntohs(SRC_UPORT(tb));
        dport = ntohs(DST_UPORT(tb));
    } else {
        return TOS_ACCEPT;
    }
    if (sport == 53 || dport == 53) /* for dns packet port */
        goto NEXT;
    else
        return TOS_ACCEPT;
}
```

这里使用了`goto NEXT;`跳转，但这种处理方式可能导致代码逻辑混乱。更好的做法是重构代码，避免使用goto语句。

## 5. 错误的proc文件路径处理

在模块清理函数中：

```c
static void __exit portal_cleanup_module(void)
{
    // 移除proc文件
    remove_proc_entry("tos/portal_outside_switch", NULL);
    remove_proc_entry("tos/portal_local_switch", NULL);
    
    // 注销会话处理钩子
    se_unregister(&portal_postsession_ops);
    
    // 注销认证策略匹配函数
    tos_unregister_func(OBJ_TYPE_AUTH_POLICY, CMD_TYPE_MATCH);
    
    // 注销会话私有数据处理函数
    TS_unregister_priv_fun(portal_ts_priv_id);

    // 记录日志
    tos_sys_log(LOG_INFO,"PORTAL","Remove PORTAL module ok!",RESULT_SUCC,"NULL");
}
```

与初始化函数中的错误类似，这里的proc文件路径也不正确。

## 6. 缺少错误处理

在`portal_auth_policy_match`函数中：

```c
__u32 portal_auth_policy_match(void *obj, void *cb)
{ 
    struct tos_obj_head *pobj = obj;
    struct auth_policy *auth_policy = pobj->data;
    struct tos_obj_head *refer_obj;
    struct tos_buff *tb = cb;
    struct net_device *in = tb->idev;
    int match_ret = 0;
    int i = 0;

    // 匹配区域...
    // 匹配源地址...

    return match_ret;
}
```

函数最后返回`match_ret`，但如果所有匹配条件都通过，应该返回`auth_policy->portal_num`而不是简单的匹配结果。这可能导致认证策略无法正确应用。

## 7. 结构体定义不一致

内核模块和用户态程序中的结构体定义不一致，例如：

内核模块中：
```c
struct outside_portal_template{
    int   server_id;
    char  server_name[50];
    char  redirect_url[100];
    char  push_switch[5];
    char  userip_name[50];
    char  manufaturingno_name[100];
    char  server_ip[32];
    int   peer_port;
    int   show_flag;
    char  device_ip_name[100];
    char  user_access_addr_name[100];    
}
```

用户态程序中：
```c
struct outside_portal_template{
    int   server_id;//portal服务器id号，最大支持三个 
    char  server_name[50];
    char  redirect_url[100];//重定向url    
    char  push_switch[5];//推送开关
    char  userip_name[50];
    char  manufaturingno_name[100];//设备序列号名称    
    char  server_ip[LEN32]; // 外部服务器ip地址
    int   peer_port;//外部服务器portal端口号
    int   show_flag;//显示标记
    char  device_ip_name[100];
    char  user_access_addr_name[100];    
}
```

注意`server_ip`字段的长度定义不同：一个是`[32]`，另一个是`[LEN32]`。这可能导致内存访问越界或数据截断。

## 8. 重复的结构体定义

在用户态代码中，`outisde_portal_user_info`结构体被重复定义：

```c
// 在portal_proccess.h中
struct outisde_portal_user_info{
    u32 userip;
    //char username[253];
}

// 在portal_server.h中
struct outisde_portal_user_info{
    u32 userip;
    //char username[253];
}
```

这种重复定义可能导致编译错误或不一致的行为。

## 9. 缺少必要的错误检查

在`get_outside_portal_config_by_serverid`函数中：

```c
struct outside_portal_template * get_outside_portal_config_by_serverid(int server_id)
{
    struct tos_obj_head *obj = NULL;
    struct list_head *list, *head;
    struct outside_portal_template *outside_portal_tmp = NULL;
    int num = 0;

    num = get_obj_count_typeA(OBJ_TYPE_OUTSIDE_TEMPLATE_PORTAL);

    if (num == 0)
            return NULL;

    if ((head = get_objlist_by_type(OBJ_TYPE_OUTSIDE_TEMPLATE_PORTAL)) == NULL)
        return NULL;
    list_for_each(list, head) {
        obj = (struct tos_obj_head *)list;
        outside_portal_tmp = (struct outside_portal_template *)obj->data;
        if(server_id == outside_portal_tmp->server_id)
        {
            return outside_portal_tmp;
        }
    }
    return NULL;
}
```

这里没有检查`obj->data`是否为NULL，可能导致空指针解引用。

## 10. 拼写错误

在proc文件创建和读写函数中：

```c
local_proc_entry->read_proc = read_tos_local_portal_switch;
local_proc_entry->write_proc = write_tos_local_portal_switch;

outside_proc_entry->read_proc = read_tos_outeside_portal_switch;
outside_proc_entry->write_proc = write_tos_outeside_portal_switch;
```

注意`outeside`的拼写错误，应该是`outside`。这种拼写错误可能导致函数名不一致，引起编译错误。

## 总结

Portal内核层代码存在多个潜在问题，包括内存泄漏、逻辑错误、资源清理不完整、结构体定义不一致等。这些问题可能导致系统不稳定、内存泄漏或功能异常。建议进行代码审查和修复，特别是关注内存管理、错误处理和结构体定义的一致性。

# Portal应用程序分析

通过对当前文件夹中的代码进行分析，我可以看出这是一个Portal认证系统的用户态应用程序部分。下面我将详细分析这个进程的作用和功能。

## 1. Portal进程的主要作用

Portal进程是一个网络认证系统的用户态组件，主要负责以下功能：

1. **认证流程处理**：处理用户认证请求，包括Challenge请求、认证请求和注销请求
2. **与Portal服务器通信**：与外部Portal服务器进行通信，交换认证信息
3. **配置管理**：管理本地Portal和外部Portal的配置信息
4. **事件处理**：处理定时事件，如心跳超时等
5. **重定向配置**：生成Nginx配置文件，用于HTTP请求重定向到认证页面

## 2. 主要功能模块分析

### 2.1 认证协议处理

Portal进程实现了Portal认证协议，主要包括以下消息类型：

- **REQ_CHALLENGE/ACK_CHALLENGE**：挑战请求和响应
- **REQ_AUTH/ACK_AUTH**：认证请求和响应
- **REQ_LOGOUT/ACK_LOGOUT**：注销请求和响应
- **REQ_INFO/ACK_INFO**：信息请求和响应

这些消息处理在`portal_msg_handle`函数中实现，例如Challenge处理：

```c
case REQ_CHALLENGE:
    memcpy(&chapmsg, msg, sizeof(struct portal_msg));
    chapmsg.type = ACK_CHALLENGE;
    if(chapmsg.chap_flag == 0)
    {
        chapmsg.attrnum = 1;
        chapmsg.attrtype = ATTR_CHALLENGE;
        chapmsg.attrlen = 16;
        srand((int)time(0));
        ReqID_int = (int) (1 + rand());
        chapmsg.reqid = ReqID_int & 0x0000ffff;
        ret = getChallenge(chapmsg.challenge);
        // ...
    }
    sendto(portal_fd, &chapmsg, sizeof(struct portal_chap_msg), 0, 
           (struct sockaddr *) &from, fromlen);
    break;
```

### 2.2 事件处理机制

Portal进程实现了一个事件处理机制，用于处理定时事件，如心跳超时等。主要通过以下函数实现：

- `init_event`：初始化事件列表
- `Add_Event`：添加事件
- `Delete_Event`：删除事件
- `Next_Event`：获取下一个要触发的事件
- `Event_Process`：处理事件

事件处理的核心逻辑在`Event_Process`函数中：

```c
void Event_Process()
{
    // ...
    tm = time(NULL);
    if(tm < ev->ev_timer)
    {
        return;
    }
    
    // ...
    type = ev->ev_type;
    if(type == CFG_PORTAL_HEARTBEAT_TIMEOUT)
    {
        timer = portal_msg.msg.outside_portal.hearbeat_timeout_interval;
    }
    
    // ...
    Delete_Event(name, &portal_evlist);
    Add_Event(name, type, timer, (EVENT_HANDLE)(thread_arg->handle), &portal_evlist); 
    
    ((EVENT_HANDLE)(thread_arg->handle))();
    // ...
}
```

### 2.3 配置管理

Portal进程管理两种类型的Portal配置：

1. **本地Portal配置**：在本地设备上运行的Portal服务
2. **外部Portal配置**：外部的Portal服务器配置

配置管理主要通过以下函数实现：

- `set_nginx_file`：生成Nginx配置文件，用于HTTP请求重定向
- `set_portal_nginx_file`：为所有外部Portal服务器生成Nginx配置
- `get_outside_portal_protocol_default`：获取外部Portal协议的默认配置

### 2.4 进程管理

Portal进程实现了守护进程的功能，主要通过以下函数实现：

- `setup_daemon`：将进程转换为守护进程
- `write_pid`：写入PID文件
- `read_pid`：读取PID文件
- `check_pid`：检查PID文件是否存在

守护进程的实现代码：

```c
int setup_daemon(void)
{
    switch(fork()) {
    case -1:
        exit(1);
    case 0:
        if(setsid() == -1) {
            exit(1);
        }
        switch(fork()) {
        case -1:
            exit(1);
        case 0:
            chdir("/");
            umask(0);
            return 0;
        default:
            exit(0);
        }
    default:
        exit(0);
    }
}
```

### 2.5 主循环处理

Portal进程的主循环在`Call_Server`函数中实现，主要功能是：

1. 监听控制套接字和Portal套接字
2. 处理定时事件
3. 处理来自控制套接字的消息
4. 处理来自Portal套接字的消息

```c
void Call_Server()
{
    // ...
    for(; ;)
    {
        // ...
        long next_time = next_event_time();
        // ...
        if(ndes == 0)
        {
            Event_Process();        
        }
        else
        {
            if(FD_ISSET(ctl_fd, &readfds))
            {
                // ...
                Control_Handle_portal();
                // ...
            }
            
            if(portal_fd > 0)
            {
                if(FD_ISSET(portal_fd, &readfds))
                {
                    // ...
                    portal_msg_handle();
                    // ...
                }
            }
        }
    }
}
```

## 3. 进程启动流程

Portal进程的启动流程在`main`函数中实现，主要步骤如下：

1. 检查PID文件，确保只有一个实例运行
2. 将进程转换为守护进程
3. 初始化控制套接字和Portal套接字
4. 初始化Portal配置
5. 初始化事件处理机制
6. 进入主循环

```c
int main(int argc, char **argv)  
{
    // ...
    if (!check_pid(PidFile))
    {
        setup_daemon();    
    }
    // ...
    if(0 != init_ctl_socket())
    {
        return 1;
    }
    
    if(0 != init_portal_sock())
    {
        return 1;
    } 
    
    // ...
    free_event_list();
    init_event();            
    
    // ...
    Call_Server();
    // ...
    return 0;
}
```

## 4. 与内核模块的交互

Portal用户态进程通过以下方式与内核模块交互：

1. **控制套接字**：用于接收内核模块发送的控制消息
2. **配置对象**：通过配置对象共享认证策略和配置信息
3. **会话私有数据**：内核模块通过会话私有数据存储认证状态

## 5. 总结

Portal进程是一个网络认证系统的用户态组件，主要负责处理认证请求、管理配置信息、与外部Portal服务器通信等功能。它通过事件处理机制实现定时任务，通过守护进程方式在后台运行，通过套接字与内核模块和外部服务器通信。

该进程是Portal认证系统的核心组件，负责实现认证协议的处理逻辑，为网络接入控制提供认证服务。


# Portal应用程序关键函数分析（含流程图）

通过分析Portal应用程序的代码，我将详细解析三个关键函数的功能和流程，并为每个函数提供流程图。

## 1. portal_msg_handle函数分析

`portal_msg_handle`函数位于`portal_proccess.c`第494行，主要负责处理从Portal服务器接收到的消息。

### 功能概述
该函数处理Portal认证协议中的各种消息类型，包括挑战请求、认证请求、注销请求和信息请求等。

### 处理流程
1. 接收来自Portal套接字的数据包
2. 根据消息类型进行不同处理：
   - **REQ_CHALLENGE**：处理挑战请求
     - 生成随机挑战码
     - 返回ACK_CHALLENGE响应
   - **REQ_AUTH**：处理认证请求
     - 提取用户名、挑战码和密码
     - 调用RADIUS认证接口
     - 返回ACK_AUTH响应
   - **REQ_LOGOUT**：处理注销请求
     - 调用防火墙接口删除用户规则
     - 返回ACK_LOGOUT响应
   - **REQ_INFO**：处理信息请求
     - 更新心跳时间戳
     - 设置Portal状态为在线
     - 重置心跳超时事件
     - 返回ACK_INFO响应

### 流程图

```
┌─────────────────────────────┐
│      portal_msg_handle      │
└──────────────┬──────────────┘
               ▼
┌─────────────────────────────┐
│  接收来自Portal套接字的数据包  │
└──────────────┬──────────────┘
               ▼
┌─────────────────────────────┐
│     解析消息类型(msg->type)   │
└──────────────┬──────────────┘
               ▼
┌──────────────────────────────────────────┐
│                 消息类型?                 │
└───┬─────────────┬──────────┬─────────────┘
    ▼             ▼          ▼             ▼
┌─────────┐  ┌─────────┐ ┌─────────┐  ┌─────────┐
│REQ_CHALL│  │REQ_AUTH │ │REQ_LOG  │  │REQ_INFO │
│  ENGE   │  │         │ │  OUT    │  │         │
└────┬────┘  └────┬────┘ └────┬────┘  └────┬────┘
     ▼             ▼          ▼             ▼
┌─────────┐  ┌─────────┐ ┌─────────┐  ┌─────────┐
│生成随机  │  │提取用户 │ │获取用户 │  │更新心跳 │
│挑战码    │  │认证信息 │ │IP地址   │  │时间戳   │
└────┬────┘  └────┬────┘ └────┬────┘  └────┬────┘
     ▼             ▼          ▼             ▼
┌─────────┐  ┌─────────┐ ┌─────────┐  ┌─────────┐
│设置响应  │  │调用RADIUS│ │调用防火 │  │设置Portal│
│类型为ACK_│  │认证接口  │ │墙接口删 │  │状态为在线│
│CHALLENGE │  │         │ │除用户规则│  │         │
└────┬────┘  └────┬────┘ └────┬────┘  └────┬────┘
     ▼             ▼          ▼             ▼
┌─────────┐  ┌─────────┐ ┌─────────┐  ┌─────────┐
│发送响应  │  │设置响应 │ │设置响应 │  │删除现有 │
│          │  │类型为   │ │类型为   │  │心跳超时 │
│          │  │ACK_AUTH │ │ACK_LOGOUT│ │事件     │
└────┬────┘  └────┬────┘ └────┬────┘  └────┬────┘
     │             ▼          ▼             ▼
     │        ┌─────────┐ ┌─────────┐  ┌─────────┐
     │        │发送响应 │ │发送响应 │  │添加新的 │
     │        │         │ │         │  │心跳超时 │
     │        │         │ │         │  │事件     │
     │        └────┬────┘ └────┬────┘  └────┬────┘
     │             │          │             ▼
     │             │          │        ┌─────────┐
     │             │          │        │发送响应 │
     │             │          │        │         │
     │             │          │        │         │
     │             │          │        └────┬────┘
     ▼             ▼          ▼             ▼
┌─────────────────────────────────────────────────┐
│                     返回                        │
└─────────────────────────────────────────────────┘
```

## 2. Control_Handle_portal函数分析

`Control_Handle_portal`函数位于`portal_proccess.c`第776行，主要负责处理来自控制套接字的消息。

### 功能概述
该函数处理内部控制消息，包括配置更新、状态查询和注销通知等。

### 处理流程
1. 接受来自控制套接字的连接
2. 读取控制消息
3. 根据消息类型进行不同处理：
   - **CFG_PORTAL_SEND_MSG**：更新Portal配置
     - 更新服务器IP、端口等配置
     - 如果设备IP变更，重新生成Nginx配置
     - 如果心跳超时间隔变更，重置事件处理
   - **CFG_PORTAL_QUERY_STATUS**：查询Portal状态
     - 返回当前Portal状态
   - **CFG_PORTAL_NOTIFY_LOGOUT**：通知用户注销
     - 构造注销消息
     - 发送给Portal服务器

### 流程图

```
┌─────────────────────────────┐
│    Control_Handle_portal    │
└──────────────┬──────────────┘
               ▼
┌─────────────────────────────┐
│  接受来自控制套接字的连接    │
└──────────────┬──────────────┘
               ▼
┌─────────────────────────────┐
│       读取控制消息          │
└──────────────┬──────────────┘
               ▼
┌─────────────────────────────┐
│     解析消息类型            │
└──────────────┬──────────────┘
               ▼
┌──────────────────────────────────────────┐
│                 消息类型?                 │
└───┬─────────────┬──────────────┬─────────┘
    ▼             ▼              ▼
┌─────────────┐┌─────────────┐┌─────────────┐
│CFG_PORTAL_  ││CFG_PORTAL_  ││CFG_PORTAL_  │
│SEND_MSG     ││QUERY_STATUS ││NOTIFY_LOGOUT│
└──────┬──────┘└──────┬──────┘└──────┬──────┘
       ▼              ▼              ▼
┌─────────────┐┌─────────────┐┌─────────────┐
│检查服务器IP ││返回当前     ││构造注销消息 │
│是否变更     ││Portal状态   ││             │
└──────┬──────┘└──────┬──────┘└──────┬──────┘
       ▼              │              ▼
┌─────────────┐       │       ┌─────────────┐
│检查端口     │       │       │发送注销消息 │
│是否变更     │       │       │到Portal服务器│
└──────┬──────┘       │       └──────┬──────┘
       ▼              │              │
┌─────────────┐       │              │
│检查设备IP   │       │              │
│是否变更     │       │              │
└──────┬──────┘       │              │
       ▼              │              │
┌─────────────┐       │              │
│如果设备IP变更│       │              │
│重新生成Nginx│       │              │
│配置         │       │              │
└──────┬──────┘       │              │
       ▼              │              │
┌─────────────┐       │              │
│检查心跳超时 │       │              │
│间隔是否变更 │       │              │
└──────┬──────┘       │              │
       ▼              │              │
┌─────────────┐       │              │
│如果心跳超时 │       │              │
│间隔变更，   │       │              │
│重置事件处理 │       │              │
└──────┬──────┘       │              │
       ▼              ▼              ▼
┌─────────────────────────────────────────┐
│              关闭连接                   │
└──────────────────┬──────────────────────┘
                   ▼
┌─────────────────────────────────────────┐
│                 返回                    │
└─────────────────────────────────────────┘
```

## 3. 心跳超时事件处理

第684行的代码片段位于`portal_msg_handle`函数中的REQ_INFO消息处理部分，涉及到心跳超时事件的处理。

### 功能概述
这段代码负责重置Portal心跳超时事件，确保在收到Portal服务器的心跳消息后，更新超时计时器。

### 处理流程
1. 删除现有的心跳超时事件
2. 添加新的心跳超时事件，超时时间为配置的心跳超时间隔
3. 设置心跳超时处理函数为`portal_heartbeat_timeout`

### 流程图

```
┌─────────────────────────────┐
│      心跳超时事件处理       │
└──────────────┬──────────────┘
               ▼
┌─────────────────────────────┐
│  删除现有的心跳超时事件     │
│  Delete_Event(EVENT_PORTAL_ │
│  HEARTBEAT_TIMEOUT_NAME)    │
└──────────────┬──────────────┘
               ▼
┌─────────────────────────────┐
│  添加新的心跳超时事件       │
│  Add_Event(EVENT_PORTAL_    │
│  HEARTBEAT_TIMEOUT_NAME)    │
└──────────────┬──────────────┘
               ▼
┌─────────────────────────────┐
│  设置心跳超时处理函数       │
│  portal_heartbeat_timeout   │
└──────────────┬──────────────┘
               ▼
┌─────────────────────────────┐
│  更新时间戳                 │
│  time_update = time(NULL)   │
└──────────────┬──────────────┘
               ▼
┌─────────────────────────────┐
│            返回            │
└─────────────────────────────┘
```

### portal_heartbeat_timeout函数
虽然代码片段中没有显示`portal_heartbeat_timeout`函数的实现，但从上下文可以推断，该函数在心跳超时时被调用，主要功能是：
1. 将Portal状态设置为离线
2. 可能会清理相关资源或通知其他组件Portal服务器已离线

## 总结

这三个函数共同构成了Portal应用程序的核心功能：
1. `portal_msg_handle`：处理Portal协议消息，实现认证、注销等功能
2. `Control_Handle_portal`：处理内部控制消息，实现配置更新和状态查询
3. 心跳超时事件处理：维护与Portal服务器的连接状态，确保及时检测到服务器离线

这些函数通过事件驱动的方式工作，在主循环中通过select机制监听套接字和定时器事件，实现了Portal认证系统的完整功能。

# Portal认证系统配置函数分析

通过分析提供的代码，我将对四个关键配置函数进行详细解析，并提供流程图。

## 1. tos_auth_policy_add 函数分析

`tos_auth_policy_add` 函数位于 `auth_policy_cfg.c` 第99行，主要负责添加认证策略。

### 功能概述
该函数用于创建新的Portal认证策略，包括设置认证区域、认证服务器、认证方式等参数。

### 实现逻辑
1. 解析输入参数，提取认证策略的各项配置
2. 检查参数有效性
3. 构造认证策略对象
4. 调用底层接口添加认证策略对象
5. 返回操作结果

### 流程图

```
┌─────────────────────────────┐
│    tos_auth_policy_add      │
└──────────────┬──────────────┘
               ▼
┌─────────────────────────────┐
│     解析输入参数            │
└──────────────┬──────────────┘
               ▼
┌─────────────────────────────┐
│     检查参数有效性          │
└──────────────┬──────────────┘
               ▼
┌─────────────────────────────┐
│   构造认证策略对象          │
│  (struct auth_policy)       │
└──────────────┬──────────────┘
               ▼
┌─────────────────────────────┐
│ 调用add_obj_vs添加认证策略  │
└──────────────┬──────────────┘
               ▼
┌─────────────────────────────┐
│        返回结果             │
└─────────────────────────────┘
```

## 2. tos_server_local_modify 函数分析

`tos_server_local_modify` 函数位于 `portal_server_cfg.c` 第110行，主要负责修改本地Portal服务器配置。

### 功能概述
该函数用于修改本地Portal服务器的配置参数，包括服务器IP、端口、认证页面URL等。

### 实现逻辑
1. 初始化本地Portal配置结构体
2. 解析输入参数，提取需要修改的配置项
3. 根据参数类型设置对应的配置项
4. 调用 `set_local_portal_default` 函数应用配置
5. 返回操作结果

### 流程图

```
┌─────────────────────────────┐
│   tos_server_local_modify   │
└──────────────┬──────────────┘
               ▼
┌─────────────────────────────┐
│ 初始化local_portal结构体    │
└──────────────┬──────────────┘
               ▼
┌─────────────────────────────┐
│     遍历输入参数            │
└──────────────┬──────────────┘
               ▼
┌─────────────────────────────┐
│   根据参数类型设置配置项    │
│ - 服务器IP                  │
│ - 端口                      │
│ - 认证页面URL               │
│ - 其他参数                  │
└──────────────┬──────────────┘
               ▼
┌─────────────────────────────┐
│ 调用set_local_portal_default│
│ 应用配置                    │
└──────────────┬──────────────┘
               ▼
┌─────────────────────────────┐
│        返回结果             │
└─────────────────────────────┘
```

## 3. tos_server_outside_template_modify 函数分析

`tos_server_outside_template_modify` 函数位于 `portal_server_cfg.c` 第928行，主要负责修改外部Portal服务器模板配置。

### 功能概述
该函数用于修改外部Portal服务器的模板配置，包括重定向URL、推送开关、用户IP名称等参数。

### 实现逻辑
1. 初始化外部Portal模板结构体
2. 解析输入参数，提取需要修改的配置项
3. 根据参数类型设置对应的配置项
4. 调用 `set_outside_portal` 函数应用配置
5. 返回操作结果

### 流程图

```
┌───────────────────────────────────┐
│ tos_server_outside_template_modify│
└────────────────┬──────────────────┘
                 ▼
┌───────────────────────────────────┐
│ 初始化outside_portal_template结构体│
└────────────────┬──────────────────┘
                 ▼
┌───────────────────────────────────┐
│       遍历输入参数                │
└────────────────┬──────────────────┘
                 ▼
┌───────────────────────────────────┐
│     根据参数类型设置配置项        │
│ - 服务器名称                      │
│ - 重定向URL                       │
│ - 推送开关                        │
│ - 用户IP名称                      │
│ - 设备IP名称                      │
│ - 用户访问地址名称                │
└────────────────┬──────────────────┘
                 ▼
┌───────────────────────────────────┐
│   调用set_outside_portal函数      │
│   应用配置(CFG_OUTSIDE_PORTAL_MODIFY)│
└────────────────┬──────────────────┘
                 ▼
┌───────────────────────────────────┐
│          返回结果                 │
└───────────────────────────────────┘
```

## 4. tos_server_outside_protocol_modify 函数分析

`tos_server_outside_protocol_modify` 函数位于 `portal_server_cfg.c` 第1369行，主要负责修改外部Portal服务器协议配置。

### 功能概述
该函数用于修改外部Portal服务器的协议配置，包括服务器IP、端口、心跳间隔、设备IP和开关状态等参数。

### 实现逻辑
1. 初始化外部Portal协议结构体
2. 获取当前Portal协议配置数量
3. 解析输入参数，提取需要修改的配置项
4. 根据参数类型设置对应的配置项
5. 调用 `set_outside_portal_protocol_default` 函数应用配置
6. 返回操作结果

### 流程图

```
┌───────────────────────────────────┐
│ tos_server_outside_protocol_modify│
└────────────────┬──────────────────┘
                 ▼
┌───────────────────────────────────┐
│初始化outside_portal_protocol结构体│
└────────────────┬──────────────────┘
                 ▼
┌───────────────────────────────────┐
│  获取当前Portal协议配置数量       │
└────────────────┬──────────────────┘
                 ▼
┌───────────────────────────────────┐
│       遍历输入参数                │
└────────────────┬──────────────────┘
                 ▼
┌───────────────────────────────────┐
│     根据参数类型设置配置项        │
│ - 服务器IP                        │
│ - 端口                            │
│ - 心跳间隔                        │
│ - 设备IP                          │
│ - 开关状态                        │
└────────────────┬──────────────────┘
                 ▼
┌───────────────────────────────────┐
│检查设备IP有效性(portal_check_ip)  │
└────────────────┬──────────────────┘
                 ▼
┌───────────────────────────────────┐
│设置开关状态(portal_switch_set)    │
└────────────────┬──────────────────┘
                 ▼
┌───────────────────────────────────┐
│调用set_outside_portal_protocol_default│
│应用配置                           │
└────────────────┬──────────────────┘
                 ▼
┌───────────────────────────────────┐
│          返回结果                 │
└───────────────────────────────────┘
```

## 总结

这四个函数共同构成了Portal认证系统的配置管理功能：

1. `tos_auth_policy_add`: 负责添加认证策略，定义用户认证的规则和方式
2. `tos_server_local_modify`: 负责修改本地Portal服务器配置，管理内置Portal服务
3. `tos_server_outside_template_modify`: 负责修改外部Portal服务器模板，定义重定向和参数传递方式
4. `tos_server_outside_protocol_modify`: 负责修改外部Portal服务器协议参数，管理与外部Portal服务器的通信

这些函数通过统一的参数解析和配置应用机制，实现了Portal认证系统的灵活配置。配置变更通过底层对象管理接口持久化，并通过控制接口通知Portal进程应用新的配置。