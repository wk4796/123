# 个人自用数据备份脚本 (Rclone)

这是一个我个人使用的 Bash 脚本，用于实现数据文件的自动化和手动备份到多种云存储（通过 Rclone），并提供详细的 Telegram 消息通知和云端备份保留策略。脚本提供了友好的命令行菜单界面，方便用户进行配置和操作。

## 目录

* [功能特性](#功能特性)

* [系统要求](#系统要求)

* [安装与设置](#安装与设置)

* [使用方法](#使用方法)

  * [主菜单导航](#主菜单导航)

  * [1. 自动备份与计划任务](#1-自动备份与计划任务)

  * [2. 手动备份](#2-手动备份)

  * [3. 自定义备份路径与模式](#3-自定义备份路径与模式)

  * [4. 压缩包格式与选项](#4-压缩包格式与选项)

  * [5. 云存储设定 (Rclone)](#5-云存储设定-rclone)

  * [6. 消息通知设定 (Telegram)](#6-消息通知设定-telegram)

  * [7. 设置备份保留策略 (云端)](#7-设置备份保留策略-云端)

  * [8. Rclone 安装/卸载](#8-rclone-安装卸载)

  * [9. 从云端恢复到本地](#9-从云端恢复到本地)

  * [10. \[助手\] 配置导入/导出](#10-助手-配置导入导出)

  * [11. 日志与维护](#11-日志与维护)

  * [0. 退出脚本](#0-退出脚本)

  * [99. 卸载脚本](#99-卸载脚本)

* [常见问题与故障排除](#常见问题与故障排除)

* [贡献](#贡献)

* [许可证](#许可证)

## 功能特性

本脚本提供以下核心功能：

1. **灵活的备份模式**：

   * **归档模式 (Archive)**：将源数据打包成压缩文件（ZIP 或 Gzip Tarball）后上传，支持历史版本保留和从云端恢复。适合重要文件和配置的快照备份。

   * **同步模式 (Sync)**：直接将本地目录结构同步到云端，保持云端与本地数据一致。效率高，适合频繁变动的大量文件。

2. **自动备份设定**：允许用户设置自动备份的间隔天数。配合 Cron Job，脚本将根据此间隔智能判断是否执行备份，无需频繁修改 Cron 任务。

3. **手动备份**：随时触发一次立即备份上传操作，并将备份过程和结果实时展示在终端。

4. **自定义备份路径**：用户可以根据需求指定一个或多个要备份的文件或文件夹的绝对路径。

5. **多样化压缩选项**：

   * 支持 `.zip` 和 `.tar.gz` 两种压缩格式。

   * 可调整压缩级别（1-9）。

   * 支持为 `.zip` 压缩包设置密码保护。

   * 支持**独立打包**（每个源一个压缩包）或**合并打包**（所有源打包成一个**压缩包）。

6. **广泛的云存储支持 (基于 Rclone)**：利用 Rclone 强大的后端支持，可以备份到：

   * **对象存储**：Amazon S3、Cloudflare R2、MinIO、Backblaze B2、Microsoft Azure Blob Storage 等 S3 兼容服务。

   * **云盘服务**：Mega.nz、pCloud (部分需要手动配置)。

   * **传统协议**：WebDAV、SFTP、FTP。

   * **功能性后端**：支持创建 `Crypt` (加密) 远程端对数据进行端到端加密，以及 `Alias` (别名) 远程端简化路径。

   * 提供助手引导创建常见远程端，并支持手动配置。

7. **备份后完整性校验**：在“归档模式”下，支持在文件上传成功后，对云端文件进行完整性校验，确保数据传输过程中未损坏（默认开启，可关闭以提高速度）。

8. **消息通知 (Telegram)**：在备份开始、压缩完成、上传成功/失败以及保留策略执行后，通过 Telegram 机器人发送详细的通知消息。

9. **备份保留策略 (云端)**：允许用户设置自动清理云端旧备份的策略，支持按数量（保留最新 N 个）或按天数（保留 N 天内）进行清理，有效管理存储空间。**注意：此策略仅对“归档模式”生效。**

10. **带宽限制**：可设置 Rclone 上传/下载的带宽限制，避免备份过程占用过多网络资源。

11. **从云端恢复**：提供命令行菜单界面，帮助用户从已配置的云存储目标浏览并下载历史备份文件到本地进行恢复（仅适用于“归档模式”生成的备份）。

12. **配置导入/导出**：方便地将所有配置导出到文件，或从文件导入配置，便于多设备部署或迁移。

13. **详细日志管理**：支持设置终端和文件日志级别（DEBUG, INFO, WARN, ERROR），并可方便地查看日志文件，便于排错。

## 系统要求

* **操作系统**：Linux 或 macOS (推荐在 Linux 服务器环境使用)。

* **Shell**：Bash (脚本使用 Bash 语法编写)。

* **必要工具 (依赖项)**：

  * `rclone`: 核心工具，用于与云存储进行数据传输和管理。

  * `zip`, `unzip`: 用于 `.zip` 文件的压缩和解压。

  * `tar`: 用于 `.tar.gz` 文件的压缩。

  * `curl`: 用于安装 Rclone 和发送 Telegram 消息。

  * `realpath`: 用于解析文件真实路径 (通常包含在 `coreutils` 包中)。

  * `df`, `du`: 用于检查磁盘空间 (通常包含在 `coreutils` 包中)。

  * `less`: 用于分页查看日志文件。

* **Cron 服务**：用于设置自动备份的定时任务 (Linux/macOS 系统自带)。

## 安装与设置

请按照以下步骤将脚本安装到你的系统并进行初步设置：

1. **保存脚本文件**

   * 在你的终端中，创建一个名为 `personal_backup_rclone.sh` 的文件。例如，将其保存在你的主目录 (`~`) 下：

     ```bash
     nano ~/personal_backup_rclone.sh
     ```

   * 将本脚本文件（在 GitHub 仓库中将是 `personal_backup_rclone.sh` 的文件内容）完整复制并粘贴到 `nano` 编辑器中。

   * 保存并退出 `nano`：按下 `Ctrl + X`，输入 `Y`，然后按 `Enter`。

2. **给予脚本执行权限**

   * 在终端中执行以下命令，使脚本可运行：

     ```bash
     chmod +x ~/personal_backup_rclone.sh
     ```

3. **设置快捷启动 (可选，但推荐)**

   * 为了方便每次输入 `bf` 即可运行脚本，请在你常用的 Shell 配置文件（如 `~/.bashrc` 或 `~/.zshrc`）中添加一个别名：

     ```bash
     nano ~/.bashrc  # 或 nano ~/.zshrc
     ```

   * 滚动到文件底部，添加以下行：

     ```bash
     alias bf='bash ~/personal_backup_rclone.sh'
     ```

     **注意**：如果 `personal_backup_rclone.sh` 文件不在你的主目录，请替换为其实际的完整路径。

   * 保存并退出 (`Ctrl + X`, `Y`, `Enter`)。

   * 使配置立即生效：

     ```bash
     source ~/.bashrc  # 或 source ~/.zshrc
     ```

     或者，直接关闭并重新打开终端。

4. **安装脚本依赖**

   * 首次运行脚本时，它会检测并提示你安装缺失的依赖项。脚本内置了交互式安装助手，你可以根据提示选择安装。

   * 手动安装命令示例（根据您的操作系统）：

     * **Debian/Ubuntu 系统**：

       ```bash
       sudo apt update
       sudo apt install zip unzip tar realpath coreutils curl rclone less
       ```

     * **CentOS/RHEL 系统**：

       ```bash
       sudo yum install zip unzip tar coreutils curl rclone less
       # 或 sudo dnf install zip unzip tar coreutils curl rclone less
       ```

     * **macOS 系统 (使用 Homebrew)**：

       ```bash
       brew install zip unzip tar coreutils curl rclone less
       ```

   * `rclone` 的安装也可以通过脚本的 `[8. Rclone 安装/卸载]` 菜单选项来完成。

5. **设置 Cron Job (实现自动备份)**

   * 为了让自动备份功能生效，你需要设置一个 Cron Job，让它每天定期运行你的脚本来检查是否到了备份时间。

   * 在终端中打开 Cron 表编辑器：

     ```bash
     crontab -e
     ```

   * 如果之前有设置过旧的备份 Cron 任务，请务必将其删除。

   * 在文件底部添加以下一行新的 Cron Job 条目（请替换为你的脚本实际完整路径和希望的执行时间）：

     ```cron
     0 0 * * * bash /root/personal_backup_rclone.sh check_auto_backup > /dev/null 2>&1
     ```

     **重要**：请务必将 `/root/personal_backup_rclone.sh` 替换为你脚本的**实际完整路径**。
     这行 Cron Job 的含义是：每天的午夜 00:00，执行你的备份脚本并带上 `check_auto_backup` 参数。脚本会根据你设定的间隔智能判断是否需要执行备份。

     **解释这行 Cron Job：**

     * `0 0 * * *`: 这表示每天的午夜 00:00（即凌晨 12 点）执行。

       * 第一个 `0`: 每小时的第 0 分钟。

       * 第二个 `0`: 每天的第 0 小时。

       * `* * *`: 每天、每月、每周的每一天。

     * `bash /root/personal_backup_rclone.sh check_auto_backup`: 这是要执行的命令，即运行你的 Bash 脚本，并传递 `check_auto_backup` 参数，让脚本以自动备份模式运行。

     * `> /dev/null 2>&1`: 这会将脚本的所有标准输出 (stdout) 和标准错误 (stderr) 重定向到 `/dev/null`，意味着这些输出将不会显示在终端或邮件中，以避免 Cron 任务产生过多的输出。

   * 保存并退出 Cron 表 (`Ctrl + X`, `Y`, `Enter`)。

## 使用方法

安装完成后，你可以在终端中输入 `bf` 来启动脚本。

### 主菜单导航

脚本启动后，你将看到一个清晰的主菜单（在终端中将带有颜色和边框效果）：
```
================ 个人自用数据备份 ================
==================== 功能选项 ====================
1.自动备份设定 (当前间隔: X 天)
2.手动备份
3.自定义备份路径 (当前路径: 未设置/已设置路径)
4.压缩包格式 (当前支持: ZIP)
5.云存储设定 (支持: S3/R2, WebDAV)
6.消息通知设定 (Telegram)
7.设置备份保留策略 (云端)
0.退出脚本
99.卸载脚本
请输入选项:
```
你可以通过输入对应的数字选项来执行功能。

### 1. 自动备份与计划任务

选择此选项以进入子菜单，管理自动备份的间隔天数和配置 Cron 定时任务：

* **设置自动备份间隔**：设定脚本执行自动备份的最小天数间隔。

* **\[助手\] 配置 Cron 定时任务**：辅助你向系统 `crontab` 添加一个定时任务，确保脚本每天在你指定的时间运行，并根据间隔判断是否执行备份。

### 2. 手动备份

选择此选项会立即触发一次备份操作。脚本将按照你设定的备份路径、打包策略和模式进行压缩（如果适用），并上传到所有已配置并启用的 Rclone 目标。

### 3. 自定义备份路径与模式

进入此子菜单来管理要备份的源文件/文件夹，并设置备份模式和打包策略：

* **添加新的备份路径**：输入你希望备份的文件或文件夹的绝对路径。

* **查看/管理现有路径**：查看已添加的路径，并可修改或删除。

* **设置打包策略**：选择在“归档模式”下，多个备份源是“每个源单独打包”还是“所有源打包成一个”压缩包。

* **设置备份模式**：在“归档模式”和“同步模式”之间切换。

### 4. 压缩包格式与选项

此菜单用于配置备份文件的压缩方式：

* **切换压缩格式**：在 `.zip` 和 `.tar.gz` 之间切换。

* **设置压缩级别**：为压缩操作设置压缩率（1 最快，9 最高）。

* **设置/清除 ZIP 密码**：为 ZIP 压缩文件设置密码，仅对 `zip` 格式有效。

### 5. 云存储设定 (Rclone)

这是配置 Rclone 云存储目标的核心菜单：

* **查看、管理和启用备份目标**：列出所有已配置的 Rclone 目标，并允许添加新的目标（远程端名称 + 路径）、删除、修改路径，以及切换目标的启用/禁用状态。

* **\[助手\] 创建新的 Rclone 远程端**：提供向导式的菜单，帮助你轻松创建各种常见的 Rclone 远程端，例如 S3 兼容（R2, AWS S3, MinIO）、Backblaze B2、Azure Blob、Mega.nz、pCloud、WebDAV、SFTP、FTP，以及功能性远程端如 `Crypt` 和 `Alias`。

* **测试 Rclone 远程端连接**：选择一个已配置的 Rclone 远程端，测试其连接性并显示基本信息。

* **设置带宽限制**：为 Rclone 的上传和下载操作设置带宽限制。

* **备份后完整性校验**：开启或关闭备份文件上传后的完整性校验功能。

### 6. 消息通知设定 (Telegram)

配置 Telegram 机器人，以便在备份过程中接收通知：

* **启用/禁用通知**：切换 Telegram 通知功能。

* **设置/修改凭证**：输入你的 Telegram Bot Token 和 Chat ID。**注意：这些凭证会被保存到脚本的配置文件中。**

### 7. 设置备份保留策略 (云端)

配置云端备份文件的自动清理策略：

* **按数量保留**：指定只保留最新的 N 个备份文件。

* **按天数保留**：指定只保留最近 N 天内创建的备份文件。

* **关闭保留策略**：取消任何自动清理行为，所有备份将永久保留。
  **注意：此功能仅对“归档模式”创建的备份文件有效。**

### 8. Rclone 安装/卸载

提供方便的选项来管理 Rclone 的安装：

* **安装或更新 Rclone**：使用 Rclone 官方脚本安装或更新 Rclone。

* **卸载 Rclone**：从系统中移除 Rclone 本体程序。

### 9. 从云端恢复到本地

选择此选项以从云端浏览并下载备份文件进行恢复：

* 你将需要选择一个已启用的 Rclone 目标。

* 脚本会列出该目标下的 `.zip` 或 `.tar.gz` 备份文件。

* 选择要下载的文件，然后选择是直接解压到指定目录还是仅列出其内容。

* **注意：此功能仅适用于“归档模式”生成的备份文件。**

### 10. \[助手\] 配置导入/导出

此功能用于备份或恢复脚本的配置：

* **导出配置到文件**：将当前所有设置保存到一个配置文件中，通常位于脚本所在目录。

* **从文件导入配置**：从一个指定的配置文件中加载设置，这将覆盖当前所有配置，并需要重启脚本以生效。

### 11. 日志与维护

管理脚本的日志行为和进行一些维护操作：

* **设置日志级别**：分别为终端输出和文件日志设置详细程度（DEBUG, INFO, WARN, ERROR）。

* **查看日志文件**：使用 `less` 命令分页查看脚本的运行日志。

### 0. 退出脚本

安全退出脚本。

### 99. 卸载脚本

此选项会提示你是否确认卸载脚本。如果确认，脚本将删除自身文件、配置文件 (`~/.config/personal_backup_rclone/config`) 和日志文件 (`~/.local/share/personal_backup_rclone/log.txt`) 及轮转日志。

## 常见问题与故障排除

这里列出了一些在开发和使用过程中可能遇到的常见问题及其解决方案：

1. **“`/path/to/personal_backup_rclone.sh: line X: command not found` 或 `syntax error`”**

   * **问题原因**：这通常是由于在复制粘贴脚本代码时，引入了不可见的特殊字符（如不间断空格 `\u00A0`），或者文件编码/换行符格式问题（例如在 Windows 系统下编辑后上传到 Linux）。Bash 无法识别这些隐藏字符。

   * **解决方案**：

     1. **重新复制粘贴**：最彻底的方法是再次从 GitHub 仓库中复制 `personal_backup_rclone.sh` 的内容。

     2. **使用 `nano` 编辑器**：在 Linux 终端中，用 `nano ~/personal_backup_rclone.sh` 打开文件。

     3. **清空内容**：在 `nano` 中按住 `Ctrl + K` 连续删除所有旧代码。

     4. **粘贴新代码**：使用 `Ctrl + Shift + V`（Linux）或 `Cmd + V`（macOS）粘贴最新、干净的代码。

     5. **清除换行符 (可选)**：保存并退出后，可以尝试运行 `dos2unix ~/personal_backup_rclone.sh` 命令（如果系统没有，先安装 `sudo apt install dos2unix`）。

     6. **避免跨系统编辑**：尽量避免在 Windows 系统上编辑此脚本，直接在 Linux 终端中操作。

2. **“检测到以下依赖项缺失：rclone, zip 等”**

   * **问题原因**：你的系统缺少运行备份功能所需的工具。

   * **解决方案**：按照脚本提示，根据你的操作系统类型，在终端中运行相应的安装命令（参考 [安装与设置](#安装与设置) 部分）。脚本也提供了交互式安装助手，你可以直接在主菜单中选择安装。

3. **Rclone 连接/上传失败，提示“配置、凭证和网络连接”问题**

   * **问题原因**：

     * Rclone 远程端配置信息（如 Access Key, Secret Key, Endpoint URL, 用户名, 密码等）输入错误。

     * Rclone 远程端名称或目标路径不正确。

     * 网络连接问题，无法访问云存储服务。

     * 云服务提供商的防火墙规则阻止了连接。

   * **解决方案**：

     * 仔细检查你在脚本菜单 \[5. 云存储设定 (Rclone)\] -> \[1. 查看、管理和启用备份目标\] 或 \[2. \[助手\] 创建新的 Rclone 远程端\] 中输入的各项信息，确保它们与你的云服务提供商提供的信息完全一致。

     * 使用菜单 \[5. 云存储设定 (Rclone)\] -> \[3. 测试 Rclone 远程端连接\] 来诊断连接问题。

     * 对于某些需要浏览器认证的 Rclone 类型（如 Google Drive, Dropbox），你可能需要手动在终端运行 `rclone config` 命令进行配置。

     * 检查服务器的防火墙设置，确保允许对外访问云存储服务端口。

4. **Telegram 消息通知无法发送**

   * **问题原因**：

     * Telegram Bot Token 或 Chat ID 不正确。

     * 网络连接问题，无法访问 Telegram API (`api.telegram.org`)。

     * Bot 被停止或权限不足。

   * **解决方案**：

     * 仔细核对你在菜单 \[6. 消息通知设定 (Telegram)\] 中输入的 Bot Token 和 Chat ID。

     * 确保你的服务器可以访问 `api.telegram.org`。

     * 在 Telegram 中检查你的机器人是否正常运行，并确保它有向你发送消息的权限。

5. **临时目录空间不足导致备份失败**

   * **问题原因**：在“归档模式”下，脚本会在临时目录创建压缩文件。如果待备份的数据量很大，临时目录所在的分区可能没有足够的空间。

   * **解决方案**：

     * 清理临时目录所在分区的空间。

     * 更改临时目录的默认位置（这需要修改脚本中的 `TEMP_DIR` 变量，并确保新位置有足够的读写权限和空间）。

## 贡献

欢迎对本脚本进行改进和贡献！如果你有任何建议、Bug 报告或功能请求，请在 GitHub 仓库中提交 Issue 或 Pull Request。

## 许可证

本项目采用 [MIT 许可证](LICENSE)。

感谢使用！希望这个脚本能帮助你轻松管理个人数据备份
