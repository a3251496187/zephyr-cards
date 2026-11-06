Q: NOR Flash 为什么写前必须先擦？
A: 位单向：编程 1→0，擦除 0→1；0 无法直接翻回 1，而擦除最小单位是 4 KB Sector。
---
Q: 页编程和扇区擦除的单位分别是多少？
A: 256 B / 4 KB。页编程不许跨页，由驱动自动拆；擦除的地址和长度都要 4 KB 对齐。
---
Q: 一次 4 KB 扇区擦除要多久？
A: typ 45 ms / max 400 ms。期间 CPU 轮询 BUSY，阻塞应用。
---
Q: WEL 是什么？为什么每页写前都要发 WREN？
A: Write Enable Latch。WREN 置 1，CS 上升沿自动清零，一次性锁存器，所以每次 Page Program 前都要重新 WREN。
---
Q: JEDEC ID `EF 40 16` 怎么读出来？
A: 发 0x9F，MISO 返回 3 字节：EF = Winbond，40 16 = W25Q32（32 Mbit）。
---
Q: DTS 里 `size = <0x02000000>` 为什么是 4 MB？
A: DTS 的 size 单位是 bit，32 Mbit = 4 MB，驱动内部 /8；分区 reg 的单位才是 byte。
---
Q: `write_block_size = 1` 是能按位写吗？
A: 不是。只说明 API 接受 1 字节对齐的请求；跨页由驱动拆成多条 Page Program。
---
Q: 为什么搜不到 `flash_write` 的回调实现？
A: 实现叫 `spi_nor_write()`（spi_nor.c:822），编译期由 `DEVICE_DT_INST_DEFINE(..., &spi_nor_api)` 挂到 `dev->api`，运行时只是函数指针调用。
---
Q: 驱动怎么在 4 KB 和 64 KB 擦除档之间选？
A: 遍历擦除档位表，取「地址对齐 + 剩余长度够用」中 exp 最大者（spi_nor.c:919-938）。
---
Q: prj.conf 里没写 `CONFIG_SPI`，为什么还能编译过？
A: Kconfig `select`：DTS 节点 okay → `DT_HAS_JEDEC_SPI_NOR_ENABLED` → `SPI_NOR` 自动带出 SPI/FLASH（Kconfig.nor:6-45）。
---
Q: `CONFIG_SPI_NOR_SFDP_MINIMAL` 的代价是什么？
A: 参数编译期写死（256 B 页 + 4 KB/64 KB 擦除），32 KB 档 0x52 不在驱动表里；要全档位改用 RUNTIME 或 DEVICETREE。
---
Q: Flash API 能在 ISR 里调吗？
A: 绝对不行。页写亚毫秒、擦除几十毫秒，驱动在轮询等待，会把中断延迟拖爆。
---
Q: `flash_area` 的偏移 0 指哪里？
A: 分区起点（你的测试区 = 芯片 0x3FF000）；`flash_map` 负责加 fa_off，`flash_read` 的偏移才是芯片绝对地址。
---
Q: LittleFS 和 EXT4 的根本区别？
A: EXT4 面向可随机覆写的块设备；LittleFS 专为 NOR 设计（先擦后写、掉电恢复、磨损分布）。
---
Q: OTA 镜像能放 LittleFS 里吗？
A: 不行。镜像放 MCUboot 专用分区，签名校验和启动切换由 bootloader 管。
---
Q: 10 万次擦写寿命怎么省？
A: 配置类用 NVS/Settings 做磨损均衡；别每次改参数就裸擦扇区；日志用 LittleFS 顺序写。
---
Q: 某字节现为 `0xF0`，想写成 `0x0F`，不擦除能实现吗？
A: 不能。需要把已为 0 的位翻回 1，只能擦除整个 Sector。
---
Q: `flash_erase(dev, 0x3FF001, 0x1000)` 返回什么？
A: `-EINVAL`，地址没 4 KB 对齐（spi_nor.c:896）。
---
Q: 从分区偏移 250 写 10 字节，驱动拆几圈？各多大？
A: 两圈：6 B（页尾）+ 4 B（下页）。同理 252 + 16 会拆成 4 B + 12 B。
---
Q: `.config` 里看不到 `CONFIG_SPI_NOR` 说明什么？
A: DTS 节点没 okay 或 compatible 写错，驱动没参与编译；先查生成的 DTS。
---
Q: 整片擦除为什么危险？
A: `0xC7` 一次 10~50 s，且影响全部 4 MB（含别人的分区）；学习阶段禁止。
---
Q: `flash_area_write` 和 `flash_write` 的偏移差在哪？
A: 前者相对分区起点（自动加 fa_off），后者相对芯片 0 地址。
---
Q: 驱动等待 BUSY 的两档轮询间隔是多少？
A: 写 `K_TICKS(1)`、擦 `K_MSEC(50)`（spi_nor.c:216-221）。
---
Q: `app/Kconfig` 里写 `source "src/hal/Kconfig"` 为什么找不到文件？
A: Kconfig 相对路径按 `$srctree`（= ZEPHYR_BASE = zephyr/）解析，不是相对 app/；必须写 `source "$(APPLICATION_SOURCE_DIR)/src/hal/Kconfig"`。
---
Q: OpenOCD / `west flash` 报 `Error: open failed` 最常见的原因？
A: STM32CubeProgrammer 或 `stlinkserver` 进程独占 ST-Link；关掉 CubeProgrammer 再烧。若 PnP 完全看不到设备则是物理层问题。
---
Q: 为什么不能在这个工作区直接跑 `west update`？
A: 本地 zephyr 树实际是 3.7.2，但 west.yml 声明 v4.4.0；直接 update 会升大版本，极可能破坏 out-of-tree 板定义和已验证结论。
