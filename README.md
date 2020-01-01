# YiControl Tools

LVGL 资源工具集，用于将 `assets` 目录中的 LVGL 字体（.h）和图片（.c）文件打包成二进制文件，并刷入 ESP32 的 `res` 分区。

## 下载

前往 [Releases](https://github.com/Zhang-abab/tools/releases) 下载对应平台的压缩包：

| 平台 | 文件 |
|------|------|
| macOS (Apple Silicon) | `yicontrol-tools-darwin-arm64.tar.gz` |
| Linux (x64) | `yicontrol-tools-linux-x64.tar.gz` |
| Windows (x64) | `yicontrol-tools-windows-x64.zip` |

解压后包含以下可执行文件：

| 工具 | 说明 |
|------|------|
| `pack_resources` | 资源打包工具 |
| `resource_manager` | 资源管理与验证工具 |
| `verify_res_partition` | ESP32 资源分区验证工具 |

> Windows 下可执行文件带 `.exe` 后缀。

## 快速开始

### 打包资源

```bash
# 基本用法（扫描 ../assets，输出到 ../build/resources.bin）
./pack_resources

# 指定输入和输出路径
./pack_resources -i ../assets -o ../build/resources.bin

# 显示详细信息
./pack_resources -v
```

#### 命令行参数

| 参数 | 简写 | 默认值 | 说明 |
|------|------|--------|------|
| `--assert-dir` | `-i` | `../assets` | 资源目录路径 |
| `--output` | `-o` | `../build/resources.bin` | 输出bin文件路径 |
| `--verbose` | `-v` | — | 显示详细信息 |

### 资源管理

```bash
# 查看资源文件信息
./resource_manager resources.bin info

# 列出所有资源
./resource_manager resources.bin list
./resource_manager resources.bin list -d   # 显示详细信息

# 验证资源文件完整性
./resource_manager resources.bin verify

# 提取指定资源
./resource_manager resources.bin extract <资源ID> <输出文件>

# 搜索资源
./resource_manager resources.bin search <关键字>
```

### 验证 ESP32 分区

```bash
# 使用默认端口 (/dev/tty.usbmodem2401)
./verify_res_partition

# 指定端口
./verify_res_partition /dev/ttyUSB0
```

### ESP32 刷写

```bash
# 刷写资源分区
idf.py partition-table-flash --partition-name res --partition-file build/resources.bin

# 指定端口
idf.py -p /dev/ttyUSB0 partition-table-flash --partition-name res --partition-file build/resources.bin
```

## 输出文件

打包后生成三个文件：

| 文件 | 说明 |
|------|------|
| `resources.bin` | 二进制资源文件，用于刷入 ESP32 |
| `resources.json` | 资源索引文件，用于调试和管理 |
| `resources.h` | C头文件，包含资源ID枚举和API声明 |

## 支持的颜色格式

| 格式名称 | 枚举值 | 描述 |
|---------|--------|------|
| LV_COLOR_FORMAT_RGB565A8 | 0x14 | RGB565 + 8位Alpha通道 |
| LV_COLOR_FORMAT_RGB565 | 0x12 | 16位RGB565格式 |
| LV_COLOR_FORMAT_RGB888 | 0x0F | 24位RGB888格式 |
| LV_COLOR_FORMAT_ARGB8888 | 0x10 | 32位ARGB8888格式 |
| LV_COLOR_FORMAT_I8 | 0x0A | 8位索引色 |
| LV_COLOR_FORMAT_A8 | 0x0E | 8位Alpha通道 |

## 二进制文件结构

```
Header (32 bytes):
├── Magic: "LVGL" (4 bytes)
├── Version: 3 (4 bytes)
├── Resource count (4 bytes)
├── Total size (4 bytes)
├── Header checksum (4 bytes)
└── Reserved (12 bytes)

Resource entries (变长):
字体条目 (16 bytes):
├── Type: 1 (4 bytes)
├── Offset (4 bytes)
├── Size (4 bytes)
└── Checksum (4 bytes)

图片条目 (32 bytes):
├── Type: 2 (4 bytes)
├── Offset (4 bytes)
├── Size (4 bytes)
├── Checksum (4 bytes)
├── Width (4 bytes)
├── Height (4 bytes)
├── Color format (4 bytes)
└── Magic: 0x19 (4 bytes)

Data section:
└── Resource data (variable size)
```

## 资源文件格式

### 字体文件 (.h)

```c
const uint8_t source_han_sans_medium[]= {
    0x00, 0x01, 0x02, ...
};
```

### 图片文件 (.c)

```c
const LV_ATTRIBUTE_MEM_ALIGN LV_ATTRIBUTE_LARGE_CONST LV_ATTRIBUTE_IMAGE_ICO_1 uint8_t ico_1_map[] = {
    0xff, 0xff, 0xff, ...
};

const lv_image_dsc_t ico_1 = {
  .header.cf = LV_COLOR_FORMAT_RGB565A8,
  .header.magic = LV_IMAGE_HEADER_MAGIC,
  .header.w = 64,
  .header.h = 64,
  .data_size = 4096 * 3,
  .data = ico_1_map,
};
```

## 分区配置

确保 `partitions.csv` 文件中包含 `res` 分区：

```csv
# Name,     Type, SubType, Offset,   Size,      Flags
res,        0x40,  0x00,    0x420000, 0x9D0000,
```

## 开发集成

### C代码集成

```c
#include "resources.h"

extern lvgl_resource_manager_t g_resource_manager;

lv_image_dsc_t* load_image_by_id(lvgl_resource_id_t id) {
    // 实现资源加载逻辑
}

bool check_resource_version() {
    return g_resource_manager.version == LVGL_RESOURCES_FORMAT_VERSION;
}
```

### CMake 集成

```cmake
target_include_directories(${COMPONENT_LIB} PRIVATE
    ${CMAKE_BINARY_DIR}
)

add_custom_target(pack_resources
    COMMAND ${CMAKE_SOURCE_DIR}/tools/pack_resources
    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/tools
    COMMENT "Packing LVGL resources"
)
```

## 错误处理

### 常见错误

1. **数据大小不匹配**
   ```
   ❌ ico_1_map 数据大小不匹配 - 期望: 12288, 实际: 12000
   ```
   检查图片文件中的 `data_size` 字段是否正确

2. **未找到数组定义**
   ```
   警告: 在 file.c 中未找到图片数据数组
   ```
   检查文件格式，确保包含正确的C数组定义

3. **无法解析颜色格式**
   ```
   警告: 无法解析数据大小表达式 'UNKNOWN_EXPR'
   ```
   检查图片描述符中的表达式格式

### 调试技巧

- 使用 `-v` 参数查看详细处理过程
- 检查生成的 `.json` 文件了解资源索引信息
- 使用 `resource_manager` 验证生成的bin文件
