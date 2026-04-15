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

