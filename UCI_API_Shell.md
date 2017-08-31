# UCI API（Shell版）
[TOC]

## API的内部实现
```bash
. /lib/functions.sh    
. /lib/config/uci.sh    
```
这两行是UCI shell版API的内部实现的源代码所在文件。 
接下来就让我们来先读一读其封装实现。
### 集大成者config_load
在使用任何其他API之前都需要先调用config_load [config file name]操作，这一步也是整个API封装实现的核心。    
```bash
config_load() {                               ... 
        uci_load "$@"
}

CONFIG_APPEND=
uci_load() {                                                             
        local PACKAGE="$1"                                               
        local DATA                                                       
        local RET                                                        
        local VAR                                                        
                                                                         
        _C=0
        # 清空全局变量
        if [ -z "$CONFIG_APPEND" ]; then                                 
                for VAR in $CONFIG_LIST_STATE; do                        
                        export ${NO_EXPORT:+-n} CONFIG_${VAR}=           
                        export ${NO_EXPORT:+-n} CONFIG_${VAR}_LENGTH=    
                done                                                     
                export ${NO_EXPORT:+-n} CONFIG_LIST_STATE=           
                export ${NO_EXPORT:+-n} CONFIG_SECTIONS=             
                export ${NO_EXPORT:+-n} CONFIG_NUM_SECTIONS=0        
                export ${NO_EXPORT:+-n} CONFIG_SECTION=              
        fi                                                           
        # 通过/sbin/uci程序把对应的配置文件导出到变量$DATA中                                           
        DATA="$(/sbin/uci ${UCI_CONFIG_DIR:+-c $UCI_CONFIG_DIR} ${LOAD_STATE:+-P /var/state} -S -n export "$PACKAGE" 2>/dev/null)"
        RET="$?"                                                              # 这一句很关键eval "$DATA"，这里需要理解$DATA中的内容是什么，同时需要知道eval这个命令的作用                                                    
        [ "$RET" != 0 -o -z "$DATA" ] || eval "$DATA"       
        unset DATA               
        ${CONFIG_SECTION:+config_cb}                         
        return "$RET"                                       
}
```
```DATA="$(/sbin/uci ${UCI_CONFIG_DIR:+-c $UCI_CONFIG_DIR} ${LOAD_STATE:+-P /var/state} -S -n export "$PACKAGE" 2>/dev/null)"```    
这样我们可以简化为```DATA="$(/sbin/uci -P /var/state -S -n export "$PACKAGE" 2>/dev/null)"```
如果这里的package是wireless，那么其内容可能为:
```bash
root@WIA3300-20:/lib# /sbin/uci -P /var/state -S -n export wireless
package wireless
config wifi-device 'wifi0'
        option type 'qcawifi'
        option channel 'auto'
        option macaddr 'e4:3e:d7:7c:76:70'
        option disabled '0'
        option txpower '26'
        option regdomain 'CN'
        option hwmode '11ng'
        option htmode 'HT20'
        option country '156'
```
而```eval $DATA```展开来其实就是:
```bash
eval package wireless
eval config wifi-device 'wifi0'
eval option type 'qcawifi'
eval option channel 'auto'
...
```
其实质就是调用已经定义好的shell函数```package```、```config```等    
详细的函数在```/lib/functions.sh```中，其内容如下:
```bash
package() {   
        return 0      
} 
config () {    
        local cfgtype="$1" 
        local name="$2"                
        export ${NO_EXPORT:+-n} CONFIG_NUM_SECTIONS=$(($CONFIG_NUM_SECTIONS + 1))
        
        # 系统生成的一个对于当前package唯一的name，在config_get中使用
        name="${name:-cfg$CONFIG_NUM_SECTIONS}"  
        
        # CONFIG_SECTIONS这个全局变量很重要，在后面的config_foreach中会用到
        append CONFIG_SECTIONS "$name"  
        [ -n "$NO_CALLBACK" ] || config_cb "$cfgtype" "$name" 
        export ${NO_EXPORT:+-n} CONFIG_SECTION="$name"   
        export ${NO_EXPORT:+-n} "CONFIG_${CONFIG_SECTION}_TYPE=$cfgtype" 
}                                                                                           
option () {      
        local varname="$1"; shift 
        local value="$*"   
        
        # 这里定义用来表示一条option记录的全局唯一变量，并把option的值存储在其中
        export ${NO_EXPORT:+-n} "CONFIG_${CONFIG_SECTION}_${varname}=$value"
        [ -n "$NO_CALLBACK" ] || option_cb "$varname" "$*" 
}                                                                                            
list() {       
        local varname="$1"; shift 
        local value="$*" 
        local len                
        config_get len "$CONFIG_SECTION" "${varname}_LENGTH" 0 
        [ $len = 0 ] && append CONFIG_LIST_STATE "${CONFIG_SECTION}_${varname}"             
        len=$(($len + 1)) 
        
        # 存储每条list的值
        config_set "$CONFIG_SECTION" "${varname}_ITEM$len" "$value" 
        config_set "$CONFIG_SECTION" "${varname}_LENGTH" "$len" 
        append "CONFIG_${CONFIG_SECTION}_${varname}" "$value" "$LIST_SEP" 
        list_cb "$varname" "$*"  
}  
```
看到这里，其实会发现，UCI的Shell版API是利用一系列环境变量来存储配置，并通过Shell函数再封装成易于使用的API。

## API config_xxx
### config_load
加载配置文件，需要在其他config_xxx指令之前先执行    
指令格式：
```config_load [name]```    

Example:     
```config_load wireless```

### config_foreach
轮询所有指定section type的section，并调用handle function进行对section进行处理    
指令格式：
```config_foreach [hanle function] [section type]```    

Example:    
```bash
handle_wifi_device(){
    local section="$1"
    local custom="$2"
    # do some you want
}
config_foreach handle_wifi_device wifi-device
```

### config_get
从指定的section.option获取值，并存储在var变量中    
指令格式：
```config_get [var] [section] [option] [default value]```    

Example:    
```bash
config_get var_channel $section channel 6
echo $var_channel
```
### config_get_bool
和config_get类似，只是取出的值被强制转化为bool类型，即0和1

### config_set
把value赋值给section.option    
指令格式：
```config_set [section] [option] [value]```    

Example:    
```bash
config_set $section channel 6
```

### config_list_foreach
轮询指定section下面的list，并调用指定的handle function进行处理    
指令格式：
```config_foreach [section type] [list option name] [hanle function] [others arg]```    

Example:    
```bash
# handle list items in a callback
handle_ac_addr(){
    local value="$1"
    # do some you want
}
config_foreach $section ac_addr handle_ac_addr
```

## API uci_xxx
这类API其实是对/sbin/uci命令的直接封装；    
源文件在/lib/config/uci.sh中；
### uci_set
### uci_get
### uci_add
### uci_remove
### uci_rename
### uci_commit

这里只简单举一个示例说明以下，这些和对应的/sbin/uci指令一一对应    
```bash
uci_set() {                                      
        local PACKAGE="$1"                               
        local CONFIG="$2"                                
        local OPTION="$3"                           
        local VALUE="$4"
        
        # 直接对/sbin/uci进行的封装
        /sbin/uci ${UCI_CONFIG_DIR:+-c $UCI_CONFIG_DIR} set "$PACKAGE.$CONFIG.$OPTION=$VALUE"     
}  
```

## API uci_xxx_state
这类API其实也是对/sbin/uci命令的直接封装；    
源文件在/lib/config/uci.sh中；    
它和uci_xxx的区别在于，这部分API是操作/var/state目录下面的配置，而该部分配置在执行uci commit2动作时，不会被存储到flash上；且执行uci show 也是无法看见的，即使路径一致。

### uci_get_state
### uci_set_state
### uci_revert_state
### uci_toggle_state
### uci_set_default

这里只简单举一个示例说明以下，这些和对应的/sbin/uci指令一一对应    
```bash
uci_set_state() {                                
        local PACKAGE="$1"                         
        local CONFIG="$2"                          
        local OPTION="$3"                    
        local VALUE="$4"        
        [ "$#" = 4 ] || return 0                        
        /sbin/uci ${UCI_CONFIG_DIR:+-c $UCI_CONFIG_DIR} -P /var/state set "$PACKAGE.$CONFIG${OPTION:+.$OPTION}=$VALUE"             
}  
```
比如：
```bash
ipaddr=uci_get network lan ipaddr
echo $ipaddr ## 输出'192.168.18.1'

uci_set_state network lan ipaddr 192.168.80.1
ipaddr=uci_get_state network lan ipaddr
echo $ipaddr ## 输出'192.168.80.1'

# 再次调用
ipaddr=uci_get network lan ipaddr
echo $ipaddr ## 输出还是为'192.168.18.1'
```
**因此这两种调用方式存储的值是独立的。**

## 秘密武器
### config_cb
### option_cb
### list_cb    

以上几个回调函数如果用户有定义，会在执行config_load时自动执行。    
比如：config_cb    
在config()函数的定义中会直接执行：
```bash
config(){
    ...
    [ -n "$NO_CALLBACK" ] || config_cb "$cfgtype" "$name" 
    ...
}
```
从这里也可以看出config_cb的入参```"$cfgtype" "$name"```    
另外，在所有的shell中，是可以直接调用config_load时定义和使用的很多全局变量的。    
比如：
```bash
CONFIG_SECTIONS
CONFIG_${CONFIG_SECTION}_TYPE
CONFIG_${CONFIG_SECTION}_${varname}
...
```
## 其他函数
### append
```bash
append [name] [value0] [value1]
```
Example:
```bash
vif0=ath0
vif1=ath1
append vifs $vif0
append vifs $vif1
echo $vifs # 输出为: ath0 ath1
```

## 你不知道的秘密
### config_load的坑
从之前config_load的实现上分析:
```bash
uci_load() {
...
        if [ -z "$CONFIG_APPEND" ]; then
                for VAR in $CONFIG_LIST_STATE; do
                        export ${NO_EXPORT:+-n} CONFIG_${VAR}=
                        export ${NO_EXPORT:+-n} CONFIG_${VAR}_LENGTH=
                done
                export ${NO_EXPORT:+-n} CONFIG_LIST_STATE=
                export ${NO_EXPORT:+-n} CONFIG_SECTIONS=
                export ${NO_EXPORT:+-n} CONFIG_NUM_SECTIONS=0
                export ${NO_EXPORT:+-n} CONFIG_SECTION=
        fi
...
}
```
* 这些全局变量会在第二次执行config_load [config name]时清零，也就是对应的对原有配置文件再执行config_foreach等操作时，实际上操作的时新load的配置文件，无法再操作上一次config_load的配置文件；    
* 如果需要操作上一次操作的配置文件，必须再一次config_load对应的配置；
 
比如：
```bash
config_load network
config_foreach handle_interface interface

config_load switch
config_foreach handle_switch_vlan switch_vlan

 # 如果不再次执行此动作，后面的config_foreach将会什么都不执行，
 # 因为interface的列表存储在CONFIG_SECTIONS变量之中，
 # 当config_load switch时，这个变量的值已经被清空，变成了switch_vlan的列表
config_load network
config_foreach handle_interface interface

```

## 参考链接
* [config-scripting](https://wiki.openwrt.org/doc/devel/config-scripting)
