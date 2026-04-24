**# OriginPro Tricky Notes**

[TOC]

可用验证版本：2024b

## 批量修改Graph中不同线的范围

```labtalk
doc -e D {set %C -b 1; set %C -e 1000}
```

逐段含义  
1. `doc -e D`  
   - `doc -e` 是 LabTalk 的“document execute”循环指令。  
   - 关键字 `D` 告诉解释器：遍历级别是 **Graph**（=Graph 窗口）。  
   结果：后面大括号里的代码会对 **项目中的每一张 Graph 页面** 各执行一次。
   - `W` = Worksheet  
   - `M` = Matrix  
   - `L` = Layer（循环所有 Graph 的所有图层）  
   - `P` = Project（仅执行一次，相当于整个项目范围）
   
2. `{ ... }`  
   每次进入一张 Graph 时，大括号里的两行立即执行。

3. `set %C -b 1;`  
   - `%C` 在 Graph 循环体内自动指向 **当前被循环到的 Graph 的当前图层**（Layer）。  
   - `-b 1` 把该图层的 **Begin Row**（起始行）锁为第 1 行。  

4. `set %C -e 1000;`  
   - 同理，`-e 1000` 把同一图层的 **End Row**（终止行）锁为第 1000 行。  

由于 `set %C -b / -e` 是对 **整个图层** 下命令，Origin 会把该层里 **所有 plot（含 g1 组）** 一次性同步成同一行段，因此达到“批量改范围”的效果。  
官方文档中把 `-b`、`-e` 描述为 “Set beginning/ending row of all plots in the active layer”，正是此用法。

## 批量修改线宽

针对全是line的情况，如果时别的需要再改。

> origin的线宽有内部转换，需要linewith*500。如界面设置2，则需要输入内部值1000

```labtalk
doc -e L { set %C -w 1000; }; doc -uw;
```

说明：选中后，按层遍历，线宽2，然后文档自动更新（doc  -uw)

## 批量绘制SheetInBook

```labtalk

[PlotVM_SheetsInBook]

string bk$ = %H;

// ★ 先锁定sheet数量（在任何graph生成前）
win -a %(bk$);
int ns = page.nlayers;

for(int i = 1; i <= ns; i++)
{
    win -a %(bk$);     // 每次强制回到workbook
    page.active = i;        // 激活第 i 个sheet

    if(wks.maxRows <= 1 || wks.nCols < 2)
        continue;

    string sh$ = wks.name$;   // 在绘图前保存

    worksheet -s 0 0 wks.nCols-1 wks.maxRows-1;

    plotvm rowpos:=selrow1 colpos:=selcol1 vmname:=$(sh$) ogl:=<new template:="blue2D">; // 绘图选项，该例绘制的是vm

    page.longname$ = sh$ + "_VM";

    // 不改short name，避免readonly错误
}
```

## 批量绘制XYXYXYinSheet

```labtalk
string bk$ = %H;
string sh$ = wks.name$;

int n = wks.nCols;

for (ii = 1; ii <= n-1; ii+=2) {

    win -a %(bk$);
    page.active$ = sh$;

    int c1 = ii;
    int c2 = ii+1;

    plotxy iy:=($(c1),$(c2)) plot:=200;
	themeApply2g ShLiu_Frame;
	legend -r;
	legend.SMARTPOS=1;
	legend.SHOWFRAME=0;
	legend.TRANSPARENCY=100;
	
	sec -p 0.1;// 等待ui反应过来，否则graph fit to page不生效
	gfitp margin:=2 aspect:=1;

    win -r %H "Graph_$(c1)_$(c2)";
    // 使用 -p 参数，坐标范围是 0-100
    // 这里的 10 90 表示图层左边起 10%，底边起 90%（即左上角）
    label -p 10 90 -n myNote "Batch Processed";
    myNote.fsize = 15;
    // 🔥 防卡顿
    sec -p 0.3;
}

```

## 循环设置列注释

```labtalk
// 用wcol动态设置
loop(ii, 1, wks.ncols) {
    wcol(ii)[C]$ = "Data $(ii)"; 
};

```

每隔两列，如果是纯数字，则转换为100-X

```labtalk
int nStart = 2; 

for(int ii = nStart; ii <= wks.ncols; ii = ii + 2) {
    // 1. 暴力提取数值
    double valA = [%(wcol(ii)[C]$ )]; 
       
    if (valA != NANUM) {
        double res = 100 - valA;
        
        // 2. 先把复杂的格式拼接到字符串变量中
        // 在字符串变量里，%% 是 %，\-(2) 是下标 2
        string strFinal$ = "$(res)%% CO\-(2)";
        
        // 3. 直接赋值变量，避开直接解析文本时的转义错误
        wcol(ii)[C]$ = strFinal$;
        
        type "第 $(ii) 列成功：原值 $(valA) -> 新值 %(strFinal$)";
    } else {
        type "第 $(ii) 列跳过：不是有效数字";
    }
}

```

## 列注释是数组

```labtalk
string strList$ = "100 90 89 85 84 80 75 74 70 60 50 40 30 20 10 0";
int nToken = 1;
int nTotal = wks.ncols;

type "开始处理，总列数: $(nTotal)";

for(int ii = 2; ii <= nTotal; ii = ii + 2) {
    // 1. 取词
    string strVal$ = strList.GetToken(nToken, " ")$;
    
    // 如果取词为空则停止（防止列数多于数字个数）
    if (strVal.Length() == 0) break;

    // 2. 数值计算
    // 使用 %() 确保将字符串转为数值
    double valA = %(strVal$); 
    double res = 100 - valA;

    // 3. 赋值 (注意 \% 代表百分号符号)
    // 这里的 $(res) 会自动转为字符串内容
    string strComment$ = "$(res)%% CO\-(2)";
    
    wcol(ii-1)[C]$ = strComment$;
    wcol(ii)[C]$ = strComment$;
    
    type "Loop ii=$(ii): 原值=%(strVal$), 计算值=$(res), 设置为:%(strComment$)";
    
    nToken++;
}

type "全部处理完成！";
doc -uw;
```



## 添加text标签

```labtalk
// 使用 -p 参数，坐标范围是 0-100
// 这里的 10 90 表示图层左边起 10%，底边起 90%（即左上角）
label -p 10 90 -n myNote "Batch Processed";
myNote.fsize = 15
myNote.text$ = "Hello Origin";
```

