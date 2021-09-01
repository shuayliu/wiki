# Python Tricky Notes



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

