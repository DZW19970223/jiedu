# Thermostat_2019A代码功能解读 #
Thermostat_2019A是实现空调智能化的一个终端节点，主要是接收空调遥控器传过来数据，然后根据数据实现相应功能。

----
- [Thermostat_2019A代码功能解读](#thermostat2019a%e4%bb%a3%e7%a0%81%e5%8a%9f%e8%83%bd%e8%a7%a3%e8%af%bb)
- [1. 程序执行流程](#1-%e7%a8%8b%e5%ba%8f%e6%89%a7%e8%a1%8c%e6%b5%81%e7%a8%8b)
- [2. 事件](#2-%e4%ba%8b%e4%bb%b6)
- [3. 具体功能](#3-%e5%85%b7%e4%bd%93%e5%8a%9f%e8%83%bd)
  - [3.1 风力调节](#31-%e9%a3%8e%e5%8a%9b%e8%b0%83%e8%8a%82)
  - [3.2 系统工作模式选择](#32-%e7%b3%bb%e7%bb%9f%e5%b7%a5%e4%bd%9c%e6%a8%a1%e5%bc%8f%e9%80%89%e6%8b%a9)
  - [3.3 显示板数据查询](#33-%e6%98%be%e7%a4%ba%e6%9d%bf%e6%95%b0%e6%8d%ae%e6%9f%a5%e8%af%a2)
  - [3.4 电能计算](#34-%e7%94%b5%e8%83%bd%e8%ae%a1%e7%ae%97)

----
# 1. 程序执行流程
应用程序入口是`simple-main.c`文件下的`int MAIN(MAIN_FUNCTION_PARAMETERS)`函数，该函数下只有三句代码，主要是初始化硬件和串口，执行最后的` emberAfMain(MAIN_FUNCTION_ARGUMENTS)`进入主程序。程序来到了`Thermostat_2019A_callbacks.c`文件下的`emberAfMainInitCallback()`函数，在该函数下：
- `appEventInit()`方法是风机、电表和阀门的初始化。
- `emberAfFanControlClusterServerAttributeChangedCallback(THERMOSTAT_EP, ZCL_FAN_CONTROL_FAN_MODE_ATTRIBUTE_ID)`和`emberAfThermostatClusterServerAttributeChangedCallback(THERMOSTAT_EP, ZCL_SYSTEM_MODE_ATTRIBUTE_ID)`是在接收到指令时会执行的回调函数。
- `emberEventControlSetDelayMS(displayBoardEventControl, 3000)`和`emberEventControlSetDelayMS(electricMeterEventControl, 3000)`是延时3秒后执行事件`displayBoardEventControl`和`electricMeterEventControl`。
# 2. 事件
应用程序一共自定义了6个自定义事件，每个事件都会对应有一个事件处理函数，它们分别是：
- `autoModeEventFunction()`自动模式事件的处理函数，指定空调的工作模式。
- `coolingModeEventFunction`制冷模式事件的处理函数，指定空调的工作模式。
- `heatingModeEventFunction()`制热模式事件的处理函数，但是程序没有做处理。
- `displayBoardEventFunction()`查询显示板数据事件的处理函数，主要查询的是显示板上的数据，温湿度等。
- `electricMeterEventFunction()`电表轮询事件的处理函数，用于查询周期频率、电压、电流、功率和电能的数据。
- `networkEventFunction()`网络事件的处理函数，用于加入和离开网络。

事件需要用`emberEventControlSetActive(control)`函数来激活，也可以用`emberEventControlSetDelayMS(control, delay)`函数来延时激活，激活后可以用`emberEventControlSetInactive(control)`函数来停止事件。如果激活事件后没有停止事件，也没有设置下次执行的时间，事件会以最快的周期执行。
# 3. 具体功能
应用程序的主要功能是风力调节和系统工作模式选择，其余还有显示板数据查询和电能计算等功能，具体如下。
## 3.1 风力调节
风力调节功能是通过改变簇`Fan Control`相关属性的值，调用回调函数，然后再在回调函数中通过`emberAfReadServerAttribute()`函数读取属性的值，从而获得调节风力的参数值，最后调用`appFanSet()`函数来调节风力的等级，风力有三个等级，控制如下：
- 低档风力只打开一个阀。
- 中档风力打开两个阀。
- 高档风力打开三个阀（风机一共三个阀）。

风力调节其实也是一种空调工作模式的选择。
## 3.2 系统工作模式选择
系统工作模式选择功能是通过改变簇`Thermostat`相关属性的值，调用回调函数，然后再在回调函数中通过`emberAfReadServerAttribute()`函数读取属性的值，从而获得选择模式的参数值，最后调用`_setSystemMode()`函数来选择空调的工作模式（调用该方法后要先停掉所有工作模式的事件，再进行判断模式选择），主要模式有关机模式、自动模式、制冷模式、制热模式和仅风机模式,各模式工作方式如下：
- 关机模式，关闭所有开关，什么都不工作。
- 自动模式，读取相关的簇属性值，获得室内的温度，判断温度是否少于或等于26，如果是则保留当前工作状态，30s后再检查一次，如果大于26，则启动制冷模式，30s后检查一次。
- 制冷模式，读相关簇属性拿到室内温度和设定的温度，然后通过一系列处理运算得出参数值，然后把参数值写入相关簇属性中（制冷装置怎么工作的我不太懂），启动制冷调温到设定的温度。
- 制热模式，不需要，没做处理。
- 仅风机模式，只关闭制冷阀门，不关闭风机的阀门，只剩下风机在工作。
## 3.3 显示板数据查询
显示板数据查询的方法在事件`displayBoardEventControl`中调用，也就是`appSpiReceive()`方法，主要是通过串行外设接口接收到的数据进行选择（我猜应该是接收空调遥控器发送过来的数据），数据的第0位就是选择的条件，可以选择的选项有温控模式、室内温度、室内湿度和申请入网离网等，数据解释如下：
- 在温控模式选项下，数据第2位表示选择空调的工作模式是制冷还是制热，并把数据写入相关簇属性中；数据第3位表示制冷或者制热模式下设定的温度，并把数据写入相关簇属性中；数据第4位是设置风机的工作模式，并把数据写入相关簇属性中。
- 在室内温度的选项下，数据第2位和第3位是室内温度的数据，并把数据写入相关簇属性中。
- 在室内湿度的选项下，数据第2位和第3位是室内湿度的数据，并把数据写入相关簇属性中。
- 在申请入网离网选项下，数据第2位是判断入网还是离网，当是入网时，激活事件`networkEventControl`进行入网。
## 3.4 电能计算
电能计算的方法在事件`electricMeterEventControl`中调用，也就是`electricityMeterReadtest()`方法，该方法下可以读出电表的频率、电压、电流和功率的数据，并且把这些属性写入相关簇属性中，而且还能统计电能，读出电表中的电能数据，然后写入相关簇属性中，等到下次激活事件时，再把这个数据从属性中读出来，然后加上从电表上读到的新数据，再把数据写入相关簇属性中，一直这样循环，从而达到统计电能的作用。