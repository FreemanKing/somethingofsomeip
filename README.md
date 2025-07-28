# somethingofsomeip
something of someip

# 代码下载

    git clone -b 3.5.1 git@github.com:COVESA/vsomeip.git

## 剔除vsomeip的git

    cd vsomeip
    rm -rf .git .github/

## 上传到自己的GitHub

    1. 删除vsomeip v3.5.1的git相关内容
    2. 上传vsomeip v3.5.1到自己GitHub

## 安装依赖软件

## googletest
    wget https://sourceforge.net/projects/boost/files/boost/1.85.0/boost_1_85_0.tar.gz

    tar -zvxf boost_1_85_0.tar.gz
    cd boost_1_85_0

    ./bootstrap.sh --prefix=/usr/

    ./b2

    sudo ./b2 installboost

## boost

### 法1

    sudo apt install libboost-dev libboost-filesystem-dev libboost-python-dev libboost-test-dev libboost-thread-dev

### 法2

    wget https://sourceforge.net/projects/boost/files/boost/1.85.0/boost_1_85_0.tar.gz

    tar -zvxf boost_1_85_0.tar.gz
    cd boost_1_85_0

    ./bootstrap.sh --prefix=/usr/

    ./b2

    sudo ./b2 installdlt

## dlt
    sudo apt-get install dlt-viewer

    sudo apt install zlib1g-dev

    git clone https://github.com/GENIVI/dlt-daemon.git
    cd dlt-daemon
    mkdir build
    cd build
    cmake ..
    make
    sudo make install

    如果sudo apt-get install dlt-viewer无法安装
    那么运行下面的
    sudo add-apt-repository ppa:costamagnagianfranco/locutusofborg-ppa && sudo apt-get update

## doxygen
    sudo apt-get install doxygen graphviz


# 编译
## 编译vsomeip
    cd vsomeip
    mkdir build
    cd build
    cmake -DGTEST_ROOT=/usr/src/googletest ..
    make -j2
    sudo make install # 这里是安装到系统里，demo可以直接调用
## 编译vsomeip-test
    cd vsomeip
    cd build
    cd test
    cmake ..
    make -j2