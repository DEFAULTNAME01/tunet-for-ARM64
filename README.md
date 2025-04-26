本分支针对对瑞芯微或者其他arm64，兼容openwrt，在wls上进行交叉编译，完成后使用scp上传到嵌入式linux系统，并设置开机启动服务
编译 aarch64-unknown-linux-musl 平台 tunet 命令行工具全过程
1.安装 Rust 环境
在开发环境（x86-64 Linux/WSL）中：
```bash
curl https://sh.rustup.rs -sSf | sh
source ~/.cargo/env
```
安装完成后，添加交叉编译目标：
```bash
rustup target add aarch64-unknown-linux-musl
```
2.安装 aarch64-musl 交叉编译工具链
下载并解压 musl 交叉编译器：
```
wget https://musl.cc/aarch64-linux-musl-cross.tgz
tar xf aarch64-linux-musl-cross.tgz
sudo mv aarch64-linux-musl-cross /opt/
```
确认工具链位置：
```
ls /opt/aarch64-linux-musl-cross/bin
```
里面有：
```
aarch64-linux-musl-gcc
aarch64-linux-musl-strip
```
...
3.手动编译 OpenSSL (可选步骤，如果需要 openssl-sys 支持)
如果项目需要 OpenSSL 静态链接，则需要编译 OpenSSL：
```
git clone https://github.com/openssl/openssl.git
cd openssl
export CC=/opt/aarch64-linux-musl-cross/bin/aarch64-linux-musl-gcc
./Configure linux-aarch64 no-shared no-async no-dso no-tests --prefix=/opt/openssl-aarch64
make -j$(nproc)
sudo make install_sw
```
设置环境变量：
```
export OPENSSL_DIR=/opt/openssl-aarch64
export OPENSSL_STATIC=1
```
4.配置 Cargo 交叉编译（可选）
如果想让 cargo 自动识别交叉编译器，可以添加 .cargo/config.toml：
```
mkdir -p .cargo
vi .cargo/config.toml
```
内容如下：
```
[target.aarch64-unknown-linux-musl]
linker = "/opt/aarch64-linux-musl-cross/bin/aarch64-linux-musl-gcc"
```
5.编译 tunet 命令行版本
进入 tunet-rust-master 源码根目录后：

直接只编译 tunet 子项目（不编 tunet-cui）：
```
cargo build --release --package tunet --target aarch64-unknown-linux-musl
```
6.Strip 优化文件体积
编译完成后，执行：
```
/opt/aarch64-linux-musl-cross/bin/aarch64-linux-musl-strip target/aarch64-unknown-linux-musl/release/tunet
```
strip 以后，文件体积大幅缩小，适合 OpenWRT 路由器空间。
7.上传 tunet 到 OpenWRT 路由器
使用 scp 上传：
```
scp target/aarch64-unknown-linux-musl/release/tunet root@你的路由器IP:/root/
```
8.设置 tunet 执行权限
在路由器上：
```
chmod +x /root/tunet
```
9.测试手动登录/注销
启用自动登录需要手动登录一次保存密码，在路由器上运行：
```
/root/tunet login
```
显示 login_ok 说明成功！
注销：
```
/root/tunet logout
```
查看状态：
```
/root/tunet status
```
10.（可选）
配置开机自启
编写 /etc/init.d/tunet 启动脚本：
```
#!/bin/sh /etc/rc.common

START=99
STOP=15

BIN_PATH="/root/tunet"
LOG_FILE="/var/log/tunet.log"
PID_FILE="/var/run/tunet.pid"

start() {
    echo "Starting tunet..."
    [ -x "$BIN_PATH" ] || {
        echo "Error: $BIN_PATH not found or not executable."
        return 1
    }
    mkdir -p /var/run
    mkdir -p /var/log
    "$BIN_PATH" login > "$LOG_FILE" 2>&1 &
    echo $! > "$PID_FILE"
}

stop() {
    echo "Stopping tunet..."
    [ -f "$PID_FILE" ] && kill "$(cat "$PID_FILE")" && rm -f "$PID_FILE"
}
```
然后赋权并启用：
```
chmod +x /etc/init.d/tunet
/etc/init.d/tunet enable
/etc/init.d/tunet start
```
这样可以通过 LuCI 管理 tunet 启动，也可以开机自动连网！


#打 .ipk 包标准流程（超简版）
1.准备目录结构
在你的 Linux 开发机上建一个临时目录，比如：
```
mkdir -p tunet-ipk/usr/bin
```
然后把编译好的 tunet 复制进去：
```
cp target/aarch64-unknown-linux-musl/release/tunet tunet-ipk/usr/bin/
chmod +x tunet-ipk/usr/bin/tunet
```
2.准备控制文件 control
创建 tunet-ipk/CONTROL 目录
```
mkdir -p tunet-ipk/CONTROL
```
在 tunet-ipk/CONTROL/ 里新建一个 control 文件：
```
vim tunet-ipk/CONTROL/control
```
写入内容示例：
```
Package: tunet
Version: 0.9.5
Depends: libc
Architecture: aarch64_generic
Maintainer: YourName <your@email.com>
Description: Tsinghua University network command line login tool.
```
注意事项：

Package: 是名字，安装后就叫 tunet

Version: 填你实际的版本

Depends: 写必要依赖（libc足够了）

Architecture: 必须是 OpenWRT 支持的，比如你的就是 aarch64_generic
（你可以通过 opkg print-architecture 确认）

Description: 简短描述就好
3.打包生成 ipk
进入 tunet-ipk/ 目录上一级：
```
cd tunet-ipk
```
然后一条命令打包：
```
tar --format=gnu -czvf ../tunet_0.9.5_aarch64_generic.ipk --owner=0 --group=0 --numeric-owner *
```
注意这里用的是 tar.gz 格式，但是扩展名改为 .ipk！

然后你会在上一层看到：
```
tunet_0.9.5_aarch64_generic.ipk
```
这就是你的最终安装包了！

安装：
```
opkg install /tmp/tunet_0.9.5_aarch64_generic.ipk
```
卸载：
```
opkg remove tunet
```
非常优雅！
给 .ipk 包加 postinst 脚本，可以做到：

安装完成后，自动执行一些命令，比如：

自动赋执行权限

自动注册 /etc/init.d/ 开机自启

自动启动 tunet login

1.创建 postinst 文件
在你打包目录 tunet-ipk/CONTROL/ 下，新建一个叫 postinst 的文件：
```
vim tunet-ipk/CONTROL/postinst
```
2.写入 postinst 内容（示范）
下面是专门为你的 tunet 工具量身定制的 postinst 示例：
```
#!/bin/sh
```
# tunet post-install script

# 确保 tunet 可执行
chmod +x /usr/bin/tunet

# 创建 init.d 启动脚本
```
cat <<'EOF' > /etc/init.d/tunet
```
```
#!/bin/sh /etc/rc.common

START=99
STOP=15

BIN_PATH="/usr/bin/tunet"
LOG_FILE="/var/log/tunet.log"
PID_FILE="/var/run/tunet.pid"

start() {
    echo "Starting tunet..."
    [ -x "\$BIN_PATH" ] || {
        echo "Error: \$BIN_PATH not found or not executable."
        return 1
    }
    mkdir -p /var/run
    mkdir -p /var/log
    "\$BIN_PATH" login > "\$LOG_FILE" 2>&1 &
    echo \$! > "\$PID_FILE"
}

stop() {
    echo "Stopping tunet..."
    [ -f "\$PID_FILE" ] && kill "\$(cat "\$PID_FILE")" && rm -f "\$PID_FILE"
}
EOF
```
# 赋予init.d脚本执行权限
```
chmod +x /etc/init.d/tunet
```
# 启用开机自启
```
/etc/init.d/tunet enable
```
# 启动一次 tunet
```
/etc/init.d/tunet start
```
exit 0
3.给 postinst 加执行权限
```
chmod +x tunet-ipk/CONTROL/postinst
```
（一定要加，不然打包后 opkg 解包执行不了！）
重新打包 .ipk
（同上面流程）重新打包：
```
cd tunet-ipk
tar --format=gnu -czvf ../tunet_0.9.5_aarch64_generic.ipk --owner=0 --group=0 --numeric-owner *
cd ..
```
得到新版本 .ipk，里面带了 postinst 脚本！
然后在 OpenWRT 上安装：
```
opkg install /tmp/tunet_0.9.5_aarch64_generic.ipk
```
安装完成后它会自动：

给 /usr/bin/tunet 加权限

自动生成 /etc/init.d/tunet

自动设置开机启动

自动立即 login！

完美！
官方资料：
https://thu.services/utils/

https://thu.services/file/RealNameAuthenticationFAQ20190614.pdf

https://thu.services/file/CampusNetworkLectureNotes201909.pdf

https://tuna.moe/blog/2022/ospp-summer-2022/


以下是引用原版内容：

# tunet-rust
清华大学校园网 Rust 库与客户端。

[![Azure DevOps builds](https://strawberry-vs.visualstudio.com/tunet-rust/_apis/build/status/Berrysoft.tunet-rust?branch=master)](https://strawberry-vs.visualstudio.com/tunet-rust/_build)

## GUI（桌面端）
基于 [Slint](https://slint-ui.com/) 开发。使用如下命令启动：

``` bash
$ tunet-gui
```

| 平台    | 亮                                   | 暗                                  |
| ------- | ------------------------------------ | ----------------------------------- |
| Windows | ![Windows](assets/windows.light.png) | ![Windows](assets/windows.dark.png) |
| Linux   | ![Linux](assets/linux.light.png)     | （暂无图片）                        |
| macOS   | ![macOS](assets/mac.light.png)       | ![macOS](assets/mac.dark.png)       |

## GUI（移动端）
基于 [Flutter](https://flutter.dev/) 开发。会尽量保证 iOS 版本能用，但是没钱发布。

| 平台    | 亮                                   | 暗                                  |
| ------- | ------------------------------------ | ----------------------------------- |
| Android | ![Android](assets/android.light.png) | ![Android](assets/android.dark.png) |

## CUI（命令行图形界面）
使用如下命令启动：

``` bash
# 使用默认（自动判断）方式登录/注销
$ tunet-cui
# 使用 auth4 方式登录/注销
$ tunet-cui -s auth4
```

![Console](assets/console.png)

## 命令行
### 登录/注销
``` bash
# 使用默认（自动判断）方式登录
$ tunet login
# 使用默认（自动判断）方式注销
$ tunet logout
# 使用 auth4 方式登录
$ tunet login -s auth4
# 使用 auth4 方式注销
$ tunet logout -s auth4
```
### 在线状态
``` bash
# 使用默认（自动判断）方式
$ tunet status
# 使用 auth4 方式
$ tunet status -s auth4
```
### 查询/强制下线在线 IP
``` bash
# 查询
$ tunet online
# IP 上线
$ tunet connect -a IP地址
# IP 下线
$ tunet drop -a IP地址
```
### 流量明细
``` bash
# 使用默认排序（注销时间，升序）查询明细
$ tunet detail
# 使用登录时间（升序）查询明细
$ tunet detail -o login
# 使用流量降序查询明细
$ tunet detail -o flux -d
# 使用流量降序查询明细，并按注销日期组合
$ tunet detail -o flux -dg
```
### Nushell 集成
`status`、`online`、`detail` 子命令支持 `--nuon` 参数，可以配合 Nushell 得到结构化的数据：
``` bash
# 在线状态表格
> tunet status --nuon | from nuon
# 查询在线 IP 表格
> tunet online --nuon | from nuon
# 明细表格
> tunet detail --nuon | from nuon
# 使用流量降序查询明细，并按注销日期组合
> tunet detail -g --nuon | from nuon | sort-by flux -r
```

### Windows 服务/macOS launchd
``` bash
# 注册服务
$ tunet-service register
# 注册服务，并定时5分钟连接一次
$ tunet-service register -i "5min"
# 注销服务
$ tunet-service unregister
```
注意 `tunet-service.exe` 自身是服务程序，如需删除应先注销服务。

### Systemd
由于不同 Linux 发行版的服务机制不同，没有提供 `register` 和 `unregister` 命令。
Debian 打包提供了 `tunet@.service` 文件。对于用户 `foo`，可以运行
``` bash
# 启用服务
$ sudo systemctl enable tunet@foo
# 启动服务
$ sudo systemctl start tunet@foo
```
可以通过编辑该文件来调整重复登录的间隔。

## 密码
用户名和密码在第一次登录时根据提示输入。请不要在不信任的电脑上保存密码。可以在桌面端图形界面点击“删除并退出”，或在命令行使用如下命令删除：
``` bash
$ tunet deletecred
```
注意：由于 Linux 的限制，目前没有找到合适的持续化密码保存方法，因此会直接明文存储。

## 网络状态
针对不同平台使用平台特定的方式尝试获得当前的网络连接方式，如果是无线网连接还会获取 SSID。
如果无法获取，则尝试连接特定的网址来判断。

<table>
  <thead>
    <tr>
      <th>平台</th>
      <th>网络状态</th>
      <th>WIFI SSID</th>
      <th>MAC 地址</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>Windows</td>
      <td colspan="2">Windows::Networking::Connectivity</td>
      <td>GetAdaptersAddresses</td>
    </tr>
    <tr>
      <td>Linux</td>
      <td>（无）</td>
      <td>Netlink</td>
      <td rowspan="4">getifaddrs</td>
    </tr>
    <tr>
      <td>Android</td>
      <td>ConnectivityManager</td>
      <td>WifiManager</td>
    </tr>
    <tr>
      <td>macOS X</td>
      <td rowspan="2">System Configuration</td>
      <td>Core WLAN</td>
    </tr>
    <tr>
      <td>iOS</td>
      <td>NEHotspotNetwork</td>
    </tr>
  </tbody>
</table>

## 编译说明
使用 `cargo` 直接编译：
``` bash
$ cargo build --release --workspace --exclude native
```
即可在 `target/release` 下找到编译好的程序。

若要为 Android 编译 APK：
``` bash
$ cd tunet-flutter
$ make apk
```
即可在 `tunet-flutter/build/app/outputs/flutter-apk/app-<架构>-release.apk` 找到打包。

## 安装说明
从 Releases 即可找到最新版分发。

### Arch Linux
有第三方打包的 AUR 和 [archlinuxcn](https://www.archlinuxcn.org/) 源可以安装。
