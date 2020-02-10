# epidemic_map
# 背景
2020年伊始，我们经历了一个特别的春节。大部分人每天都会打开手机收看前一天的疫情，其中疫情图是各大 APP 最吸引眼球的地方。比如：
![-w628](media/丁香园.jpg)


又或者这样
![-w768](media/杭州.jpg)

我们每天在家闭门不出为国家做贡献的同时也可以持续不断深耕自己的技术，今天和大家分享篇疫情地图可视化。只需要一行命令，100 行源代码就可以画出各个省份的疫情图，此文章旨在帮助大家更加直观了解自己家乡的情况，顺带也对自己之前学习的知识进行实践和思考。

# 核心思想
疫情图的核心在于**疫情数据获取**以及**疫情数据可视化**

## 疫情数数据获取
疫情数据获取有各种各样的渠道，建议大家在**合法，合规**的情况下去获取，此文章数据是通过某易新闻的公开数据所获得的，数据仅是作为 demo 展示。具体的地址已在代码中说明，此地址是一个 Get 请求，大家可以先拷贝地址到浏览器中查看数据格式。
请求成功后服务端响应的数据格式（仅罗列我们需要的数据）：

| 参数  | 类型 | 备注 |
| --- | --- | --- |
| code | int |  |
| msg | String  |  |
| data | Object |  |

data 里面的参数

| 参数  | 类型 | 备注 |
| --- | --- | --- |
| list | List<Object> | 各地级市确诊人数 |

 
Object 里面的参数

| 参数  | 类型 | 备注 |
| --- | --- | --- |
|  name |  String | 地级市名(简称) |
| province  |String| 省份 |
| confirm  | int  | 确诊人数 |


## 数据可视化
数据可视化离不开地图的支持，地图的展示效果是采用了开源的 [pyecharts](http://pyecharts.herokuapp.com/) 项目，pyecharts 是一个用于生成 Echarts 图表的类库。Echarts 是百度开源的一个数据可视化 JS 库。用 Echarts 生成的图可视化效果非常棒，为了与 Python 进行对接，方便在 Python 中直接使用数据生成图。

# 详细步骤
搞清楚了核心思想，后面的内容是手把手教大家如何搭建环境，获取数据，使用 pyecharts 来做数据可视化，最后是调试项目。

## 环境搭建
为了快速开发此功能并且尽可能的缩减代码代码量，此 demo 选择使用 Python 来进行开发，所需要的 pyecharts 是对 Echarts 进行的延伸。我们只需要配置好 Python 的开发工具即可，在获取数据以及使用 pyecharts 还需要导入一些基础库。

### 安装 Python 环境
此处不多说，我相信你们可以的，Mac 上面是自带 Python2.7 的，也可以参考网上的一些[网上的安装教程](https://www.liaoxuefeng.com/wiki/1016959663602400/1016959856222624)

### 安装 pip
pip 是 Python 包管理工具，该工具提供了对Python 包的查找、下载、安装、卸载的功能。
目前如果你在 python.org 下载最新版本的安装包，则是已经自带了该工具。
Python 2.7.9 + 或 Python 3.4+ 以上版本都自带 pip 工具。
可以使用此命令行来查看当前 pip 的版本 
```
pip --version
```
如果找不到的话，问题也不大，这边我也给大家找好了在线安装的方法，只需要在终端输入以下 2 行命令即可
```
$ curl https://bootstrap.pypa.io/get-pip.py -o get-pip.py   # 下载安装脚本
$ sudo python get-pip.py    # 运行安装脚本
```


### 安装工程所需要的插件
在代码中大家会发现我们 import 一些开源的库
```
import math
import time
from fake_useragent import UserAgent 
from pyecharts.charts import Map
from pyecharts import options as opts
import requests
import json
import sys
```

请求网络数据需要用到的插件

```
pip install fake_useragent # 伪装请求，随机生成UserAgent
pip install requests # HTTP请求库。
```

地图展示需要用到的插件

```
pip install echarts-countries-pypkg  # 世界地图
pip install echarts-china-provinces-pypkg  # 中国省级地图
pip install echarts-china-cities-pypkg  # 中国城市地图
```
直接把上述的命令行拷贝到终端中，逐行执行即可

## 数据获取 & 过滤
代码非常的简介明了，直接使用 requests 库构建一个 GET 请求，服务器响应的数据即为 全国所有城市的疫情情况。
```
ua = UserAgent(verify_ssl=False)
headers = {'User-Agent': ua.random}
url = "https://c.m.163.com/ug/api/wuhan/app/index/feiyan-data-list?t=1580469818264"


def getEpidemicInfo(url):
    try:
        response = requests.get(url, headers=headers)
        print(response.status_code)
        if response.status_code == 200:
            content_field = json.loads(response.text)
            epidemicInfo = content_field['data']['list']
            return epidemicInfo
        else:
            print('请求错误码：' + response.status_code)
            return None
    except Exception as e:
        print('此页有问题！', e)
        return None

```
请求地址里面的 t 代表时间戳，实战过程中发现此参数没啥实际意义
数据格式已经在上文中陈述了。接下去就比较简单了，我们拿到数据后需要对数据进行一定的过滤，我们仅需要**某个省份的所有城市的疫情信息即可**

代码依旧非常的简洁，把上面获取到的数据过滤成我们需要的数据

```
# 生成本省疫情列表
def makedict(list):
    cityList = {}
    for item in list:
        for k, v in item.items():
        # 1
            if v == sys.argv[1]:
            #2
                if str(item["confirm"]).isdigit():
                # 3
                    if v == "北京" or v == "上海" or v == "天津":
                        cityList[item['name'] + '区'] = int(item["confirm"])
                    elif "自治州" in v:
                        continue
                    else:
                        cityList[item['name'] + '市'] = int(item["confirm"])
    return cityList
```
1. sys.argv[1] 是一个传参，代表我们手动输入的省份，比如浙江，江西，湖北等
2. “confirm” 关键字用于匹配响应内容里面的内容，在上文疫情数数据获取有提及，是代表当前城市的疫情人数
3. **pyecharts 是根据城市的完整名称来适配的**，此处需对特殊的地级市进行适配，防止地图展示异常

当我输入浙江的时候，最后得到的数据格式为

```
{'湖州市': 9, '丽水市': 16, '舟山市': 7, '衢州市': 15, '金华市': 47, '嘉兴市': 30, '绍兴市': 33, '宁波市': 126, '台州市': 124, '杭州市': 151, '温州市': 396}
```

## 数据可视化

到了最核心的一步了，对逐行代码进行分析
方法里面的 dict 对应我们过滤得到的数据

```
def makeEpidemicInfoMap(dict):

    # 省和直辖市
    province_distribution = dict
    value = province_distribution.values()
    print(province_distribution)
    title = str(int(time.strftime("%Y%m%d")) - 1) + sys.argv[1] + "疫情地图"
    epidemicCount = []
    for k, v in dict.items():
        epidemicCount.append(v)
    # 1
    epidemicCount.sort(reverse=True)
    maxEpidemic = handle(epidemicCount.pop(0))
    maxEpidemic = int(maxEpidemic)
    # 2
    map = Map()
    # 3
    map.set_global_opts(
        title_opts=opts.TitleOpts(title=title),
        visualmap_opts=opts.VisualMapOpts(max_=200, is_piecewise=True,
                                          pieces=[
                                              {"max": 9999999, "min": maxEpidemic, "label": ">" + str(maxEpidemic),
                                               "color": "#780707"},  # 数据范围分段，分颜色，可以根据数据大小具体分配大小
                                              {"max": int(maxEpidemic), "min": int(maxEpidemic / 8) * 7,
                                               "label": str(int(maxEpidemic / 8) * 7) + "-" + str(int(maxEpidemic)),
                                               "color": "#B40404"},
                                              {"max": int(maxEpidemic / 8) * 7, "min": int(maxEpidemic / 8) * 4,
                                               "label": str(int(maxEpidemic / 8) * 4) + "-" + str(
                                                   int(maxEpidemic / 8) * 7 - 1), "color": "#CD1111"},
                                              {"max": int(maxEpidemic / 8) * 4, "min": int(maxEpidemic / 8),
                                               "label": str(int(maxEpidemic / 8)) + "-" + str(
                                                   int(maxEpidemic / 8) * 4 - 1), "color": "#F68181"},
                                              {"max": int(maxEpidemic / 8), "min": 1,
                                               "label": "1-" + str(int(maxEpidemic / 8)), "color": "#F5A9A9"},
                                              {"max": 0, "min": 0, "label": "0", "color": "#FFFFFF"},
                                          ], )  # 最大数据范围，分段
    )
    # 4
    map.add(title, data_pair=province_distribution.items(), maptype=sys.argv[1], is_roam=True)
    map.render(sys.argv[1] + '疫情地图.html')

```
1. 根据疫情数量对所有城市进行降序排序并得到最大的疫情数，maxEpidemic 是最接近最大疫情数的高位数，比如最大疫情数为“357”，则 maxEpidemic=300 ，引入此参数的目的是为了让地图更加的直观
2. 用PyEcharts绘制地图需要先初始化一个Map对象，用于地理区域数据的可视化。
3. 对 map 进行设值，相当于一个建造者模式，VisualMapOpts 是 PyEcharts 的视觉映射配置项， 

```
    # 指定 visualMapPiecewise 组件的最大值。
    max =100
    # 是否为分段型
    is_piecewise: bool = False,
 
    # 自定义的每一段的范围，以及每一段的文字，以及每一段的特别的样式。例如：
    # pieces: [
    #   {"min": 1500}, // 不指定 max，表示 max 为无限大（Infinity）。
    #   {"min": 900, "max": 1500},
    #   {"min": 310, "max": 1000},
    #   {"min": 200, "max": 300},
    #   {"min": 10, "max": 200, "label": '10 到 200（自定义label）'},
    #   {"value": 123, "label": '123（自定义特殊颜色）', "color": 'grey'}, //表示 value 等于 123 的情况
    #   {"max": 5}     // 不指定 min，表示 min 为无限大（-Infinity）。
    # ]

```
 详细配置可以见 PyEcharts 官网。此处的范围分为6 段，每一段的范围均是根据上述计算出来的 maxEpidemic 进行动态调整的，目的是为了保证疫情图的视觉效果，这**里面我仅是做了非常简略的范围模型，仅供参考**
 
 4. 使用 PyEcharts 在当前目录下面生成一个网页


# 调试

执行 python  map.py [省份]，如

```
python /Users/xxx/map.py 浙江

```
会在当前目录下面得到一份名字为 **浙江疫情地图.html** 的文件，直接使用浏览器打开即可，最后的展示效果是不是很酷
![-w688](media/浙江疫情图.jpg)

![-w698](media/广东.jpg)

![-w665](media/福建.jpg)


# 特别提醒
部分省份地级市命名尾缀包含自治州和区的原因，数据展示可能存在异常，请各位小伙伴多多海涵。此篇文章是希望大家对地图绘制有个初步的了解，这块知识点比较小众，有兴趣的小伙伴可以自行深入学习或者也可以关注公众号后给我留言。
武汉加油! 中国加油 !
