> https://blog.csdn.net/weixin_44827364/article/details/104078347   ROS分布式多机通讯的配置与实现
> https://blog.csdn.net/hehedadaq/article/details/82898307    ros分布式多机通信完整教程
>
> 天之博特：http://doc.tianbot.com/ros2go/2063903#ROS_8



# 一、山速多机通信配置笔记：

## 1. 保证各模块通电正常

## 2. 主机Network设置：

![image-20210629212814948](./photo/network_settings.png)

```
Settings --> Wired --> options --> IPv4 Settings ##(Method: Manual, Adress & NetMask)
```

`eg： 192.168.1.30 ,    255.255.255.0`

## 3. Ros `Start.sh` setting:

> path is `/opt/ros/scripts/start.sh`

add :  (主机IP)

```bash
export ROS_MASTER_URI=http://192.168.1.30:11311
export ROS_IP=192.168.1.30
```

start.sh as follows:

```
#!/bin/bash


if [ -f /opt/ros/kinetic/setup.bash ];then
     source /opt/ros/kinetic/setup.bash
fi

if [ -f /home/shansu/catkin_ws/devel/setup.bash ];then
     source /home/shansu/catkin_ws/devel/setup.bash
fi
export ROS_MASTER_URI=http://192.168.1.30:11311
export ROS_IP=192.168.1.30
echo "ROS environment is Ready"
sleep 3
roslaunch bulldog_driver bulldog_base.launch

```

## 4. 主、从机`hosts`配置

> path is `/etc/hosts`

master： add slave address  

`eg: 192.168.1.134	nvidia-desktop`   #(中间用tab键隔开)

slave: add master address

`eg: 192.168.1.30	bulldog`

## 5. 配置自启动文件

> path is `etc/rc.local`

add ROS `start.sh` to `rc.local`

eg: 

```bash
# ......
/opt/ros/scripts/start.sh
exit 0
```

## 6. 配置从机环境变量`.bashrc`

```
export ROS_MASTER_URI=http://192.168.1.30:11311
export ROS_IP=192.168.1.134
```

```bash
source ~/.bashrc
```







## others:

1. kill master

   ```bash
   sudo killal -9 rosmaster
   ```

   

2. 手柄调教工具

   ```bash
   sudo apt install jstest-gtk
   
   jstest-gtk
   ```

   

# 二、天之博特教程：

# ROS多机通信

在充分的单机学习测试之后，往往要进行真实机器人的测试，那么就要接触到多机通信的问题。ROS采用的分布式网络通信，只要正确的设置ROS_MASTER_URI和ROS_IP两个环境变量，就可以成功实现ROS的多机通信，有时我们也称之为主从机设置。但是在这里初学者往往因为没有深入理解ROS通信机制而产生问题。ROS Wiki的说明也比较笼统，但是需要先行阅读。后面我们详细说明在多机通信产生问题时如何处理。

http://wiki.ros.org/ROS/NetworkSetup
http://wiki.ros.org/ROS/Tutorials/MultipleMachines

## ROS消息通信机制

![img](https://img.kancloud.cn/be/68/be685b55e3b4d43915f9de9a805ed62c_746x366.png)

我们简单梳理一下，在消息通信时，什么时候会用到ROS_MASTER_URI和ROS_IP这两个环境变量。

- Talker启动，通过ROS_MASTER_URI连接到ROS Master，注册发布者的信息，包含所要发布的话题名称（bar）和消息类型；ROS Master会将该发布者的注册信息存储到发布者注册列表中，等待接收者。
- Listener启动，通过ROS_MASTER_URI连接到ROS Master，将其订阅的话题名bar和消息类型在ROS Master上注册（实际上也包括自己的地址和端口信息）。
- ROS  Master根据Listener所需的订阅话题在发布者注册列表寻找与之匹配的话题，并返回Talker的XML-RPC地址信息，即foo:1234。Listener根据alker的XML-RPC地址向Talker请求TCP的连接地址，Talker的地址就是图中所示的foo:2345。其中foo就是在执行talker时的环境变量ROS_IP的。

**因此，如果需要正确接收到talker发布的消息，必须在talker和listener上都正确设置ROS_MASTER_URI，必须正确设置Talker的ROS_IP。一般情况下，都是双向互通，所以listener的ROS_IP也要正确设置。**

## 最常见问题：可以看到消息列表，但是无法接收消息内容

https://answers.ros.org/question/349095/ros-on-multiple-machine-not-working-properly/#349098
 这是最困扰大家的一个问题。我们用robot和ros2go来代表机器人上的电脑和远程操控的电脑。那么首先大家能够在robot和ros2go上能够相互ping通（相互，保证是在同网段），也可以使用ssh成功从ros2go端登录到robot。多机通信大家一般都会知道在ros2go端将ROS_MASTER_URI设置为robot的ip，这样就可以成功的使用`rostopic list`看到话题列表。不过`rostopic echo /foobar`是看不到任何消息的（很大概率，也有例外，后面我们说明）。这个最主要的问题就是没有在robot上设置ROS_IP这个环境变量。

类似的，如果ros2go端的ROS_IP没有设置正确，可能发生在RViz中能够看到传感器的数据，但是发布2D Nav  Goal而机器人完全没反应的状况。总结一下，如果能看到话题列表而没有话题内容，就是因为通讯是没问题的可以在ROS Master  上注册，但是上报地址错了导致后面无法进行点对点通信。

### 安全可靠：全部用IP地址进行设置

ROS_MASTER_URI和ROS_IP都可以在.bashrc文件中设置。我们假设robot的IP地址为192.168.1.100，ros2go的ip地址为192.168.1.111，roscore在robot上启动（一般我们都是在robot上启动roscore）。那么
 在robot上

```
export ROS_MASTER_URI=http://192.168.1.100:11311
export ROS_IP=192.168.1.100
```

在ros2go上

```
export ROS_MASTER_URI=http://192.168.1.100:11311
export ROS_IP=192.168.1.111
```

这样的设置是一定可以互通的。因为本地回环的存在，在robot上可以设置为

```
export ROS_MASTER_URI=http://localhost:11311
export ROS_IP=192.168.1.100
```

### 方便：用正则截取相应设备IP

用IP进行设置确实不会出问题，但是只要更换网络或者设备的IP发生变化等，多机通信就失效了需要重新设置。在Linux系统中我们有多种方式可以获得当前的IP地址。以下都是设置ROS_IP的方式。

最简单的情况下，仅有ipv4地址时，可以如下设置

```
export ROS_IP=`hostname -I`
```

但是有ipv6的地址时，需要截取

```
export ROS_IP=`hostname -I | cut -d " " -f 1`
```

不过有多个ipv4地址的情况下，比如使用NVidia Jetson系列板卡，会虚拟出一个usb网络，那么不能够保证截取到正确的ipv4地址。
 非常确定的要截取某个网络下的ipv4地址时，可以使用如下命令

```
IP=$(ip addr show wlan0 | grep -w inet | awk '{print $2}' | awk -F / '{print $1}')
export ROS\_IP=${IP}
```

这样可以截取wlan0网络下的ipv4地址，已经非常灵活通用。但是在网络未准备好的情况下，截取的内容是空，这时候启动roscore或者其他节点时会报如下错误：

```
invalid ROS_IP (an empty string)
```

这样虽然在启动roscore时会用hostname信息替代，但是节点启动时上报的ROS_IP信息为空，会导致节点间无法通信。需要将ROS_IP的设置注释掉，或者保证ROS_IP能拿到正确的地址。

### 官方：设置hostname

当我们不设置ROS_IP的时候，实际上节点上传到ROS Master的地址是本机的hostname。使用`rosnode info`查看可以看到节点的地址类似于 http://ros2go:33333/

为什么我们没有设置ROS_IP时，节点之间还是可以成功通信呢？原因是系统启动时会上报自己的hostname给路由器，路由器的dns可以在局域网内成功解析hostname，对应到正确的IP地址从而使得ROS实现多机通信。但是因为多种因素，路由器dns解析的成功率非常不靠谱。

最简单的hostname设置：路由器可以正确解析hostname，那么可以按照官方教程，仅设置/etc/hostname文件（实际上这个文件一般也都设置好了），写入hostname即可。如果可以起作用，那么我们不用设置ROS_IP，就可以实现多机通信。这种情况最大的问题就是不可靠，也是产生问题的根源。
 强制解析：设置/etc/hosts文件，在文件中加入强制hostname和IP地址的对应关系。这样设置的问题和纯IP设置类似，在网络条件发生变化的时候还是要修改。好处是hostname可以有意义，比IP地址好记的多。

使用ROS2GO和Tianbot官方的设备时，我们已经将比较优化的设置写入了.bashrc，大家可以根据需要去掉注释。