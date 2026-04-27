# OriginPro 2024b LabTalk Tricky Notes (完整版)

[TOC]

可用验证版本：**2024b**

---

## 一、 批量图形属性调整 (Graph & Layer)

### 1. 批量修改 Graph 中不同线的范围
```labtalk
// 遍历项目中的每一个 Graph 窗口 (D)
doc -e D {
    set %C -b 1;    // 将当前图层所有 plot 的起始行 (Begin) 锁为第 1 行
    set %C -e 1000;  // 将当前图层所有 plot 的终止行 (End) 锁为第 1000 行
}
```

### 2. 批量修改线宽
```labtalk
// 遍历项目中的每一个图层 (L)；内部值 1000 对应界面上的线宽 2
doc -e L { set %C -w 1000; }; 

// 强制更新文档显示，使修改立即生效
doc -uw; 
```

---

## 二、 自动化批量绘图 (Batch Plotting)

### 1. 批量绘制 SheetInBook (针对每一个 Sheet 生成 VM 图)
```labtalk
[PlotVM_SheetsInBook]

string bk$ = %H;              // 记录当前 Workbook 名称

// ★ 先锁定 sheet 数量（在任何 graph 生成前）
win -a %(bk$);
int ns = page.nlayers;        // 获取总 Sheet 数量

for(int i = 1; i <= ns; i++)
{
    win -a %(bk$);            // 每次循环强制回到 workbook
    page.active = i;          // 激活第 i 个 sheet

    if(wks.maxRows <= 1 || wks.nCols < 2)
        continue;

    string sh$ = wks.name$;   // 在绘图前保存当前 Sheet 名

    worksheet -s 0 0 wks.nCols-1 wks.maxRows-1; // 选中当前 Sheet 所有数据

    // 绘图选项，该例绘制的是 vm，使用 blue2D 模板
    plotvm rowpos:=selrow1 colpos:=selcol1 vmname:=$(sh$) ogl:=<new template:="blue2D">; 

    page.longname$ = sh$ + "_VM"; // 设置 Graph 的长名称
}
```

### 2. 绘制所有 Sheet 中的某一列到同一个 Graph 里
```labtalk
string strBook$ = %H;         // 记录数据簿名称
win -a %(strBook$);           // 确保其处于激活状态

int nSheets = page.nlayers;   // 获取总 Sheet 数量
type "工作簿 %(strBook$) 中共有 $(nSheets) 个 Sheet";
type "--------------------------------";

int nCol = 14;                // 目标列号

// 1. 先创建一个新的折线图窗口
win -t plot line;
string targetGraph$ = %H;     // 记录新生成的 Graph 名称

// 2. 回到数据簿开始循环
win -a %(strBook$);

for(int ii = 1; ii <= nSheets; ii++) {
    // 强制切换当前激活的 Sheet
    page.active = ii; 
    string strSheet$ = wks.name$;
    
    type "正在处理第 $(ii) 个 Sheet: %(strSheet$)";

    if (wks.ncols >= nCol) {
        // 使用具体的索引定义 Range，确保引用准确
        range rData = [%(strBook$)]$(ii)!wcol(nCol);
        
        // 绘图：将数据添加到目标 Graph 的第 1 图层
        plotxy iy:=rData plot:=200 ogl:=[%(targetGraph$)]1!;
        type "   --> 已添加第 $(nCol) 列到图表";
    }
    else {
        type "   --> 跳过：列数不足";
    }
}

// 3. 图后美化处理
win -a %(targetGraph$);
legend.SHOWFRAME = 0;         // 隐藏图例边框
legend -r;                    // 刷新图例
legend.fsize = 24;            // 图例字号
sec -p 0.1;                   // 等待渲染
gfitp margin:=2 aspect:=1;    // 自动调整页面布局
themeApply2g theme:="ShLiu_Frame"; // 应用自定义主题
label -p 10 90 -n myNote "This is My Notes"; // 添加水印

doc -uw;                      // 刷新文档
```

### 3. 批量绘制 XYXYXY in Sheet (每个 XY 对生成一个独立 Graph)
```labtalk
string bk$ = %H;              // 记录当前 Workbook
string sh$ = wks.name$;       // 记录当前 Sheet

int n = wks.nCols;            // 获取列数

for (ii = 1; ii <= n-1; ii+=2) {

    win -a %(bk$);            // 强制回到数据源
    page.active$ = sh$;

    int c1 = ii;              // X 列
    int c2 = ii+1;            // Y 列

    // 绘制 XY 对
    plotxy iy:=($(c1),$(c2)) plot:=200;
    
    themeApply2g ShLiu_Frame; // 应用主题
    legend -r;
    legend.SMARTPOS=1;
    legend.SHOWFRAME=0;
    legend.TRANSPARENCY=100;
    
    sec -p 0.1;               // 等待 UI 反应，确保 gfitp 生效
    gfitp margin:=2 aspect:=1;

    win -r %H "Graph_$(c1)_$(c2)"; // 重命名窗口
    
    // 添加标签，位置：左边起 10%，底边起 90%
    label -p 10 90 -n myNote "Batch Processed";
    myNote.fsize = 15;
    
    sec -p 0.3;               // 防止执行过快导致 UI 卡顿
}
```

---

## 三、 工作表注释与批量填充 (Column Metadata)

### 1. 循环设置列注释 (基础用法)
```labtalk
// 使用 wcol 动态遍历所有列并设置 Comments [C]
loop(ii, 1, wks.ncols) {
    wcol(ii)[C]$ = "Data $(ii)"; 
};
```

### 2. 转换数值并设置特殊格式 (100-X)
```labtalk
int nStart = 2;               // 从第 2 列开始

for(int ii = nStart; ii <= wks.ncols; ii = ii + 2) {
    // 1. 提取当前列注释中的数值内容
    double valA = [%(wcol(ii)[C]$ )]; 
       
    if (valA != NANUM) {
        double res = 100 - valA; // 计算差值
        
        // 2. 拼接复杂格式：%% 转义为 %，\-(2) 转义为下标 2
        string strFinal$ = "$(res)%% CO\-(2)";
        
        // 3. 赋值给注释栏
        wcol(ii)[C]$ = strFinal$;
        
        type "第 $(ii) 列成功：原值 $(valA) -> 新值 %(strFinal$)";
    } else {
        type "第 $(ii) 列跳过：不是有效数字";
    }
}
```

### 3. 使用预设数组列表填充注释
```labtalk
// 定义预设的数值列表
string strList$ = "100 90 89 85 84 80 75 74 70 60 50 40 30 20 10 0";
int nToken = 1;
int nTotal = wks.ncols;

type "开始处理，总列数: $(nTotal)";

for(int ii = 2; ii <= nTotal; ii = ii + 2) {
    // 1. 从列表中取词
    string strVal$ = strList.GetToken(nToken, " ")$;
    
    // 如果取词为空则停止循环
    if (strVal.Length() == 0) break;

    // 2. 数值计算
    double valA = %(strVal$); 
    double res = 100 - valA;

    // 3. 构建格式化字符串 (X 列和 Y 列同时设置)
    string strComment$ = "$(res)%% CO\-(2)";
    
    wcol(ii-1)[C]$ = strComment$;
    wcol(ii)[C]$ = strComment$;
    
    type "Loop ii=$(ii): 原值=%(strVal$), 计算值=$(res), 设置为:%(strComment$)";
    
    nToken++;
}

type "全部处理完成！";
doc -uw;                      // 刷新界面
```

### 按comments的不一样设置公式

wcol没有很多属性，需要转换为range再操作

```labtalk
loop(ii, 1, wks.ncols) {
    // 只有偶数列才处理
    if (mod(ii, 2) == 0) {
        
        // 【关键修改】去拿前一列 (ii-1) 的 Comment 提取数字
        // 这样即使第二列 Comment 是空的，也能拿到第一列的数据
        double v = value(token(wcol(ii-1)[C]$, 1, "%")$);
        double factor = v / 100;

        if (factor != 0) {
            type -s "Column($(ii)): 结果为 $(factor)";
            range rCol = wcol(ii);
            rCol.formula$ = "Col($(ii)) / $(factor)";
            rCol.execute(); 
        } else {
            type -s "警告：Column($(ii)) 对应的前一列没有找到有效百分比！";
        }
    }
}
```



---

## 四、 跨 Sheet 批量填充 (Multiple Sheets)

### 1. 批量设置特定列注释 (从 Sheet 名提取)
```labtalk
string strBook$ = %H;         // 记录工作簿名称
win -a %(strBook$); 

int nSheets = page.nlayers;   // 总 Sheet 数量
int nCol = 11;                // 目标列号

type "开始批量设置注释...";

for(int ii = 1; ii <= nSheets; ii++) {
    // 1. 切换激活 Sheet
    page.active = ii; 
    
    // 2. 获取当前 Sheet 名称
    string strSheet$ = wks.name$; 
    
    // 3. 将名称写入指定列的 Comments [C]
    if (nCol <= wks.ncols) {
        wcol(nCol)[C]$ = strSheet$;
        type "第 $(ii) 个 Sheet: [%(strSheet$)] -> 已写入第 $(nCol) 列注释";
    }
    else {
        type "第 $(ii) 个 Sheet: [%(strSheet$)] -> 错误：列数不足，跳过";
    }
}

type "全部设置完成！";
doc -uw;
```

### 2. 批量填充全部列 (提取 Sheet 名中的数字段)
```labtalk
string strBook$ = %H; 
win -a %(strBook$); 

int nSheets = page.nlayers; 
type "正在处理工作簿: %(strBook$)";

for(int ii = 1; ii <= nSheets; ii++) {
    // 1. 激活当前 Sheet
    page.active = ii; 
   
    // 2. 提取 Sheet 名称中被 "-" 分隔的第 3 个片段
    string valNum$ = token(wks.name$, 3, "-")$; 
    
    // 3. 拼接单位
    string valWithUnit$ = valNum$ + " Hz"; 
    
    // 4. 赋值给该 Sheet 的所有列
    loop(jj, 1, wks.ncols) {
        wcol(jj)[C]$ = valWithUnit$;
    }
}

// 5. 强制更新文档链接
doc -uw; 
type "运行结束。";
```

---

## 五、 文本标签 (Label)

### 1. 添加与控制 Text 标签
```labtalk
// 使用 -p 参数设置坐标 (0-100)，10 90 为左上角
// -n 指定对象名，方便后续通过脚本调用属性
label -p 10 90 -n myNote "Batch Processed";

myNote.fsize = 15;            // 设置字号
myNote.text$ = "Hello Origin"; // 修改标签显示内容
```







# 自动化 ogs+.c文件

```labtalk
//////////////////////////////////////////////////////////
// JL_Commands.ogs - 个人科研绘图自动化脚本库
//////////////////////////////////////////////////////////

[Range]
// 批量修改 Graph 范围
int nB = 1; int nE = 1000;
if ("%1" != "") nB = %1;
if ("%2" != "") nE = %2;
doc -e D { set %C -b $(nB); set %C -e $(nE); }
doc -uw;
type -p "范围已设置为 $(nB) 到 $(nE)";

[Width]
// 批量修改线宽
double dW = 2;
if ("%1" != "") dW = %1;
int nIntW = dW * 500;
doc -e L { set %C -w $(nIntW); }; 
doc -uw;
type -p "线宽已统设为 $(dW)";

[PlotColN]
// 功能：绘制所有 Sheet 中的指定列到同一个 Graph
// 用法：JL_PlotColN [列号]
string strBook$ = %H;         // 记录数据簿名称
win -a %(strBook$);           // 确保其处于激活状态

int nSheets = page.nlayers;   // 获取总 Sheet 数量
type -s "工作簿 %(strBook$) 中共有 $(nSheets) 个 Sheet";
type -s "--------------------------------";

int nCol = 14;                // 默认目标列号
if ("%1" != "") nCol = %1;

// 1. 先创建一个新的折线图窗口
win -t plot line;
string targetGraph$ = %H;     // 记录新生成的 Graph 名称

// 2. 回到数据簿开始循环
win -a %(strBook$);

for(int ii = 1; ii <= nSheets; ii++) {
    win -a %(strBook$);
    page.active = ii;        // 强制切换当前激活的 Sheet
    string strSheet$ = wks.name$;
    
    type -s "正在处理第 $(ii) 个 Sheet: %(strSheet$)";

    if (wks.ncols >= nCol) {
        // 使用具体的索引定义 Range，确保引用准确
        range rData = [%(strBook$)]$(ii)!wcol(nCol);
        
        // 绘图：将数据添加到目标 Graph 的第 1 图层
        plotxy iy:=rData plot:=200 ogl:=[%(targetGraph$)]1!;
        type -s "   --> 已添加第 $(nCol) 列到图表";
    }
    else {
        type -s "   --> 跳过：列数不足";
    }
}

// 3. 图后美化处理
win -a %(targetGraph$);
legend.SHOWFRAME = 0;         // 隐藏图例边框
legend -r;                    // 刷新图例
legend.fsize = 24;            // 图例字号
sec -p 0.1;                   // 等待渲染
gfitp margin:=2 aspect:=1;    // 自动调整页面布局
themeApply2g theme:="ShLiu_Frame"; // 应用自定义主题
label -p 10 90 -n myNote "Batch Processed"; // 添加水印/注释

doc -uw;                      // 刷新文档
type -s "--- PlotColN 执行完毕 ---";

[Label]
// 快速添加文本标签
string strMsg$ = "Batch Processed";
if ("%1" != "") strMsg$ = "%1";
label -p 10 90 -n myNote "%(strMsg$)";
myNote.fsize = 15;
doc -uw;

[SetComments]
// 功能：批量设置列注释，支持传入前缀字符串
// 用法：JL_SetComments "前缀" [起始列号]
type -s "--- SetComments 开始执行 ---";

string strPrefix$ = "";
int nStart = 2; 

// 接收第一个参数作为前缀
if ("%1" != "") strPrefix$ = "%1";
// 接收第二个参数作为起始列号
if ("%2" != "") nStart = %2;

type -s "前缀: %(strPrefix$), 起始列: $(nStart)";

for(int ii = nStart; ii <= wks.ncols; ii = ii + 2) {
    // 获取当前列第C行（Comments）的值
    double valA = [%(wcol(ii)[C]$ )]; 
    
    if (valA != NANUM) {
        double res = 100 - valA;
        // 构造最终字符串：前缀 + 计算结果 + 单位
        // 示例：如果前缀是 Sheet1，结果是 80，则显示 "Sheet1: 80% CO\-(2)"
        string strFinal$ = "%(strPrefix$) $(res)%% CO\-(2)";
        wcol(ii)[C]$ = strFinal$;
    }
}

doc -uw;
type -s "--- SetComments 执行完毕 ---";

[SetAllComments]
// 功能：将当前工作表所有列的注释设为指定字符串
// 用法：JL_SetAllComments "内容"
type -s "--- Setting All Comments ---";

string strMsg$ = "%1";

// 遍历所有列
for(int ii = 1; ii <= wks.ncols; ii++) {
    wcol(ii)[C]$ = strMsg$;
}

doc -uw;
type -s "已将所有列注释设为: %(strMsg$)";

```

```c

void jl_auto_mount()
{
    // 获取 User Files 路径下的 OGS 文件路径
    string strFile = GetOriginPath(ORIGIN_PATH_USER) + "JL_Commands.ogs";
    
    stdioFile ff;
    if( !ff.Open(strFile, file::modeRead) )
    {
        printf("--- [错误] 找不到文件: %s ---\n", strFile);
        return;
    }

    printf("--- [JL 引擎] 正在扫描命令库... ---\n");
    printf("----------------------------------\n");

    string strLine;
    int nCount = 0;
    while( ff.ReadString(strLine) )
    {
        strLine.TrimLeft();
        strLine.TrimRight();

        // 识别 [SectionName] 标签
        if( strLine.GetAt(0) == '[' && strLine.GetAt(strLine.GetLength()-1) == ']' )
        {
            string strSection = strLine.Mid(1, strLine.GetLength() - 2);
            
            // 跳过 Main 标签
            if( strSection.CompareNoCase("Main") == 0 ) continue;

            // 构造并执行定义：统一支持 3 个参数，带引号保护
            string strCommand = "JL_" + strSection;
            string strDefine;
            strDefine.Format("def %s {run.section(JL_Commands.ogs, %s, \"%%1\" \"%%2\" \"%%3\")}", strCommand, strSection);
            
            if( LT_execute(strDefine) )
            {
                // 在控制台列出找到的命令
                printf("%2d. 命令: %-15s -> 对应标签: [%s]\n", nCount + 1, strCommand, strSection);
                nCount++;
            }
        }
    }
    ff.Close();

    printf("----------------------------------\n");
    printf("--- 挂载完成，共加载 %d 个命令 ---\n", nCount);
}

// 只要这个文件在 Autoload 文件夹下，这个函数就会在启动时自动跑一遍
void __main()
{
    jl_auto_mount();
}
```



