# 向 gitlab 上传文件
``` bash
1、项目的浏览器地址：
    http://gitlab.web.site:0000/huawei/aios/-/tree/main?ref_type=heads
2、使用HTTP克隆：
    http://gitlab.miaohan.xyz/huawei/aios.git
```
第一步：在项目网页中，手动"创建新分支"，命名为"aios_merged_ver_0304".
第二步：在电脑本地，clone 原仓库
``` bash
C:\Users\Administrator\Desktop>git clone http://gitlab.web.site:0000/huawei/aios.git
 (clone后，原仓库将位于C:\Users\Administrator\Desktop\aios)
```
第三步：确认当前所在的分支
``` bash
git branch
```
第四步：切换到刚才新建的分支
``` bash
git checkout aios_merged_ver_0304
```
第五步：复制要上传的 "soul_0304" 项目文件
``` bash
xcopy C:\Users\Administrator\Desktop\soul_0304\* C:\Users\Administrator\Desktop\aios\ /E /H /C /I /Y
```
第六步：上传
``` bash
git status
git add .
git commit -m "Your_Command_Is_What"
git push origin HEAD:aios_merged_ver_0304
```
