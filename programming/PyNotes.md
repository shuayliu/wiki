# Python Tricky Notes

[TOC]

## 读取EIS测量的数据

```python
import numpy as np
import os,sys

DATAFORMAT = {"tab":'\t',
              "space":' ',
              "comma":','
              }
DELIMITER = 'tab'
SKIPROWS = 1
POTENTIAL = 1
IMPORT_TYPE = "Zreal_Zimag"
FREQ_RANGE=[0,10000]


def loadData(filename,IMType = IMPORT_TYPE):
    if POTENTIAL == 0:
        data = np.loadtxt(filename, skiprows=SKIPROWS,
                          delimiter=DATAFORMAT[DELIMITER]).T
    elif POTENTIAL == 1:
        with open(filename, encoding='utf8') as _f:
            d = _f.read().splitlines()[SKIPROWS:]

    data = [_d.split(DATAFORMAT[DELIMITER]) for _d in d]
    potential = float(data[0][0])

    # 现在的数据是带有电位的data
    # 提取出电位，将剩下的转换为fre，zr，zi的数据格式
    data = np.array([_d[1:] for _d in data 
                        if FREQ_RANGE[0]<=float(_d[1])<FREQ_RANGE[1]],
                    dtype=np.float64).T


    if IMType == "Zreal_Zimag":
        try:
            freq,Zreal,Zimg = data[0:3]
        except:
            print("Data is not in right columns")
            return
    elif IMType == "Z_Phase":
        # 如果输入phase 和Z
        freq= data[0]
        Rad = data[2]
        ZMod= data[1]
        Zreal=ZMod * np.cos(Rad)
        Zimg =ZMod * np.sin(Rad) 
    
    return freq,Zreal,Zimg,potential
```



## 与pyDRTTool联合对EIS数据进行DRT处理

[F. Ciucci‘s Site](https://sites.google.com/site/drttools/download)

```python
import DRT_main as drt

def calculateDRT(filename):
    freq,Zreal,Zimg,potential =loadData(filename)
    eisObj = drt.Simple_run(drt.EIS_object(freq,Zreal,Zimg),
                    rbf_type='Gaussian',
                    data_used='Combined Re-Im Data',
                    induct_used=1,
                    der_used='1st order',
                    lambda_value=1E-3,
                    shape_control='FWHM Coefficient',
                    coeff=0.5)
    return potential,eisObj.out_tau_vec,eisObj.gamma
```

To run the Python version DRTtools, you need:

- Python 3
- Numpy, Pandas, Scipy, Matplotlib, PyQt5, and CVXPY

The DRTtools toolbox was tested and implemented on a Windows-based machine. 

Detailed installation instructions are available in the user's guide (also included with the standard distribution).

[1] T.H. Wan, M. Saccoccio, C. Chen, F. Ciucci, Influence of the Discretization Methods on the Distribution of Relaxation Times Deconvolution: Implementing Radial Basis Functions with DRTtools, Electrochimica Acta, 184 (2015) 483-499.

## batch 处理文件夹

```python
def batchCalculate(dirr):
    if not os.path.isdir(dirr):
        return
    else:
        print("data is dir")
        sPath = os.path.join(dirr,'result')
        if not os.path.isdir(sPath):
            os.mkdir(sPath)
        sys.stdout.write("#"*int(81)+'|')
        j=0
        dirlst = os.listdir(dirr)
        for file in dirlst:
            _filename = os.path.join(dirr,file)
            if os.path.isfile(_filename):
                potential,tau,gamma = calculateDRT(_filename)
                
                # Save file
                np.savetxt(os.path.join(sPath,"DRT_"+file),
                            X=np.array([tau,gamma]).T,
                            delimiter='\t',
                            header='Tau(s)\tGamma(Ohm)\n#Potential=%s'%potential,
                            comments='')

                j+=1
                sys.stdout.write('\r'+(j*80//len(dirlst))*'-'+'->|'+"\b")
```



## 去除重复 removeDuplicates

```python
def removeDuplicates(srcDir,dstDir):
    dirlst = os.listdir(srcDir)
    if dstDir == None:
        dstDir = os.path.join(srcDir,'SPLITE')
    if not os.path.exists(dstDir):
        os.mkdir(dstDir)
    E_lst = []
    for file in dirlst:
        absFilePath = os.path.join(srcDir,file)
        if os.path.isfile(absFilePath):
            with open(absFilePath, encoding='utf8') as _f:
                _f.readline()
                d =_f.readline().split('=')[-1].replace('\n','')
            if not d in E_lst:
                E_lst.append(d)
                os.system('copy %s %s'%(os.path.join(srcDir,file).replace('\\','\\\\').replace('/','\\\\'),
                                        os.path.join(dstDir,file).replace('\\','\\\\').replace('/','\\\\')))   

```



## 处理多个文件夹下的EIS，并挑选数据

准备工作

```bash
cd H:/PythonProject/pyDRTtools/
bpath = "K:/DATA"
```
Python Code

```python
import jDRT_batch as batch
import time,os

tstart=time.time()
for file in os.listdir(bpath):
     srcDir = os.path.join(bpath,file)
     batch.batchCalculate(srcDir)
     batch.removeDuplicates(os.path.join(srcDir,'result'),os.path.join(srcDir,'SPLIT'))
print("USING TIME--------------%s--------------TIME USING"%(time.time()-tstart))
```





## 整合进Originlab

预先打开OriginLab 的COM接口

```python
import OriginExt as oe
import sys
app = oe.Application()
app.Visible = 0 # app.Visible=1时，time using 46s，app.Visible=0时，time using 18s
```

参考文档：[OriginLab官方](https://www.originlab.com/doc/ExternalPython/OriginExt)

批处理部分

```python
tstart = time.time()
print("Let's Begin------------------------")
for dirr in os.listdir(bpath):
     wbook = app.Pages(app.CreatePage(app.OPT_WORKSHEET, dirr, "Origin"))
     wbook.Layers(0).Destroy()
     splitedPath = os.path.join(os.path.join(bpath, dirr), 'SPLIT')
     print("\n--------------\nProcessing DIR:%s"%splitedPath)

     for file in os.listdir(splitedPath):
        wks = wbook.AddLayer(file)

        sys.stdout.write("\rProcessing file: %s"%file)

        with open(os.path.join(splitedPath, file)) as _f:
            d = _f.read().splitlines()
        
        data = np.array([_d.split('\t') for _d in d[2:]], dtype=np.float64).T
        wks.SetData(data)
        
        # 设置Headers和Column Name
        headers = d[0].split('\t')
        comments = d[1].replace('#','')
        for ii in range(0, len(headers)):
            wks.Columns(ii).SetLongName(headers[ii])
            wks.Columns(ii).SetComments(comments)

        sys.stdout.write("\rProcessed File: %s "%file)
print("\nUSING TIME--------------%s--------------TIME USING" %(time.time()-tstart))

```

退出和保存

```python
app.Save(R"K:\DRT-full.opgu")
app.Exit()
del app
```



## 爬取html页面，通过xpath提取指定信息

```python
import requests,html
from time import sleep
from lxml import etree

for ids in range(500, 550):
    _url = 'http://m.chinayabisi.com/music/show-'+str(ids)+'.html'
    _dom = requests.get(_url)
    _tree = etree.HTML(_dom.text, etree.HTMLParser(encoding='utf-8'))
    _title = _tree.xpath('//*[@id="activity-name"]')[0].text.strip()
    _content = html.unescape(etree.tostring(_tree.xpath(
        '/html/body/div[1]/div[2]/div[1]/div/div[3]')[0], method='html').decode('utf-8'))
    with open('K:\'+_title+'.md', mode='w',encoding='utf-8') as _f:
        _f.write(html2text.html2text(_content))
    print(_url)
    print(_title)
    sleep(1.53)
```



## 从微信公众号的页面提取信息，将其替换为网站内容（厦园雅歌）

```python
import requests
import os
import bs4
from bs4 import BeautifulSoup
from time import sleep
from random import randint

def getContent(url):
    try:
        r=requests.get(url,timeout=20)
        r.raise_for_status()
        return r.content
    except:
        return ""

def writeFile(path,content):
    if not os.path.exists(path):
        with open(path,"wb") as file:
            file.write(content)
    else:pass



def getImg(text):
    urlList = []
    soup = BeautifulSoup(text,"html.parser")
    tag = soup.find_all('img',class_="")
    for item in tag:
        if isinstance(item, bs4.element.Tag):
            try:
                src = item.get('src')
            except:
                continue
#            print("获取到图片地址："+src)
            urlList.append(src)
    return urlList

def getImgFile(root,text):
    for url in getImg(text):            
        extName = '.'+url.split('=')[-1]
        picName=url.split('/')[-2] + extName
        path = root+'/' + picName
        try:
            if not os.path.exists(root):
                os.mkdir(root)
            if not os.path.exists(path):
                r = requests.get(url)
                sleep(randint(1,5000)/1000)
                with open(path, "wb") as f:
                    f.write(r.content)
                    f.close()
                    print(picName + "已经保存成功！")
#            else:
#                print(picName + "已存在！")
        except:
            print("爬取失败！")         
            return url

html="k:/121.html"
d = open(html,encoding='utf-8').read()

getImg(open("k:/html/厦园雅歌_2018-12-16_当我们在买买买时，我们在买什么？.html",encoding='utf-8').read())


names =[('k:/html/'+name) for name in os.listdir('k:/html') if name.endswith('.html')]
ret=[]
for name in names:
    ret.append(getImgFile('k:/usr/uploads/imgs',open(name,encoding='utf-8').read()))
    
    
#修复无法爬取的错误    
nret = [i for i in ret if i!= []]
urllst =[nr.replace('?x-oss-process=style/xmorient','') for nr in nret[0]]
urllst.extend([nr.replace('?x-oss-process=style/xmorient','') for nr in nret[1]])

urllst.extend([nr.replace('/0','/0?wx_fmt=jpeg') for nr in nret[2]])
urllst.extend([nr.replace('/0/jpeg','/0?wx_fmt=jpeg') for nr in nret[3]])
    
root = 'k:/usr/uploads/imgs'
for url in urllst:            
    extName = '.'+url.split('=')[-1]
    picName=url.split('/')[-2] + extName
    path = root+'/' + picName
    try:
        if not os.path.exists(root):
            os.mkdir(root)
        if not os.path.exists(path):
            r = requests.get(url)
            sleep(randint(1,5000)/1000)
            with open(path, "wb") as f:
                f.write(r.content)
                f.close()
                print(picName + "已经保存成功！")
#            else:
#                print(picName + "已存在！")
    except:
        print(url)
        print("爬取失败！")         




#替换标签和href连接
        
        
f = open("k:/121.html",encoding='utf-8',mode='r')
html = f.read()
html = html.replace("section","div")
#url替换
html = html.replace('http://mmbiz.qpic.cn/mmbiz_jpg/','/usr/uploads/imgs/').replace('http://mmbiz.qpic.cn/mmbiz_png/','/usr/uploads/imgs/')
html = html.replace('https://mmbiz.qpic.cn/mmbiz_jpg/','/usr/uploads/imgs/').replace('https://mmbiz.qpic.cn/mmbiz_png/','/usr/uploads/imgs/')
html = html.replace('http://mmsns.qpic.cn/mmsns','usr/uploads/imgs')
#后缀替换
html = html.replace('/0\"','/0?wx_fmt=jpeg\"')
html = html.replace('?x-oss-process=style/xmorient\"','\"')
html = html.replace('/0/jpeg\"','/0?wx_fmt=jpeg\"')
html = html.replace('/0?wx_fmt=jpeg\"','.jpeg\"')
html = html.replace('/0?wx_fmt=png\"','.png\"')
f.close()
f = open("k:/121-1.html",encoding='utf-8',mode='w')
f.write(html)

for name in names:
    f = open(name,encoding='utf-8')
    html = f.read()
    html = html.replace("section","div")
    #url替换
    html = html.replace('http://mmbiz.qpic.cn/mmbiz_jpg/','/usr/uploads/imgs/').replace('http://mmbiz.qpic.cn/mmbiz_png/','/usr/uploads/imgs/')
    html = html.replace('https://mmbiz.qpic.cn/mmbiz_jpg/','/usr/uploads/imgs/').replace('https://mmbiz.qpic.cn/mmbiz_png/','/usr/uploads/imgs/')
    html = html.replace('http://mmsns.qpic.cn/mmsns','usr/uploads/imgs')
    #后缀替换
    html = html.replace('/0\"','/0?wx_fmt=jpeg\"')
    html = html.replace('?x-oss-process=style/xmorient\"','\"')
    html = html.replace('/0/jpeg\"','/0?wx_fmt=jpeg\"')
    html = html.replace('/0?wx_fmt=jpeg\"','.jpeg\"')
    html = html.replace('/0?wx_fmt=png\"','.png\"')
    html = html.replace('/640?wx_fmt=jpeg\"','.jpeg\"')
    html = html.replace('/640?wx_fmt=png\"','.png\"')

    f.close()
    f = open(name,encoding='utf-8',mode='w')
    f.write(html)
    
    
#处理数据库
import sqlite3
conn = sqlite3.connect('k:/60d6c1e1a772d.db')
c = conn.cursor()
html = open('k:/121.html',encoding='utf-8').read()
soup = BeautifulSoup(html,"html.parser")
tag = soup.find('div',class_="rich_media")
text = "%s"%tag
c.execute("INSERT INTO songs_contents (title,slug,text) VALUES ('Python测试','test','%s')"%text)
c.execute(sql)
c.close()

import sqlite3
import time
conn = sqlite3.connect('k:/60d6c1e1a772d.db')
c = conn.cursor()
count = 12;
errLst = []
names =[('k:/html/'+name) for name in os.listdir('k:/html') if name.endswith('.html')]

for name in names:
    try:
        timestr,title=name.split('_')[1:]
        title = title.replace('.html','')
        timecr = int(time.mktime(time.strptime(timestr +" 09:00:00","%Y-%m-%d %H:%M:%S")))
        
        
        content = "%s"%BeautifulSoup(open(name,encoding='utf-8').read(),
                             "html.parser").find('div',class_="rich_media")
        content = content.replace('\'','"')
        
        slug = "%s"%count
        count = count + 1
        c.execute("INSERT INTO songs_contents (title,slug,created,modified,text,authorID) VALUES (?,?,?,?,'%s',1)"%content
                  ,(title,slug,timecr,timecr))
    except:
        errLst.append(name)
        print(name+"遇到了错误")
            
conn.commit()
```



## 阻抗转复数电容

```python
def impedance2ComplexCapacitance(Frequency, Zreal, Zimag, Potential):
  
    #numpy potimisiztion
    Zmod2 = (Zreal**2 + Zimag**2) * np.pi * 2 * Frequency
    Creal = Zimag/Zmod2
    Cimag = Zreal/Zmod2
    
    return Frequency,Creal,Cimag,Potential
```



## 复电容图计算界面电容

```python
def findCapacitanceAtImaginaryCapacitancePeak(Frequency,Creal,Cimag,Potential):
    # from scipy.signal import argrelmax
    from scipy.signal import find_peaks
    peaks,_ = find_peaks(Cimag)
    # peaks = argrelmax(Cimag)
    if not len(peaks) == 0:
        return Potential, Creal[peaks[0]],Frequency[peaks[0]]
    else:
        print("Cannot find peaks at potential %s"%Potential)
        return -999,-999,-999
```



## 整合计算电容的batch

```python
def load2ComplexCapacitance(_filename):
    freq,Zreal,Zimg,potential = loadData(_filename)
    return impedance2ComplexCapacitance(freq,Zreal,Zimg,potential)


def batchCalculate(dirr):
    if not os.path.isdir(dirr):
        return
    else:
        print("data is dir")
        sPath = os.path.join(dirr,'result')
        if not os.path.isdir(sPath):
            os.mkdir(sPath)
        sys.stdout.write("#"*int(81)+'|')
        j=0
        dirlst = os.listdir(dirr)
        
        result = np.array([])
        for file in dirlst:
            _filename = os.path.join(dirr,file)
            if os.path.isfile(_filename):
                
                # Frequency,Creal,Cimag,Potential = load2ComplexCapacitance(_filename)
                
                # _,Cap,Freq = findCapacitanceAtImaginaryCapacitancePeak(Frequency, Creal, Cimag, Potential)
                
                # if Cap != -999:
                #     result = np.append(result,[Potential,Cap,Freq])
                
                #单频电容
                
                Freq,Zreal,Zimg,Potential = loadData(_filename)
                
                Frequency,Creal,Cimag,_ = impedance2ComplexCapacitance(Freq,Zreal,Zimg,Potential)

                # 单频率电容
                idx = np.where( (Freq>1290)&(Freq<1300) )
                Cap = 1.0/(2*np.pi*Freq[idx]*Zimg[idx])
                
                result = np.append(result,[Potential,Cap[0],Freq[idx][0]])
                # Save file
                np.savetxt(os.path.join(sPath,"ColeCole_"+file),
                            X=np.array([Frequency,Creal,Cimag]).T,
                            delimiter='\t',
                            header='Frequency(Hz)\tCreal(F)\tCimag(F)\n#Potential=%s'%Potential,
                            comments='')

                j+=1
                sys.stdout.write('\r'+(j*80//len(dirlst))*'-'+'->|'+"\b")
        
        # result = np.delete(result,np.where(result<-100))
        savedArray = np.array(result.reshape(len(result)//3,3))
        np.savetxt(os.path.join(sPath,"RESULT.dat"),
                   savedArray,
                   delimiter='\t',
                   header = 'Potential\tCreal\tFrequency\t',
                   comments='')
        
        data = pd.DataFrame(savedArray,
                            columns=['Potential','Creal','Frequency']
                            ).groupby('Potential').agg([np.mean,np.std])
        
        data.to_csv(os.path.join(sPath,"STATISTICS.dat"))
```

