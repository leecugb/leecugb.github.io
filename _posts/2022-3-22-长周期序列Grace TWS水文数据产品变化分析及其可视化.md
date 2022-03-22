# 长周期序列Grace TWS水文数据产品变化分析及其可视化

地球重力观测卫星Grace和Grace-FO的不同级别数据产品被科学家和其他兴趣用户广泛应用于研究地球系统的质量迁移。然而，将Grace/Grace-FO数据（Level-0级原始数据或Level-1/2低级地球引力场数据产品）处理成不同领域用户喜闻乐见的产品却不是一件容易的事情。为了扩大Grace/Grace-FO数据的应用群体，多家科研机构推出了Level-3级数据产品。

在这些Level-3级数据产品中，比较有影响力的是德国地学研究中心（German Research Centre for Geosciences， GFZ）的陆地水储量（terrestrial water storage over non-glaciated regions）数据产品。本文将介绍该水文数据产品的长周期变化分析和可视化方法。

## 数据格式
德国地学中心（GFZ）的Level-3级重力卫星水文数据产品存储为NetCDF（network Common Data Form）格式，每个文件包含``time``, ``lon``, ``lat``, ``tws``, ``std_tws``, ``leakage``, ``model_atmosphere``等7个数据子集。``tws``包含了一个``n x 180 x 360``的数组，代表了``n``个月度的水储量变化数据；``time``包含了``n``个月度的时间信息（相对于格林威治时间2002-4-18 00：00：00的天数时间差）；``lon``存储了一个``1 x 360``的数组，记录``tws``中数据的经度信息；``lat``存储了一个``1 x 180``的数组，记录了``tws``中数据的纬度信息。

## 数据处理
为了方便地处理德国地学中心的一百多个Level-3级重力卫星水文数据NetCDF文件，我们写了个简易且实用的脚本程序，使用方法如下：

### 应用示例
首先，导入该python包
```
>>> import GFZ_TWS
``` 
#### 下载数据
可以选择从http://gravis.gfz-potsdam.de/home手动下载，将下载好的数据文件集中放在某一路径下，设为rootDir。然后用rootDir初始化Reader对象。

```
>>> tws=GFZ_TWS.Reader(r'./tws/')
```
也可以选择使用工具包中Reader类的自动下载函数完成数据下载。
```
>>> tws.update()
```


#### 绘制区域陆地水储量变化折线图
通过调用tws.plot(vector)函数即可绘制区域陆地水储量变化折线图，vector为兴趣区矢量文件。
```
>>> data=tws.plot('./tws/mask.shp')

```
![](/images/grace_plot.png)


data变量存储了折线数据。

#### 可视化全球变化趋势
```
plt.imshow(tws.gradient)

``` 
![](/images/gradient.png)







#### 源代码





```python


import netCDF4 as nc
from osgeo import gdal, osr, ogr
import numpy as np
import sys, string, os, re
from datetime import datetime, timedelta
import tempfile
import matplotlib.pyplot as plt
from ftplib import FTP           

class Reader:
    def __init__(self,rootDir):
        self.__init_data(rootDir)
    
    
    def update(self,rootDir):
        ftp=FTP()
        ftp.connect('isdcftp.gfz-potsdam.de',21)
        ftp.voidcmd('TYPE I')
        ftp.login()
        ftp.cwd('grace/GravIS/GFZ/Level-3/TWS')
        for file in ftp.nlst():
            if re.match(r'GRAVIS.*?\.nc',file):
                os.system('wget '+'ftp://isdcftp.gfz-potsdam.de/grace/GravIS/GFZ/Level-3/TWS/'+file+' -t 0 -O '+os.path.join(rootDir,file))
        
    
    
    def __init_data(self, rootDir):
        start=datetime(2002,4,18,0,0,0)
        self.tws = np.array([])
        self.time = np.array([])
        for file in os.listdir(rootDir):
            if re.match(r'^GRAVIS.*?GFZ.*?\.nc$',file):
                pathname = os.path.join(rootDir, file)
                dataset = nc.Dataset(pathname)
                if self.tws.size == 0:
                    self.tws = dataset['tws'][:].data
                    self.time = np.array([start+timedelta(days=i) for i in dataset['time'][:].data])
                else:
                    self.tws = np.vstack([self.tws, dataset['tws'][:].data])
                    self.time = np.hstack([self.time, [start+timedelta(days=i) for i in dataset['time'][:].data]])
        temp=a.tws.reshape(-1,180*360)
        mask=(temp!=-9e33).all(0)
        X=np.vstack([[ (i-start).days for i in a.time],np.ones(temp.shape[0])])
        X=X.T
        self.gradient=np.zeros(180*360)*np.nan
        self.gradient[mask]=np.linalg.inv(X.T.dot(X)).dot(X.T.dot(   temp[:,mask]   ))[0]
        self.gradient=self.gradient.reshape(180,360)
        
    def save_file(self,data,filename):
        
        driver = gdal.GetDriverByName('GTiff')
        raster = driver.Create(filename, data.shape[1],data.shape[0], 1, gdal.GDT_Float32)
        raster.SetMetadataItem('AREA_OR_POINT', 'Point')
        raster.SetGeoTransform((0, 1, 0, 90, 0, -1))
        sr = osr.SpatialReference()
        sr.SetWellKnownGeogCS('WGS84')
        raster.SetProjection(sr.ExportToWkt())
        raster.GetRasterBand(1).WriteArray(  data*365      )
        raster.FlushCache()
        raster=None




    
    def plot(self,vector):
        mask=self.create_mask(vector)
        y=np.array([(lambda x:x[mask&(x!=-9e33)].mean())(np.repeat(np.repeat(tws,10,axis=0),10,axis=1)) for tws in self.tws])        
        temp=np.vstack([ self.time   ,y   ])
        temp=temp[:,np.argsort(temp[0])]
        plt.figure(figsize=(25,3))    
        plt.scatter(temp[0],temp[1],s=10)
        plt.plot(temp[0],temp[1])
        plt.grid()
        return temp
    
    
    def create_mask(self,vector,NoData_value = 0):
        vs=ogr.Open(vector)
        vs_lyr = vs.GetLayer()
        with tempfile.NamedTemporaryFile() as temp:
            ts = gdal.GetDriverByName('GTiff').Create(temp.name, 3600,1800, 1, gdal.GDT_Byte)
            ts.SetMetadataItem('AREA_OR_POINT', 'Point')
            sr = osr.SpatialReference()
            sr.SetWellKnownGeogCS('WGS84')
            ts.SetProjection(sr.ExportToWkt())
            ts.SetGeoTransform((0, 0.1, 0, 90, 0, -0.1))
            b1 = ts.GetRasterBand(1)
            b1.SetNoDataValue(NoData_value)
            gdal.RasterizeLayer(ts, [1], vs_lyr, burn_values=[100])
            mask=ts.GetRasterBand(1).ReadAsArray()
        return mask==100
```
