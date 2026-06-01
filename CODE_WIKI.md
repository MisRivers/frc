# 胖鼠采集（Fat Rat Collect）项目文档

## 项目概述

胖鼠采集（Fat Rat Collect）是一款面向 WordPress 的自动化内容采集插件，适用于资讯站、内容聚合站及需要结构化采集网页内容的业务场景。插件支持通过可配置规则完成内容提取、链接补全、图片处理、数据清洗、自动采集与自动发布。

### 核心功能
- 微信公众号文章采集
- 微信公众号历史文章采集
- 登录态页面采集（支持 Cookie）
- 简书文章采集
- 知乎问答采集
- 列表采集与历史采集
- 详情页采集
- 分页采集
- 自动采集（定时任务）
- 自动发布到 WordPress
- 调试模式
- 内容增强（动态内容、自动标签、标签内链等）
- 内容去重
- 特色图片处理
- 图片本地化（兼容对象存储插件）
- 数据处理（基于 HTML 与 jQuery 的内容过滤、替换及规则化处理）
- 自定义站点采集
- 相对链接补全
- 图片链接类型处理

---

## 目录结构

```
fatratcollect/
├── fatratcollect.php       # 主插件文件，插件入口
├── composer.json           # Composer 依赖配置
├── composer.lock
├── README.md
├── LICENSE
├── readme.txt
├── includes/               # 核心功能模块
│   ├── fatrat-apierror.php
│   ├── fatrat-data.php
│   ├── fatrat-data-detail.php
│   ├── fatrat-debugging.php
│   ├── fatrat-kit.php
│   ├── fatrat-options.php
│   ├── fatrat-options-add-edit.php
│   ├── fatrat-spider.php
│   └── fatrat-validation.php
├── src/                    # 源代码目录
│   ├── Controller/
│   │   └── TaskController.php
│   ├── Helpers/
│   │   ├── helpers.php
│   │   └── helpers2.php
│   └── Service/
│       ├── AbsoluteUrl.php
│       ├── DownloadImage.php
│       └── GetTransCoding.php
├── v3/                     # V3 版本（Vue3 + REST API）
│   ├── app/
│   │   ├── src/
│   │   │   ├── api/
│   │   │   ├── components/
│   │   │   ├── views/
│   │   │   ├── App.vue
│   │   │   ├── main.js
│   │   │   └── i18n.js
│   │   ├── package.json
│   │   └── vite.config.js
│   ├── dist/               # V3 构建文件
│   ├── includes/
│   │   ├── class-frc-v3-menu.php
│   │   └── class-frc-v3-rest.php
│   └── autoload.php
├── public/                 # 静态资源
│   ├── css/
│   └── js/
├── views/                  # 视图模板
│   ├── csrf.php
│   ├── frc-spider.php
│   ├── release-type.php
│   └── todo.html
└── vendor/                 # 第三方依赖
```

---

## 核心架构

### 插件入口文件 (fatratcollect.php)

**文件路径**：`/workspace/fatratcollect.php`

这是插件的主入口文件，主要功能包括：
- 注册插件激活/卸载钩子
- 创建/升级数据库表
- 加载 Composer 依赖
- 注册管理后台菜单
- 加载前端资源（CSS/JS）
- 注册 AJAX 接口
- 设置定时任务（Cron）
- 加载 V3 版本（Vue3 + REST API）

**核心功能函数**：
- `frc_plugin_install()`: 插件激活时执行，创建数据库表
- `frc_plugin_update()`: 检查并执行数据库升级
- `frc_loading_assets()`: 加载管理后台的 CSS 和 JS
- `frc_loading_menu()`: 注册管理后台菜单
- `frc_spider_timing_task()`: 定时采集任务
- `frc_cron_release_task()`: 定时发布任务

### 数据库表结构

#### `wp_frc_options` 表
存储采集配置信息

| 字段名 | 类型 | 说明 |
|--------|------|------|
| id | int | 主键 |
| collect_name | varchar(30) | 配置名称 |
| collect_describe | varchar(200) | 配置描述 |
| collect_type | varchar(20) | 采集类型（list/single/all/keyword） |
| collect_list_url | varchar(191) | 列表采集地址 |
| collect_list_url_paging | varchar(191) | 分页采集地址（含 {page} 占位符） |
| collect_list_range | varchar(191) | 列表采集范围（CSS 选择器） |
| collect_list_rules | varchar(1000) | 列表采集规则 |
| collect_content_range | varchar(191) | 详情采集范围 |
| collect_content_rules | varchar(1000) | 详情采集规则 |
| collect_charset | varchar(20) | 页面编码（默认 utf-8） |
| collect_cookie | varchar(5000) | Cookie 信息 |
| collect_image_download | tinyint | 图片下载方式（1:下载到本地 2:不下载 3:清除所有图片） |
| collect_image_path | tinyint | 图片路径类型（1:绝对路径 2:相对路径） |
| collect_image_attribute | varchar(20) | 目标图片源属性（src/data-src 等） |
| collect_rendering | tinyint | 渲染方式（1:普通 HTTP 2:浏览器渲染） |
| collect_remove_head | tinyint | 头处理方式 |
| collect_auto_collect | tinyint | 自动采集开关 |
| collect_auto_release | tinyint | 自动发布开关 |
| collect_release | varchar(191) | 发布配置（JSON） |
| collect_keywords_replace_rule | mediumtext | 关键词替换规则 |
| collect_custom_content | mediumtext | 自定义内容（头部/尾部，JSON） |
| collect_keywords | text | 关键词配置（JSON） |
| created_at | timestamp | 创建时间 |
| updated_at | timestamp | 更新时间 |

#### `wp_frc_post` 表
存储采集到的内容

| 字段名 | 类型 | 说明 |
|--------|------|------|
| id | int | 主键 |
| option_id | int | 关联的配置 ID |
| status | tinyint | 状态（1:待处理 2:待发布 3:已发布 5:失败） |
| title | varchar(120) | 文章标题 |
| cover | varchar(191) | 封面图片 |
| content | mediumtext | 文章内容 |
| link | varchar(191) | 原文链接（唯一索引） |
| post_id | int | 发布后的 WordPress 文章 ID |
| message | varchar(191) | 状态消息 |
| created_at | timestamp | 创建时间 |
| updated_at | timestamp | 更新时间 |

---

## 主要模块说明

### 1. 采集模块 (FRC_Spider)

**文件路径**：`/workspace/includes/fatrat-spider.php`

这是核心采集类，负责所有采集任务的执行。

**主要方法**：
- `grab_custom_page()`: 懒人采集（微信/简书/知乎预设规则）
- `grab_details_page()`: 详情页采集
- `grab_list_page()`: 列表页采集
- `grab_history_page()`: 分页历史采集
- `grab_all_page()`: 全站采集
- `grab_debug()`: 调试模式采集
- `grab_wechat_history()`: 微信公众号历史文章采集
- `single_spider()`: 单个页面采集
- `timing_spider()`: 定时采集任务
- `_QlObject()`: 构建 QueryList 对象
- `_QlPagingObject()`: 构建分页 QueryList 对象
- `_QlInstance()`: 获取 QueryList 实例
- `insert_article()`: 插入文章到数据库
- `checkPostLink()`: 检查文章链接是否已存在（去重）
- `rulesFormat()`: 格式化采集规则

### 2. 配置管理模块 (FRC_Options)

**文件路径**：`/workspace/includes/fatrat-options.php`

负责采集配置的 CRUD 操作。

**主要方法**：
- `options_paging()`: 分页获取配置列表
- `options()`: 获取所有配置
- `option()`: 获取单个配置
- `delete()`: 删除配置
- `lazy_person()`: 获取懒人预设配置（微信/简书/知乎）
- `insert_option()`: 插入预设配置
- `interface_save_option()`: 保存配置接口
- `interface_save_option_release()`: 保存发布配置
- `interface_import_default_configuration()`: 导入示例配置
- `interface_del_option()`: 删除配置接口
- `interface_upgrade()`: 数据库升级接口
- `interface_update_auto_config()`: 更新自动采集配置

### 3. 数据管理模块 (FRC_Data)

**文件路径**：`/workspace/includes/fatrat-data.php`

负责采集到的文章数据的管理和发布。

### 4. 辅助工具模块 (FRC_Kit)

**文件路径**：`/workspace/includes/fatrat-kit.php`

提供各种辅助工具功能。

### 5. 调试模块 (FRC_Debugging)

**文件路径**：`/workspace/includes/fatrat-debugging.php`

提供采集规则调试功能。

### 6. 验证模块 (FRC_Validation)

**文件路径**：`/workspace/includes/fatrat-validation.php`

处理权限验证、功能开关等。

---

## 关键类与函数

### 采集规则语法

胖鼠采集使用自定义的规则语法来配置采集：

**规则格式**：
```
key%selector|attribute|filter
```

**多个规则组合**：
```
key1%selector1|attribute1|filter1)(key2%selector2|attribute2|filter2
```

**示例**：
```
title%h1|text|null)(content%#content|html|null)(link%a|href|null
```

| 部分 | 说明 |
|------|------|
| key | 数据字段名 |
| selector | CSS 选择器 |
| attribute | 要获取的属性（text/html/href/src 等），null 表示文本内容 |
| filter | 过滤规则，可指定要保留或删除的元素 |

### Service 层类

#### AbsoluteUrl
**文件路径**：`/workspace/src/Service/AbsoluteUrl.php`

处理相对 URL 转绝对 URL。

#### DownloadImage
**文件路径**：`/workspace/src/Service/DownloadImage.php`

处理图片下载到本地 WordPress 媒体库。

#### GetTransCoding
**文件路径**：`/workspace/src/Service/GetTransCoding.php`

处理页面编码转换。

---

## 依赖关系

### Composer 依赖 (composer.json)

```json
{
  "require": {
    "jaeger/querylist": "^4.2",
    "jaeger/querylist-puppeteer": "^4.0",
    "tightenco/collect": "8.78.0",
    "ext-json": "*"
  },
  "require-dev": {
    "symfony/var-dumper": "^5.0"
  }
}
```

| 依赖包 | 说明 |
|--------|------|
| jaeger/querylist | 基于 phpQuery 的 PHP 采集工具 |
| jaeger/querylist-puppeteer | QueryList 的 Puppeteer 扩展，支持浏览器渲染 |
| tightenco/collect | Laravel Collections 的独立包，提供强大的集合操作 |
| symfony/var-dumper | 调试工具（仅开发环境） |

---

## 项目运行方式

### 1. 环境要求

- PHP 7.1 或更高版本
- WordPress 5.0 或更高版本
- MySQL 5.7 或更高版本

### 2. 安装步骤

1. 在 WordPress 后台插件市场搜索「胖鼠采集」并安装，或手动将插件文件夹上传到 `/wp-content/plugins/`
2. 激活插件
3. 访问插件管理页面开始使用

### 3. 基本使用流程

1. 在「配置中心」创建采集配置，设置采集规则
2. 在「采集中心」执行采集任务
3. 在「数据桶」查看和管理采集到的内容
4. 发布内容到 WordPress

---

## V3 版本架构

V3 版本使用现代化技术栈重构了用户界面：

- **前端**：Vue 3 + Vite + Bootstrap
- **后端**：WordPress REST API
- **国际化**：支持中英文双语

**文件结构**：
- `/workspace/v3/app/src/`: 前端源代码
- `/workspace/v3/includes/class-frc-v3-rest.php`: REST API 控制器
- `/workspace/v3/includes/class-frc-v3-menu.php`: 菜单注册
- `/workspace/v3/dist/`: 前端构建文件

---

## 定时任务 (Cron)

插件使用 WordPress Cron 系统实现自动采集和发布：

| 任务 | 钩子名 | 说明 |
|------|--------|------|
| 自动采集 | `frc_cron_spider_hook` | 定期执行采集任务 |
| 自动发布 | `frc_cron_release_hook` | 定期发布采集到的内容 |

**可用的 Cron 间隔**：
- `fifteenminutes`: 每 15 分钟
- `halfhour`: 每 30 分钟
- `twohourly`: 每 2 小时
- `threehours`: 每 3 小时
- `fourhourly`: 每 4 小时
- `eighthourly`: 每 8 小时

---

## 常用函数参考

### 辅助函数 (Helpers)

**文件路径**：`/workspace/src/Helpers/helpers.php`

| 函数 | 说明 |
|------|------|
| `startsWith($haystack, $needles)` | 检查字符串是否以指定子串开头 |
| `frc_sanitize_text($key, $default)` | 安全获取并净化 $_REQUEST 中的文本数据 |
| `frc_sanitize_textarea($key, $default)` | 安全获取并净化 $_REQUEST 中的文本域数据 |
| `frc_sanitize_array($key, $type)` | 安全获取并净化 $_REQUEST 中的数组数据 |
| `frc_option_esc_attr_e($option, $key, $default)` | 安全输出配置项属性值 |
| `translationRules($rules)` | 解析采集规则为数组 |
| `frcAddColumn($column, $alterColumnSql, $table)` | 安全添加数据库列 |
| `frcChangeColumn($sql, $table)` | 安全修改数据库列 |
| `getDb()` | 获取 $wpdb 实例 |
| `getTable($table)` | 获取表名（带前缀） |

---

## 开发指南

### 添加新的采集规则预设

在 `FRC_Options::insert_option()` 方法中添加新的预设配置。

### 扩展采集功能

1. 在 `FRC_Spider` 类中添加新的方法
2. 在 `fatratcollect.php` 的 AJAX 接口中注册新接口
3. 在前端添加对应的 UI

### 自定义数据处理

在采集后、发布前使用 `collect_custom_content` 配置添加自定义头部和尾部内容，支持变量替换：
- `{link}`: 原文链接
- `{title}`: 文章标题
- `{title+link}`: 带链接的标题
- `{author}`: 作者
- `{name}`: 名称

---

## 注意事项

1. 请在合法、合规、获得授权的前提下使用本插件
2. 采集任务可能消耗较多系统资源，特别是图片下载场景
3. 首次使用建议先导入演示示例
4. 注意目标网站的 robots.txt 规则和反爬机制
5. 建议使用 PHP 7.2+ 以获得最佳性能

---

## 常见问题

**Q: 如何采集需要登录的页面？**
A: 在配置中填写 Cookie 信息。

**Q: 如何采集动态加载的内容？**
A: 将渲染方式设置为「浏览器渲染」。

**Q: 如何处理不同编码的页面？**
A: 在配置中设置正确的页面编码，或选择自动转码选项。

**Q: 采集到的图片无法显示？**
A: 检查图片属性配置是否正确（src/data-src 等）。
