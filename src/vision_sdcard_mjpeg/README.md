# vision_sdcard_mjpeg
这里是 ESP32 Vision 的测试代码之一。  
由于 SPIFFS 的容量只能够存十秒上下的 GIF，很多花活可能整不出来；这个 Demo 使用了板上容量可高达 8GB 的 SD Nand 来支持整活。   

## 测试效果 
![image](https://user-images.githubusercontent.com/8705034/155977538-5bb3ef90-7baa-4e10-b06e-492b431bec85.png)  

见 [Bilibili BV1Fr4y1z7yS: 下北泽元素力测试 - bcccc23333](https://www.bilibili.com/video/BV1Fr4y1z7yS)  
（和那个表面看上去没什么区别所以就复制了）

## 硬件
目前版本的代码可配合 硬件版本 V3.3 使用。  

这个例子没有使用到板上那个 0402 封装的 NTC；其他的电路，包括外接的环境光传感器以及 NTC 电阻都需要焊接才能正常工作。  
这个例子通过修改代码的方式同时支持 方版（ST7789）和 圆版（GC9A01），修改方式见下文。  

## 功能
 - 冷启动后，屏幕变灰，尝试连接名为 Celestia，密码为 mimitomo （默认，可在源码中更改）的 Wi-Fi  
 - 如果超时未连接成功则进入正常工作周期；短按按键超过 250ms 可以跳过这个超时过程  
 - 如果连接成功，屏幕变蓝，同时开启一个 WebServer，通过浏览器 http 访问该设备，可以上传 mjpeg 视频到 SD 卡中，可以修改配置文件
 - 上传过程中屏幕会变成橘色，LED 闪烁，上传结束后变回蓝色
 - 短按按键退出上传模式；退出时根据 Y 轴方向判断循环模式，紫色=单次，粉色=循环，绿色=别管他 
 - 播放视频时可以按下按键暂停，暂停时处于浅度睡眠模式；再次按下会恢复播放
 - 循环模式：可以选择循环播放（常亮），或是播放视频一次后进入深度睡眠
 - 深度睡眠时，通过双击唤醒或者定时器唤醒（按键由于不是 RTC IO，无法用于深睡唤醒）
 - 如果近期使用过双击唤醒或是按下过按键，则接下来的几次定时器唤醒会较快，反之亦然
 - 如果电量低，它就也不亮了 （？又废话）
 - 支持动态亮度调节
 - 支持通过配置文件选择播放的 mjpeg，支持在网页中预览 mjpeg 文件（…的第一帧）
 - 支持通过配置文件设置屏幕方向
 - 支持在网页中 OTA 升级固件
 - 播放掉帧，性能堪忧（
 - 任何时候长按超过三秒都会被断电或是上电

## 编译 & 上传 & 初始化设置 

1. 安装 Arduino  
2. 安装 [espressif/arduino-esp32](https://github.com/espressif/arduino-esp32)  （仅在 V1.0.6 下测试过；高版本和下面某个库的兼容不好）
3. 安装 [moononournation/Arduino_GFX](https://github.com/moononournation/Arduino_GFX)  
4. 安装 [bitbank2/JPEGDEC](https://github.com/bitbank2/JPEGDEC)  
5. 安装 [DFRobot/DFRobot_LIS](https://github.com/DFRobot/DFRobot_LIS)  
6. 安装 [stevemarple/IniFile](https://github.com/stevemarple/IniFile)  
7. 下载这坨源码，打开
8. 选择：ESP32 Dev Module, 240MHz CPU, 80MHz Flash, DIO, 4MB(32Mb), Default (1.2M/1.5M), PSRAM Disable  
9. 上传
10. 建立名为 Celestia， 密码为 mimitomo 的 2.4GHz Wi-Fi，该网络中需要提供 DHCP 服务；（就是用电脑或者手机开热点的意思  
11. 给板子上电  
12. 当看到板子连入该 Wi-Fi 后，在 DHCP 地址池中找到该设备，记录下其 IP 地址
13. 使用浏览器 http 打开其 IP 地址，下文以 192.168.1.200 为例；如果你的网络中支持 MDNS 服务也可以打开 http://vision.local    
14. 使用 curl 或是其他什么你熟悉的东西上传 index.htm, edit.htm, ota.htm 三个网页文件：curl -X POST -F "file=@index.htm" http://192.168.1.200/edit
15. 将你需要播放的东西通过 ffmpeg 或是其他的什么方式转为 240x240，大约 24fps 的 mjpeg 格式文件 （参考 [RGB565_video](https://github.com/moononournation/RGB565_video) 中的示例）
16. 使用浏览器打开 http://192.168.1.200 或 http://vision.local ，选择 “文件”，并使用上方的按钮上传 mjpeg 文件到 SD Nand 根目录
17. 在左侧列表中找到 config.txt 并编辑，设置 video 为需要播放的文件名，以 / 开头，例如 video=/loop.mjpeg， Ctrl+S 保存；该文件的语法参考 [stevemarple/IniFile](https://github.com/stevemarple/IniFile)   
18. 将设备处于屏幕水平的状态，盖住光传感器，并双击，看到屏幕一绿即为配置完成
19. 你就又获得了一颗下北泽神之眼 
20. 如果想要再次更改其中内容，只需要将其断电（板上开关）等个十几秒后再接通让它冷启动，即可重复上述相关步骤。

## 自定义配置 

配置文件 /sdcard/config.txt 中的可自定义内容：   

```
 [vision]
 video=/loop.mjpeg       # 要播放的文件名。需要以 / 开头，并且文件名不要太长不然会出bug
 lcd_rotation=0          # LCD 的方向。取值范围 0~3，对应 0°, 90°, 180°, 270°，根据实际情况修改
 loop_mode=false         # true：循环播放视频模式，不会自动息屏；但依然可以用按键进入浅睡息屏模式，该模式下快速唤醒；浅睡不算很省电
                         # false：单次播放模式，播放一次后进入深睡，深睡模式下支持双击唤醒；从唤醒到播放需要几秒，但深睡省电
                         # 循环模式优先级：配置文件>冷启按键时方向>默认值
```

如果你想恢复默认值，只需要删除 /sdcard/config.txt 并重启即可，会生成一个新的默认配置的文件。  

代码中有几个可以配置的地方（其中一部分可能在日后（如果有，咕）的版本中移入配置文件）：  
```

// 系统配置
#define CONFIG_FILENAME "/config.txt"       // SD Nand 根目录中的配置文件的名称。该文件如果不存在会自动生成。 
#define CONF_GIFNAME_DEFAULT "/loop.mjpeg"  // 配置文件中的默认循环播放的文件名。另外，如果工作时检测到配置文件无效，作为默认，会尝试播放这个文件名。
#define CONF_LCD_ROTATION_DEFAULT  0        // LCD 的默认旋转方向，可以在配置文件中修改
#define DEEP_SLEEP_SHORT_S  6               // 在最近一次敲击唤醒之后的 DEEP_SLEEP_SHORT_CNT 次睡眠前，会设置定时器使用 DEEP_SLEEP_SHORT_S 作为唤醒延时；否则使用 DEEP_SLEEP_LONG_S
#define DEEP_SLEEP_SHORT_CNT  6             // 每次检测到敲击唤醒后重置计数器
#define DEEP_SLEEP_LONG_S   30              // DEEP_SLEEP_LONG_S 和 DEEP_SLEEP_SHORT_S 是定时器唤醒的时间，单位是秒
#define BAT_ADC_THRESH_LOW  1700            // 低电压保护。当每一轮工作开始时，如果 ADC 读取到的电压读数低于这个值，就不会播放 GIF，直接进入下一轮睡眠
                                            // 由于 ESP32 的 ADC 质量不咋样，并且电池的截止电压也各自不同，所以没有换算成电压。
                                            // 你可以根据自己板子的情况尝试调节这个值。  

// 屏幕配置
//Arduino_GC9A01  *gfx = new Arduino_GC9A01(bus, PIN_TFT_RST, 2, true);
Arduino_ST7789  *gfx = new Arduino_ST7789(bus, PIN_TFT_RST, 0, true, 240, 240, 0, 0, 0, 80);
                                            // 在这里选择你的屏幕种类
#define LCD_BACKLIGHT_MIN_8B  16            // 屏幕背光的最低亮度，范围为 8bit（即最大值为 255）。
                                            // 每次运行时，程序将从环境光传感器那里得到的光强度 Map 到 LCD_BACKLIGHT_MIN_8B ~ 255 区间内，作为背光 PWM 的占空比。
#define LCD_PWM_FIR_LEN 32                  // 让背光的动态调节不要那么突然

// 网络设置
const char* wifi_ssid = "Celestia";         // 冷启动时尝试连接的 Wi-Fi 名称和密码
const char* wifi_pwd = "mimitomo";
#define WIFI_CONNECT_TIMEOUT_S 20           // 冷启动时尝试连接 Wi-Fi 的时间上限，单位为秒。
                                            // 超过这个时间还没有连接成功则直接进入正常工作，也可以用按键跳过
const char* wifi_host = "vision";           // MDNS 主机名。这样设置得到的结果形如 vision.local                             

```
## 参考 
[moononournation/RGB565_video](https://github.com/moononournation/RGB565_video)   
ESP32 Arduino 中的 SDWebServer 和 OTAWebUpdater 示例  
雷元素图标来自 [Bilibili: 鱼翅翅Kira](https://space.bilibili.com/2292091)  

## 免责声明  
这段代码仅用作测试，由于滥用造成的一切不好的后果和作者无关。  
如果其中的图片素材涉及侵权，请联系我删除。  
这段代码仅调通了功能，结构十分混乱，如有引发不适与我无关。  
