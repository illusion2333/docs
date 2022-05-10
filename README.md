[toc]





# readthecods docs MD编写规范

本文档主要描述MD文档在sphinx环境下需要注意的事项

地址： <a href="https://docs.chainmaker.org.cn" target="_blank">docs.chainmaker.org.cn</a>



## 文档

**文档目录**：工蜂平台：[ChainMaker/docs/readthedocs/docs](https://git.code.tencent.com/ChainMaker/docs/tree/readthedocs/readthedocs/docs)

**图片目录**：所有图片放在[docs/readthedocs/docs/images](https://git.code.tencent.com/ChainMaker/docs/tree/readthedocs/readthedocs/docs/images) 文件夹下

预览： 推送到工蜂后，等待10秒左右可在线预览<br>预览地址：http://82.157.124.172/${branchName}/html/ 如：http://82.157.124.172/v1.2.3_spv/html/

## 编写格式规范

1. **`新文件`需要添加到索引`index.rst`中**

2.  **若文档在index.rst中则，标题无须手动添加序号**

   直接在总索引文件[docs/readthedocs/docs/index.rst](https://git.code.tencent.com/ChainMaker/docs/blob/readthedocs/readthedocs/docs/index.rst)中加入` :numbered: `即可自动生成索引序号
   
   索引文件中可直接写文件名不带.md/.rst等后缀

   若文档不在index.rst中，则标题好需要手动添加
   

样例：

```rst
.. toctree::
    :maxdepth: 2
    :caption: 快速入门
    :numbered:

    tutorial/quick_start
```

3. **每个MD文档`有且仅有`一个一级标题**

4. **图片要用相对路径（必须用`英文`路径和命名）**

   样例如下：

```markdown
<img loading="lazy" src="../images/ManagementAccount.png" style="zoom:100%;" />
```

5. **图的源文件放在/readthedocs/drawio文件夹下**
6. 新添加的图片需要提交UI由UI重新做图
7. 可支持标准html标签，但标准html标签内写入markdown语法可能不会被识别。
8. 为了确保图片性能达到最好
   1. 图片格式优先考虑`png`
   2. 图片尺寸分辨率不高于`1100px`，推荐`1024px`
   3. img标签增加`loading="lazy"`



## 新分支

1. 需修改`readthedocs\docs\conf.py`中72行左右的version和release为对应分支
2. 需修改`readthedocs\docs\intro\版本迭代.md`中的新版本的功能及链接，下方docker镜像地址
3. 需修改`readthedocs\docs\operation\智能合约.md`中合约开发镜像地址（如有修改）
4. 需修改全文档版本号



## 数学公式

**数学公式仅支持rst文档格式。**
超链接到该文档需去除后缀：如： 超链接到`math.rst `文档需写成

```
[math](./math) 
```

若为md文档则可在此网站转换为rst：
	https://cloudconvert.com/md-to-rst