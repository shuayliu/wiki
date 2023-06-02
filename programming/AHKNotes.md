#  使用Autohotkey作为批处理脚本



## 批处理Kissa软件模拟

本脚本主要以快捷键方式进行操作

```ini
#SingleInstance Force
;定义文本内容

oldfilecontent :="
(Join`r`n
KISSA-1D version 1.2
Mechanism file

0
2
A+e=B
B+P=Q+A
{_cMV}
0
{_cO2}
0
{_dMV}
1E-5
{_DO2}
1E-5
10000
0
0.5
0
0
{_kf}
0
0
0
308.65
0
1E-6
10
0
0.015
0
{_v}
-0.5
0.5
0.5
1
1
1
0
0
60
0
0.01
100
1000
300
1.05
40
1
)"

kf = 230000000
v = 0.1
cMV=8.52E-6
dMV=5.3E-07
cO2=2.6E-6
dO2=280E-7

;vAry := Array(0.01,0.02,0.05,0.08,0.1,0.2,0.5,0.8,1,2,5,8,10,20,50,100)
vAry := Array(0.005,0.01,0.05,0.1,0.5,1,5,10,50)
;vAry := Array(0.1,0.2)
;启动KISSA
Run "path\to\KISSA-1D.exe" 
WinGet, kissaID, ID
Sleep 3000
filePath = %A_Desktop%\KissaData
FileRemoveDir, %filePath%, 1 
FileCreateDir, %filePath%

for _,_v in vAry{
	fileContent := StrReplace(oldfilecontent,"{_v}",_v)
	fileContent := StrReplace(fileContent,"{_kf}",kf)	
	fileContent := StrReplace(fileContent,"{_cO2}",cO2)
	fileContent := StrReplace(fileContent,"{_dO2}",dO2)
	fileContent := StrReplace(fileContent,"{_cMV}",cMV)
	fileContent := StrReplace(fileContent,"{_dMV}",dMV)

	;写配置文件
	FileDelete, %filePath%\test.mk
	FileAppend, %fileContent%, %filePath%\test.mk

	Sleep 200
	WinActivate, ahk_id %kissaID%
	Send ^o
	Sleep 250
	Send %filePath%\test.mk
	Sleep 500
	Send {Enter}
	;Send {Enter}
	Send !y 

	;Sleep, 3000
	;Kissa命令：F9运行
	Sleep 200
	Send {F9}
	Sleep 5000

	;KISSA快捷键：打开File, 存储current
	Send !f
	Send v
	Sleep 200

	;系统对话框 设置文件名
	;Send !
	SendRaw %filePath%\%_v%-%kf%
	Sleep 300
	;Send !S 
	Send {Enter}

	;文件覆盖确认
	Sleep 200
	Send !y 
	Sleep 300
}
;退出KISSA
Send !f
Send x

Run, "path\to\processKissaDataToOrigin.bat"
;Sleep 30000
;Send {Space}
;Run 


```





## 调用Originlab的COM组件，运行labtalk代码处理数据

com组件调用

```ini
wkPth := "E:\data"
fileName:="E:\ahk.opju"

template := "C:\Users\PC\Documents\OriginLab\User Files\pac.ogw" 

; 获取文件列表，并以\n拼接
FileList := ""
Loop, Files, %wkPth%\*.txt
    FileList .= A_LoopFileFullPath "`n"

; 拼接labtalk代码
code := Format("batchProcess name:=""{1}"" fname:=""{2}"" id:=<none> fill:=resource append:=Result;",template,FileList)
; 获取COM组件
org := ComObjCreate("Origin.Application")
; 创建新的Project
org.NewProject()

org.Execute(code)

org.Save(fileName)
; org.Visible := true
org.Exit()

```

