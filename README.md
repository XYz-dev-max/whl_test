# whl_test
build a simple python project to a wheel package

本项目主要给出一个简易python项目构建wheel包的过程

首先给出项目的目录结构，project_root即为我们想要打包的python项目根目录，同级创建setup.py来打包

├── project_root
│   ├── __init__.py
│   ├── data
│   │   ├── __init__.py
│   │   ├── a.sh
│   │   ├── b.png
│   │   └── c.tgz
│   ├── dir1
│   │   ├── __init__.py
│   │   ├── abc.txt
│   │   ├── dirTest.py
│   │   └── idc-run
│   ├── dir2
│   │   ├── __init__.py
│   │   └── dirTest.py
│   ├── main.py
│   └── python
│       ├── __init__.py
│       ├── p1.py
│       └── pdir1
│           └── p2.py
└── setup.py

setup.py代码如下：

from setuptools import setup, find_packages
from setuptools.command.install_scripts import install_scripts

# 拷贝 非project_root项目下的其他路径下的data文件 到
# project_root项目下的任意包中

setup(
    name = 'printdemo',                              # package name
    version = '1.0',
    author = 'author',                               # author可以用来指定该package的作者信息
    author_email = 'author_email',                   # 这个也是指定该package的作者信息
    description = '',                                # 对当前package的较短的总结
    long_description = '',                           # 是对当前package的详细说明。这一详细说明将被展示在Python Package Index上当前项目的主页
    packages=find_packages(),                        # include all packages under src
    python_requires = '>=2.7.1',                     # python依赖版本
    package_data = {
        '' : ['**'],
    },                                               
    # include_package_data : True,                   # 可以使用这行属性值 + MNIFEST.in配置数据文件打包需求 替代package_data属性
    install_requires  = ['wheel==0.37.1',],          # 其他依赖包
    entry_points = {
        'console_scripts' : ['pytest = project_root.main:func']
    },
)

# 移除 非project_root项目下的其他路径下的data文件

首先，包含__init__.py的文件夹我们都称为一个包(package)，实现打包的过程都是围绕包这个概念展开的
上述目录结构中，project_root  data dir1 dir2 python都是包，而pdir1则不是包
有了这些信息，我们接下来逐步实现对此项目的打包

	1. 
.py文件的打包


    packages = find_packages()
    这行属性值告诉打包工具循环嵌套地寻找setup.py所在目录下的所有包，并将各个包下的.py文件打包
    
    在本项目中，pdir1不是一个包，所以pdir1/p2.py文件并未打包，想要实现对其打包
    需要在pdir1文件下增加__init__.py使其作为一个包出现

	1. 
数据文件的打包(.txt .sh .tgz .png等)


    package_data = {'' : ['**']} 
    这行属性值告诉打包工具循环嵌套地寻找所有包下的所有data文件
    
    在本项目中，data、dir1作为一个包，其下的所有data文件a.sh、b.png、c.tgz、abc.txt、idc-run(可执行文件)都将作为数据打包

	* 
如果想要打包project_root项目下非包文件下的data文件，则在非包文件下增加__init__.py变为包(参考本例中data包)
	* 
如果想要打包非project_root项目下的其他路径下的data文件，可以这样处理：


    方式一：
        data_files = {('data文件路径', 'data_name')} (这种方式在whl包发布安装后，data文件会作为python的共享数据，其安装路径也不会在项目包路径下)
    方式二：
        在setup.py代码的构建包主体 setup() 的前后增加预处理和后处理，预处理将data文件拷贝到项目包路径下，后处理移除这些data文件
        这样即实现了data文件打包到项目包路径下，又不会对原项目结构造成影响

	1. 
自动生成cli(command-line interface)命令


    entry_points = {
        'console_scripts' : ['pytest = project_root.main:func']
    }
    这条属性值告诉打包工具生成一个包安装后名为pytest的cli命令，这条命令将回调project_root.main下的func函数

    假设main.py为整个项目入口，如下：


import os
from pkg_resources import Requirement, resource_filename

def func():

    # 项目入口 代码

    # 通过命令行脚本的方式运行时
    # 找到的是package安装后的main.py的路径
    file_base_path = os.path.dirname(__file__)
    print(file_base_path)
    txt_file = file_base_path + '/dir1/abc.txt'
    with open(txt_file) as file:
        print(file.read())

if __name__ == '__main__':
    func()
    abc.txt内容如下：

txt file test!!!
    
    包发布安装后，自动生成的pytest命令内容如下：

#!/root/miniconda3/bin/python
# -*- coding: utf-8 -*-
import re
import sys
from project_root.main import func
if __name__ == '__main__':
    sys.argv[0] = re.sub(r'(-script\.pyw|\.exe)?$', '', sys.argv[0])
    sys.exit(func())
    本质是一个python脚本，执行pytest会回调project_root.main下的func函数
    
    在命令窗口执行pytest结果如下：

(base) root@LAPTOP-NJ5PKOQ2:/home/xinyu/workspace/whl_test/dist# pytest
/root/miniconda3/lib/python3.8/site-packages/project_root
txt file test!!!

至此，我们已经完成了对这个项目构建的setup.py文件的编写

在setup.py文件路径下执行  python setup.py bdist_wheel  来构建whl包
生成的whl包在当前目录的dist下 pip install ***.whl       安装

注意：
    在调试setup.py代码的过程中，每次执行python setup.py bdist_wheel之前，建议删除路径下的 build、dist、***.egg-info文件，防止上一次的构建结果影响本次构建

最后给出一些常用的构建参数：

name: 该package的名字，该名字可以由字母、数字、-组成，注意这个名字不能与其它已经上传到pypi.org的项目相同


version: 这个就是包的发布版本，可以直接写在这，也可以从其它地方引用


author: author可以用来指定该package的作者信息


author_email: 这个也是指定该package的作者信息


py_modules: 打包的.py文件


license: 支持的开源协议


description: 对当前package的较短的总结


long_description: 是对当前package的详细说明。这一详细说明将被展示在Python Package Index上当前项目的主页


long_description_content_type: 指定了long_description内容的格式。在当前情况下为markdown


url: 是当前package的主页链接。大多数情况下这是一个GitHub, GitLab, Bitbucket或者其他代码存储服务的链接


packages: 打包的python文件夹，是一系列应当包含在发布软件包文件(distribution package)中的可被import的python包文件。我们可以手动在此处罗列所有文件。或者如本例中一样使用find_packages()函数自动包含所有的python包文件以及子包文件。(from setuptools import setup, find_packages)


python_requires: python依懒版本


classifiers: 指定了当前package的其他元信息(metadata)。例如当前package兼容的python版本和操作系统，当前package提供的功能的类型，当前package的许可证等等。我们应当总是至少包括当前package所支持的python版本，操作系统和许可证。注意此处定义的classifiers关键字所包含的信息应当符合PyPI的规定。


requires : 定义依赖哪些模块


install_requires: 指定了当前package所依赖的其他python类库。这些指定的python类库将会在本package被安装的时候一并被安装。['','', ...]


platforms: 程序适用的软件平台列表


keywords: 程序的关键字列表


include_package_data: 是否自动包含包内所有受版本控制(cvs/svn/git)的数据文件，默认True


entry_points: 用来支持自动生成cli命令

想要了解更多参数或其他高级用法，参考以下链接
参考：setuptools 61.3.1.post20220401.post-20220401 documentation (pypa.io)
