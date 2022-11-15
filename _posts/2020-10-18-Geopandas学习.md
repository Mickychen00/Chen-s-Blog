---
layout: post
title: Geopandas学习笔记
toc: true
tags: [GIS,Geopandas,可视化]
author: Chen Huang 
categories: python
permalink: /:categories/:year/:month/:day/:title.html
---

最近学习了Geopandas的一些基本理念和操作。此文章是为我的学习记录。Geopandas起源于Pandas，也同时整合了许多其他依赖包的功能。目前来看，Geopandas的功能能够初步满足我对数据分析的地理化呈现的基本要求。
<!--more-->
## 基本功能与语法
### Geopandas的引入

``` python
import pandas as pd 
import geopandas as gpd
from shapely.geometry import Point, Polygon
import matplotlib.pyplot as plt
```

### Geopandas的数据读取
#### Shapefile文件读取
Geopandas能够直接读取shapefile文件，此处我以中国基础地图数据为例
``` python
pro = gpd.read_file('/Users/chen/Desktop/中国基础地图数据/中国行政区.shp') 
pro.head()
```
head集数据包含了省份名称和最重要的geometry列数据
#### CSV文件读取
此处我们以中国电厂数据库为例，首先需要用pandas读取csv文件：
``` python
pwplant = pd.read_csv('/users/chen/Onedrive/Economic data/global_power_plant_database.csv')
pw_china = pwplant[pwplant.country_long=='China'] #选出中国数据
point_with_cap = pw_china[['latitude','longitude','capacity_mw']] #读取其中的经纬度和容量数据
point_withcap.reset_index(inplace=True,drop=True)
point = point_with_cap.apply(lambda row: Point(row.longitude,row.latitude),axis=1) #以列为方向，使用shapely的Point类将经纬度转换为point点
powerplants = gpd.GeoDataFrame(point_with_cap,geometry = point) #直到此处才转为Geopandas数据集
```
#### CRS参考坐标系统
参考坐标系统是每一套地理数据的标准，如果两套地理数据的CRS不同，那么二者是无法合并和比较的。比较常用的CRS标准为 epsg 4326
这里我们将读取的火电厂数据的crs默认值改为4326
``` python
powerplants.crs = {'init':'epsg:4326'}
```
### 初步绘图
在此我绘制四个子图例子
``` python
fig = plt.figure(figsize=(20,20))
fig_spec = fig.add_gridspec(nrows=2,ncols=2,width_ratios=[1,1]) #一个幕布下的四块，我很喜欢的子图切割方式
ax0 = plt.subplot(fig_spec[0])
ax1 = plt.subplot(fig_spec[1])
ax2 = plt.subplot(fig_spec[2])
ax3 = plt.subplot(fig_spec[3])
pro.plot(ax=ax0,color='grey',edgecolor='white',linewidth=0.5,alpha=0.5)
pro.to_crs(epsg=4326).plot(ax=ax1,cmap='Reds')
powerplants.plot(ax=ax2,color='green',markersize=5,alpha=0.5)
powerplants.plot(ax=ax3,color='blue',markersize=4,alpha=0.5)
```
如果运行正常，将如图所示
![四个子图](/assets/plants_subplot.jpg)

## Polygon和Point的合并
根据之前我们之前的Shapefile文件和CSV文件的导入，以及使用Geopandas和Pandas的预处理（如经纬度转为Point，CRS转换工作等），我们现在可以开展面数据和点数据的合并工作了。这里我们将使用Geopandas中的 **sjoin** 函数
### 点与面的合并
``` python
plants_with_pro = gpd.sjoin(powerplants,pro,how='inner',op='within') #注意此处使用within
plants_with_pro.head()
```
### 面与点的合并
``` python
pro_with_plants = gpd.sjoin(pro,powerplants,how='inner',op='contains') #注意此处使用contains
pro_with_plants.head()
```
### 统计、简化和绘图
正常的话，以上两种合并方式所获得的数据规模应该是一样的
``` python
print(plants_with_pro.shape)
print(pro_with_plants.shape)
```
我本人所显示的是(2938,15)
在绘图之前，我们可能会想到一个问题，那就是我们可能需要2900多个点绘制，但是我们不需要这么多次的面绘制。这里就需要对所合并数据集进行简化。以一个省份的电厂数量为例，我们进行数据简化和绘图。

``` python
plants_counts = pd.DataFrame(pro_with_plants['NAME'].value_counts()) #统计每个省份的电厂数量
plants_counts.reset_index(inplace=True)
plants_counts.head()
```
显示的结果应该为：

|   | index | NAME |
|:---:| :---: | :---: |
| 0 | 内蒙古 | 295  |
|1|四川|228|
|2|云南|212|
|3|山东|159|
|4|新疆|148|


``` python
plants_counts.rename(columns = {'index':'name','NAME':'plants_num'},inplace=True) #这里改一下列名
```
至此为止，合并数据集已经发挥了它的全部使命，即将电厂与省份匹配然后计算每一省份的电厂数量。
接下来，需要的是将统计好的数量数据与原有 **pro** 对象合并
``` python
pro = pd.merge(pro,plants_counts,left_on='NAME',right_on='name',how='outer').drop('name',axis=1) #此处的pandas数据合并值得铭记，这里不加how声明，将会漏掉我国台湾省地区
pro.head() 
type(pro) #这里的pro依然还是geopandas数据，尽管经历了pandas的处理，这也说明二者的同源性
```
这里我们跳过列名重命名过程，直到这里，我们就可以绘制一个反应各地区电厂数量差异的地图了。

![各省份电厂数量](/assets/pro_with_plants_counts.jpg)

这是我第一篇学习笔记，写了大概一小时，接下来会记录python多进程运行的笔记
