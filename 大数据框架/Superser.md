&emsp;<a href="#0">Superset</a>  
&emsp;&emsp;<a href="#1">第1章：Superset入门</a>  
&emsp;&emsp;&emsp;<a href="#2">1.1概述</a>  
&emsp;&emsp;&emsp;<a href="#3">1.2Superset应用场景</a>  
&emsp;&emsp;<a href="#4">第2章：Superset安装及使用</a>  
&emsp;&emsp;&emsp;<a href="#5">1.安装Python环境</a>  
&emsp;&emsp;&emsp;&emsp;<a href="#6">1.1安装Miniconda</a>  
&emsp;&emsp;&emsp;&emsp;<a href="#7">1.2创建Python3.7环境</a>  
&emsp;&emsp;&emsp;&emsp;<a href="#8">1.3常用命令</a>  
&emsp;&emsp;&emsp;<a href="#9">2.Superset部署</a>  
&emsp;&emsp;&emsp;&emsp;<a href="#10">2.1安装依赖</a>  
&emsp;&emsp;&emsp;&emsp;<a href="#11">2.2安装Superset</a>  
&emsp;&emsp;&emsp;&emsp;<a href="#12">2.3启动Supterset</a>  
&emsp;&emsp;&emsp;&emsp;<a href="#13">2.4Superset.sh：启停脚本</a>  
&emsp;&emsp;<a href="#14">第3章：Superset使用</a>  
&emsp;&emsp;&emsp;<a href="#15">1.安装依赖</a>  
&emsp;&emsp;&emsp;<a href="#16">2.重启Superset</a>  
## <a name="0">Superset




### <a name="1">第1章：Superset入门


#### <a name="2">1.1概述


​		Apache Superset是一个开源的、现代的、轻量级BI分析工具，能够对接多种数据源、拥有丰富的图表展示形式、支持自定义仪表盘，且拥有友好的用户界面，十分易用。

#### <a name="3">1.2Superset应用场景


​		由于Superset能够对接常用的大数据分析工具，如Hive、Kylin、Druid等，且支持自定义仪表盘，故可作为数仓的可视化工具
![img](https://yingziimage.oss-cn-beijing.aliyuncs.com/img/202302151608683.png)

### <a name="4">第2章：Superset安装及使用


Superset官网地址：http://superset.apache.org/

#### <a name="5">1.安装Python环境


​		Superset是由Python语音编写的Web应用，要求Python3.7的环境

##### <a name="6">1.1安装Miniconda


​		conda是一个开源的包、环境管理器，可以用于在同一个机器上安装不同Python版本的软件包及其依赖，并能够在不同的Python环境之间切换，Anaconda有丰富的安装包，Miniconda有Conda、Python,我们不需要更多的工具包，故选择MiniConda

1）下载Miniconda（Pythn3版本）

https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh

2）安装Miniconda（确定联网状态）

```
bash https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh
```

当出现提示时，可以指定安装路径到/opt/module/miniconda3

3）加载环境变量配置文件，使之生效

```
source ~/.bashrc
```

4）取消激活base环境

Miniconda安装完成后，每次打开终端都会激活其默认的base环境，禁止激活默认base环境

```
conda config --set auto_activate_base false
```

##### <a name="7">1.2创建Python3.7环境


1）配置conda国内镜像

```
conda config --add channels https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/free

conda config --add channels https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/main

conda config --set show_channel_urls yes
```

2）创建Python3.7环境

```
conda create --name superset python=3.7
```

3）激活superset环境

```
conda activate superset
```

4）退出当前环境

```
conda deactivate
```

##### <a name="8">1.3常用命令


创建环境：conda create -n env_name

查看所有环境：conda info --envs

删除一个环境：conda remove -n env_name --all

#### <a name="9">2.Superset部署


以下都在python3.7（superset）版本中进行

##### <a name="10">2.1安装依赖


```
sudo yum install -y gcc gcc-c++ libffi-devel python-devel python-pip python-wheel python-setuptools openssl-devel cyrus-sasl-devel openldap-devel
```

##### <a name="11">2.2安装Superset


1）安装（更新）setuptools和pip

```
pip install --upgrade setuptools pip -i https://pypi.douban.com/simple/
```

注：pip是python的包管理工具，可以和centos中的yum类比

2）安装Supetest

```
pip install apache-superset -i https://pypi.douban.com/simple/
```

注明：-i的作用是指定镜像，如果网络错误导致不能下载，可更换镜像

```
pip install apache-superset --trusted-host https://repo.huaweicloud.com -i https://repo.huaweicloud.com/repository/pypi/simple
```

3）初始化Supetest数据库

```
superset db upgrade
```

4）创建管理员用户

```
export FLASK_APP=superset

superset fab create-admin
```

注：flask是一个python web框架，Superset使用的就是flask

5）Superset初始化

```
superset init
```

##### <a name="12">2.3启动Supterset


1）安装gunicorn

```
pip install gunicorn -i https://pypi.douban.com/simple/
```

注：gunicorn是一个Python Web Server 类比java中的TomCat

2）启动Superset

```
gunicorn --workers 5 --timeout 120 --bind hadoop102:8787  "superset.app:create_app()" --daemon
```

- workers：指定进程个数
- timeout：worker进程超时时间，超时会自动重启
- bind：绑定本机地址，即为Superset访问地址
- daemon:后台运行

此时可访问：http://hadoop102:8787，页面登录的账号密码为创建管理员用户时声明的
![](https://yingziimage.oss-cn-beijing.aliyuncs.com/img/202302151608286.png)
3)停止superset

停掉gunicorn进程

```
ps -ef | awk '/superset/ && !/awk/{print $2}' | xargs kill -9
```

退出superset环境

```
conda deactivate
```

##### <a name="13">2.4Superset.sh：启停脚本


```sh
#!/bin/bash

superset_status(){
    result=`ps -ef | awk '/gunicorn/ && !/awk/{print $2}' | wc -l`
    if [[ $result -eq 0 ]]; then
        return 0
    else
        return 1
    fi
}
superset_start(){
        source ~/.bashrc
        superset_status >/dev/null 2>&1
        if [[ $? -eq 0 ]]; then
            conda activate superset ; gunicorn --workers 5 --timeout 120 --bind hadoop102:8787 --daemon 'superset.app:create_app()'
        else
            echo "superset正在运行"
        fi

}

superset_stop(){
    superset_status >/dev/null 2>&1
    if [[ $? -eq 0 ]]; then
        echo "superset未在运行"
    else
        ps -ef | awk '/gunicorn/ && !/awk/{print $2}' | xargs kill -9
    fi
}

case $1 in
    start )
        echo "启动Superset"
        superset_start
    ;;
    stop )
        echo "停止Superset"
        superset_stop
    ;;
    restart )
        echo "重启Superset"
        superset_stop
        superset_start
    ;;
    status )
        superset_status >/dev/null 2>&1
        if [[ $? -eq 0 ]]; then
            echo "superset未在运行"
        else
            echo "superset正在运行"
        fi
esac
```

### <a name="14">第3章：Superset使用


#### <a name="15">1.安装依赖


对接不同的数据源，需安装不同的依赖

官网说明：[**https://superset.apache.org/docs/databases/installing-database-drivers**](https://superset.apache.org/docs/databases/installing-database-drivers)

```
conda install mysqlclient
```

此为mysql依赖

#### <a name="16">2.重启Superset


```
superset.sh restart
```
