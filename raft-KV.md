# 环境配置
## 我的环境与版本记录
由于我的电脑是cachyOS，因此我做了一些调整，其实大差不差，都是工具，当然由于muduo库的protobuf有要求，整体和目录都一致。
- 系统：CachyOS（Arch Linux 系）
- g++ ：(GCC) 16.1.1 20260625
- CMake：`3.26.4` 因为项目基于这个版本，所以我都架构在这个版本之上
- Protobuf：拉取git指定版本
- Boost：最新版
- Muduo：由本地 `muduo-master.zip` 构建，头文件位于 `/usr/local/include/muduo`，库位于 `/usr/local/lib`
# 框架梳理
