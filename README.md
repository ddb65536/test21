#!/bin/sh
# CSG/OpenWrt 感染检查脚本
#
# 建议阅读顺序：先看最下面的 main，再看感兴趣的 check_xxx 函数。
# main 只是按顺序调用检查函数；每一个 check_xxx 对应一个检查项目。
# title、read_text、get_hash 等是多个检查项目共用的小工具函数。
#
# 每个检查项目的完整输出追加到 /var/log/system_log，不清空已有内容。
# 检查结束后另存一份“型号名_MAC.log”，用 rtty -S 发送到当前终端。
# 不修改配置，不清日志/历史，不停止服务，不执行待检查文件。
# 读取可能更新文件访问时间；登录、进程和平台审计仍会正常记录。
#
# 下面是脚本里经常出现的语法：
#   "$变量"       读取变量；加引号是为了正确处理带空格的路径。
#   $(命令)       把命令的输出保存到变量中，不是执行文件内容。
#   $1、$2        函数收到的第一个、第二个参数。
#   local         定义只在当前函数中使用的变量，避免影响其他函数。
#   return        结束当前函数，主流程继续执行其他检查项目。
#   continue      跳过本次循环，继续检查下一个文件或进程。
#   [ -f 路径 ]   判断是不是普通文件。
#   [ -d 路径 ]   判断是不是目录。
#   [ -L 路径 ]   判断是不是符号链接（类似快捷方式）。
#   [ -e 路径 ]   判断路径是否存在；断开的符号链接需另外用 -L 判断。
#   [ -r 路径 ]   判断文件是否可读。
#   [ -x 路径 ]   判断文件是否可执行。
#   !             把判断结果反过来，例如 ! -f 表示不是普通文件。
#   &&            两个判断必须都成立；这里只用于简单的文件属性判断。
#   ||            两个判断满足任意一个即可。
#   |             把左边的输出交给右边的命令处理，不创建中间文件。
#   2>&1          把错误信息也放到报告中。
#   2>/dev/null   丢弃预期的错误提示，不清理任何已有日志。
#   $?            上一条命令的返回码；0 表示成功，非 0 表示失败。

# 本脚本不接收参数，日志位置固定。
# $# 是参数数量，-ne 0 表示数量不等于 0。
if [ "$#" -ne 0 ]; then
    printf '%s\n' 'Usage: /rom/bin/busybox sh urandom_seed_check.sh'
    exit 2
fi

# BB 保存 BusyBox 的路径，后面用 "$BB" ls、"$BB" ps 调用工具。
# 优先选择固件只读区的版本，减少执行已被木马替换的工具的风险。
# 例如 "$BB" md5sum 实际执行的是 /rom/bin/busybox md5sum。
# cat、ls、find、md5sum、readlink 等同样通过这个 ROM 工具调用。
if [ -x /rom/bin/busybox ]; then
    BB=/rom/bin/busybox
else
    BB=/bin/busybox
fi

if [ ! -x "$BB" ]; then
    printf '%s\n' 'ERROR: BusyBox not found.' >&2
    exit 3
fi

# LOG 是检查结果的固定位置。>> 表示追加，不覆盖已有日志。
LOG=/var/log/system_log

# 不把符号链接、管道或设备节点当作报告文件写入。
if [ -L "$LOG" ]; then
    printf '%s\n' 'ERROR: /var/log/system_log is a symlink.' >&2
    exit 1
fi
if [ -e "$LOG" ]; then
    if [ ! -f "$LOG" ]; then
        printf '%s\n' 'ERROR: /var/log/system_log is not a regular file.' >&2
        exit 1
    fi
fi
if [ ! -d /var/log ]; then
    printf '%s\n' 'ERROR: /var/log does not exist.' >&2
    exit 1
fi

# 新建报告只允许当前用户读写；不修改已有报告的权限。
umask 077
if ! printf '\n===== CHECK_START =====\n' >> "$LOG"; then
    printf '%s\n' 'ERROR: cannot append to /var/log/system_log.' >&2
    exit 1
fi

# 固定命令输出的语言和格式，方便不同设备之间对比。
LC_ALL=C
export LC_ALL
# 按要求统一使用 MD5，比对完整文件，不截取内容。
HASH=md5sum

# 共用函数 1：打印检查项目的标题。
title() {
    # printf 中的 \n 表示换行，%s 表示这里放一个字符串。
    printf '\n================%s================\n' "$1"
}

# 共用函数 2：显示文本文件的全部内容，不执行其中的内容。
read_text() {
    local file="$1"
    # 不跟随符号链接，避免误读链接指向的其他文件。
    if [ -L "$file" ]; then
        printf 'SKIP text: %s (symlink)\n' "$file"
        return
    fi
    # FIFO、设备节点、目录等都不是普通文件，不能当作文本读取。
    if [ ! -f "$file" ]; then
        printf 'SKIP text: %s (missing, symlink or non-regular)\n' "$file"
        return
    fi
    printf 'TEXT %s\n' "$file"
    if ! "$BB" cat "$file"; then
        printf '\nINCOMPLETE: failed to read %s\n' "$file"
    fi
    printf '\nEND_TEXT %s\n' "$file"
}

# 共用函数 3：计算文件哈希。只输出哈希值，方便其他函数比较。
# return 0 表示成功，return 1 表示失败；失败不能当成文件正常。
get_hash() {
    local file="$1"
    local result

    if [ -L "$file" ]; then
        return 1
    fi
    if [ ! -f "$file" ]; then
        return 1
    fi
    if [ ! -r "$file" ]; then
        return 1
    fi

    # 对整个普通文件计算哈希，不限制文件大小。
    if result=$("$BB" "$HASH" "$file" 2>/dev/null); then
        # 哈希命令的输出是“哈希值  文件名”。cut 只取第一个字段。
        printf '%s\n' "$result" | "$BB" cut -d ' ' -f 1
    else
        return 1
    fi
}

# 共用函数 4：显示文件信息和哈希，不读取特殊文件的内容。
# 单独显示文件时间，避免依赖某些固件没有编译的 stat applet。
show_file_time() {
    local file="$1"
    local mtime
    printf '文件修改时间（mtime）：\n'
    if ! "$BB" ls -ld --full-time "$file" 2>/dev/null; then
        # 老版本不支持完整时间格式时，仍保留普通 ls -l 的时间。
        "$BB" ls -ld "$file"
        printf '注意：当前ls只能显示简略时间，不能据此判断秒级时间。\n'
    fi
    printf '元数据变更时间（ctime，不是创建时间）：\n'
    if ! "$BB" ls -ldc --full-time "$file" 2>/dev/null; then
        printf '当前工具无法读取完整ctime。\n'
    fi
    # date -r 只读取文件修改时间，不设置系统时间。
    if [ ! -L "$file" ]; then
        if mtime=$("$BB" date -r "$file" +%s 2>/dev/null); then
            printf 'mtime=%s\n' "$mtime"
        fi
    fi
}

show_file() {
    local file="$1"
    local value
    printf '\nFILE %s\n' "$file"
    if [ ! -e "$file" ] && [ ! -L "$file" ]; then
        printf 'ABSENT\n'
        return
    fi
    # ls -l 显示权限、所有者、大小和可读的修改时间；-d 显示目录自身。
    # /etc/mp_ip_mapping 和异常文件都经过这里，不只打印哈希。
    show_file_time "$file"
    if [ -L "$file" ]; then
        printf 'LINK -> '
        "$BB" readlink "$file"
    elif [ -f "$file" ]; then
        if value=$(get_hash "$file"); then
            printf 'HASH %s %s\n' "$HASH" "$value"
        else
            printf 'INCOMPLETE: hash failed\n'
        fi
    else
        printf 'SKIP: special file content is not read\n'
    fi
}

# 共用函数 5：比较当前文件与 ROM 原始文件。
# 第一个参数是当前文件；第二个参数是固件只读区的对应文件。
# 不同只表示需要复核，也可能是用户装软件或自己修改过。
# 返回 0 表示相同或双方均不存在，1 表示不同，2 表示计算失败。
# 相同只打印一行；不同才打印文件的详细信息。
compare_file() {
    local file="$1"
    local rom="$2"
    local current_hash
    local rom_hash
    local current_link
    local rom_link

    if [ ! -e "$file" ] && [ ! -L "$file" ]; then
        if [ -e "$rom" ] || [ -L "$rom" ]; then
            printf 'REVIEW: ROM file is missing from live filesystem: %s\n' "$file"
            return 1
        fi
        printf 'ABSENT %s\n' "$file"
        return 0
    fi

    if [ -L "$file" ]; then
        # 链接不计算目标内容，只比较链接指向的位置。
        if [ -L "$rom" ]; then
            if ! current_link=$("$BB" readlink "$file"); then
                printf 'INCOMPLETE: cannot read link: %s\n' "$file"
                return 2
            fi
            if ! rom_link=$("$BB" readlink "$rom"); then
                printf 'INCOMPLETE: cannot read ROM link: %s\n' "$rom"
                return 2
            fi
            if [ "$current_link" = "$rom_link" ]; then
                printf 'SAME_LINK %s\n' "$file"
                return 0
            fi
        fi
        printf 'REVIEW: new or changed symlink: %s\n' "$file"
        show_file "$file"
        return 1
    fi

    if [ ! -f "$file" ]; then
        printf 'REVIEW: special file: %s\n' "$file"
        show_file "$file"
        return 1
    fi

    if ! current_hash=$(get_hash "$file"); then
        printf 'INCOMPLETE: cannot calculate MD5: %s\n' "$file"
        return 2
    fi
    if [ -L "$rom" ] || [ ! -f "$rom" ]; then
        printf 'REVIEW: no matching ROM file: %s MD5=%s\n' "$file" "$current_hash"
        show_file "$file"
        return 1
    fi
    if ! rom_hash=$(get_hash "$rom"); then
        printf 'INCOMPLETE: cannot calculate ROM MD5: %s\n' "$rom"
        return 2
    fi

    if [ "$current_hash" = "$rom_hash" ]; then
        printf 'SAME_CONTENT %s MD5=%s\n' "$file" "$current_hash"
        return 0
    fi

    printf 'REVIEW: differs from ROM: %s current_MD5=%s ROM_MD5=%s\n' "$file" "$current_hash" "$rom_hash"
    show_file "$file"
    return 1
}

# 文本文件先比较 MD5，只有新增或不同才显示全部正文。
compare_text_file() {
    compare_file "$1" "$2"
    if [ "$?" -eq 1 ]; then
        read_text "$1"
    fi
}

# 共用函数 6：递归检查目录，每个文件调用 compare_file。
# 参数 1：当前目录；参数 2：ROM 对应目录。
# 参数 3：text 表示不同才显示正文，metadata 表示不同只看文件信息和哈希。
check_directory() {
    local dir="$1"
    local rom="$2"
    local mode="$3"
    local file
    local relative
    printf '\nDIRECTORY %s\n' "$dir"
    if [ -L "$dir" ]; then
        show_file "$dir"
        printf 'REVIEW: directory symlink is not followed\n'
        return
    fi
    if [ ! -d "$dir" ]; then
        printf 'ABSENT\n'
        return
    fi
    # find 列出目录及其子目录，不跟随符号链接，不限制条目数量。
    # read 每次接收一个路径；IFS= 和 -r 用于保留空格和反斜杠。
    "$BB" find "$dir" -print 2>&1 |
    while IFS= read -r file; do
        # 根目录自身不需要当作文件检查。
        if [ "$file" = "$dir" ]; then
            continue
        fi
        # 只接受这个目录下的路径，其他输出按 find 错误记录。
        case "$file" in
            "$dir"/*)
                ;;
            *)
                printf 'INCOMPLETE: find error or unsupported filename: %s\n' "$file"
                continue
                ;;
        esac
        # 普通目录由 find 继续遍历；目录符号链接仍需要单独展示。
        if [ -d "$file" ]; then
            if [ ! -L "$file" ]; then
                continue
            fi
        fi
        # 去掉当前目录前缀，得到相对路径。
        # 例：dir=/etc/init.d，file=/etc/init.d/boot，结果 relative=boot。
        relative=${file#"$dir"/}
        if [ "$mode" = text ]; then
            compare_text_file "$file" "$rom/$relative"
        else
            compare_file "$file" "$rom/$relative"
        fi
    done
}

# 检查项目 1：记录采集时间、固件版本和运行环境。
# 这些信息用于区分不同设备，也有助于对齐云平台日志时间。
check_system() {
    title '系统信息检查'
    # UTC 时间不受设备配置的时区影响，但设备时钟本身仍可能不准确。
    printf 'START_UTC '
    "$BB" date -u '+%Y-%m-%dT%H:%M:%SZ'
    printf 'BUSYBOX %s\nHASH %s\n' "$BB" "$HASH"
    if [ "$BB" != /rom/bin/busybox ]; then
        printf 'WARNING: ROM BusyBox unavailable; mutable tool fallback\n'
    fi
    # UID 为 0 代表 root；普通账号可能无法读取部分信息。
    printf 'UID '
    "$BB" id -u
    "$BB" uname -a
    # 以下全部只是读取文件，不用 source 加载配置。
    read_text /proc/uptime
    read_text /proc/sys/kernel/hostname
    read_text /etc/openwrt_release
    read_text /etc/glversion
    read_text /etc/board.json
    read_text /proc/self/mounts
    read_text /proc/mounts
}

# 检查项目 2：读取现有系统日志，保留登录和攻击时间线。
# 日志先于进程、文件等大部分检查采集，尽量早点保存易失信息。
check_logs() {
    local tool
    title '系统日志检查'
    # 不启动日志跟随模式，不调用 logger，不重启日志服务。
    for tool in /rom/sbin/logread /rom/usr/sbin/logread /sbin/logread; do
        if [ ! -x "$tool" ]; then
            continue
        fi
        printf 'LOG_TOOL %s\n' "$tool"
        if [ "$tool" = /sbin/logread ]; then
            printf 'WARNING: mutable logread fallback\n'
        fi
        # 不加 -f，只读出当前全部日志后结束。
        "$tool"
        printf '\nLOGREAD_EXIT %s\n' "$?"
        printf '\nEND_LOG\n'
        return
    done
    printf 'INCOMPLETE: logread unavailable\n'
}

# 检查项目 3：查看所有可见进程，另外记录真实执行文件和命令参数。
# 进程名可以伪装，所以不能只看 ps 显示的名字。
check_processes() {
    local dir
    local pid
    local exe
    local command
    title '运行进程检查'
    "$BB" ps
    # /proc/123/exe 是 PID 123 的真实执行文件链接。
    # 链接末尾出现 (deleted) 时，可能是程序已被删除，但进程还活着。
    for dir in /proc/[0-9]*; do
        if [ ! -d "$dir" ]; then
            continue
        fi
        # basename 只取路径最后一部分，例如 /proc/123 得到 123。
        pid=$("$BB" basename "$dir")
        exe=$("$BB" readlink "$dir/exe" 2>/dev/null)
        # cmdline 的参数之间使用零字节分隔，tr 把它转换为空格方便阅读。
        command=$("$BB" tr '\000' ' ' < "$dir/cmdline" 2>/dev/null)
        printf 'PROCESS pid=%s exe=%s cmd=%s\n' "$pid" "$exe" "$command"
        # case 的 *名字* 表示字符串任意位置包含这个名字。
        # 分支里的 | 表示多个匹配条件，不是运行这些文件。
        case "$exe $command" in
            *monitord*|*ujaild*|*gl-helper*|*/config/.gl/*|*/glinet/.cache/*)
                printf 'IOC_NAME: correlate with executable and startup evidence\n'
                ;;
        esac
    done
}

# 检查项目 4：保存当前连接，不发起新连接。
check_network() {
    local file
    title '网络连接检查'
    # 直接使用 BusyBox，避开可能被木马替换的 netstat 包装脚本。-n 不解析域名。
    # -a 显示全部连接；-n 使用数字地址；-p 尽量显示对应进程。
    if [ -x /rom/bin/netstat ]; then
        /rom/bin/netstat -anp
    elif [ -x /rom/usr/bin/netstat ]; then
        /rom/usr/bin/netstat -anp
    else
        "$BB" netstat -anp
    fi
    printf '\nNETSTAT_EXIT %s\n' "$?"
    # 同时保留内核原始连接表；netstat 不支持时仍有辅助数据。
    for file in /proc/net/tcp /proc/net/tcp6 /proc/net/udp /proc/net/udp6; do
        read_text "$file"
    done
}

# 检查项目 5：记录 SOP 提到的 AW 相关遗留文件。
# 只展示事实，不能仅根据文件存在或时间认定是某个漏洞造成。
check_aw() {
    title 'AW相关遗留文件检查'
    show_file /etc/mp_ip_mapping
    printf 'NOTE: existence or mtime alone does not prove AW exploitation\n'
}

# 检查项目 6：核对 urandom_seed 木马链。
# 重点是启动脚本、升级保留条目、落地文件，以及 monitord 和 Lua 的关系。
check_seed() {
    local file
    local lua_hash
    local monitor_hash
    title 'urandom_seed木马检查'
    # 看启动脚本有没有额外执行 monitord、ulogd 等命令。
    compare_text_file /etc/init.d/urandom_seed /rom/etc/init.d/urandom_seed
    # 看攻击者是否把恶意文件加入“升级仍保留”的清单。
    compare_text_file /lib/upgrade/keep.d/netifd /rom/lib/upgrade/keep.d/netifd
    # 二进制记录完整文件的哈希，不把二进制内容混进文本日志。
    for file in /bin/ulogd /usr/bin/monitord /usr/sbin/ujaild /tmp/ujaild /var/run/monitord.pid; do
        compare_file "$file" "/rom$file"
    done

    # 两个文件都成功算出哈希后，再比较是否相同。
    if ! monitor_hash=$(get_hash /usr/bin/monitord); then
        return
    fi
    if ! lua_hash=$(get_hash /usr/bin/lua); then
        return
    fi
    if [ "$monitor_hash" = "$lua_hash" ]; then
        printf 'IOC_COMBINATION: monitord is a copy of Lua; check startup commands\n'
    fi
}

# 检查项目 7：检查此前 gl-helper 样本使用的隐藏目录。
# 目录存在只表示需要继续看内容，不能单独认定感染。
check_gl_helper() {
    local dir
    title 'gl-helper木马检查'
    for dir in /etc/config/.gl /etc/glinet/.cache /overlay/upper/etc/config/.gl /overlay/upper/etc/glinet/.cache; do
        check_directory "$dir" "/rom$dir" metadata
    done
}

# 检查项目 8：DHCP 获取地址时可能触发这里的脚本。
# 对比文件并显示正文，但绝不运行它们。
check_dhcp() {
    title 'DHCP脚本检查'
    # 先列出全部内容（含隐藏文件和子目录），方便看有哪些脚本及其时间。
    printf '\n当前目录完整列表：/etc/udhcpc.user.d\n'
    if [ -d /etc/udhcpc.user.d ] && [ ! -L /etc/udhcpc.user.d ]; then
        "$BB" ls -laR /etc/udhcpc.user.d
    else
        printf '目录不存在或为符号链接，未展开。\n'
    fi
    printf '\nROM原始目录列表：/rom/etc/udhcpc.user.d\n'
    if [ -d /rom/etc/udhcpc.user.d ] && [ ! -L /rom/etc/udhcpc.user.d ]; then
        "$BB" ls -laR /rom/etc/udhcpc.user.d
    else
        printf 'ROM目录不存在或为符号链接，未展开。\n'
    fi
    printf '\n下面逐个比对MD5，不同脚本才显示正文。\n'
    check_directory /etc/udhcpc.user.d /rom/etc/udhcpc.user.d text
}

# 检查项目 9：排查重启后自动运行的入口。
check_startup() {
    title '开机启动项检查'
    compare_text_file /etc/rc.local /rom/etc/rc.local
    compare_text_file /etc/inittab /rom/etc/inittab
    check_directory /etc/init.d /rom/etc/init.d text
}

# 检查项目 10：查看账号，不把密码哈希直接写入报告。
# 这里不用 awk，改成逐行读取，再用普通 if 判断。
check_accounts() {
    local name
    local password
    local uid
    local gid
    local comment
    local home
    local shell
    local rest
    local first_character
    local state

    title '系统账号检查'
    # passwd 每一行用冒号分隔，字段依次是：
    # 用户名、密码占位、UID、GID、备注、家目录、登录 Shell。
    # IFS=: 告诉 read 按冒号拆开这一行，放到后面的变量中。
    if [ ! -L /etc/passwd ] && [ -f /etc/passwd ]; then
        while IFS=: read -r name password uid gid comment home shell; do
            printf 'ACCOUNT %s uid=%s gid=%s shell=%s\n' "$name" "$uid" "$gid" "$shell"
            # root 以外还有 UID 0 的账号，需要人工核对用途。
            if [ "$uid" = 0 ]; then
                if [ "$name" != root ]; then
                    printf 'REVIEW: additional UID 0 account\n'
                fi
            fi
        done < /etc/passwd
    else
        printf 'INCOMPLETE: passwd missing or symlink\n'
    fi
    # shadow 的第二个字段是密码字段，只判断状态，不打印字段内容。
    if [ ! -L /etc/shadow ] && [ -f /etc/shadow ] && [ -r /etc/shadow ]; then
        while IFS=: read -r name password rest; do
            first_character=$(printf '%s\n' "$password" | "$BB" cut -c 1)

            if [ -z "$password" ]; then
                state=EMPTY_PASSWORD
            elif [ "$password" = x ]; then
                state=PLACEHOLDER_UNRESOLVED
            elif [ "$first_character" = '!' ]; then
                state=LOCKED_PASSWORD
            elif [ "$first_character" = '*' ]; then
                state=LOCKED_PASSWORD
            else
                state=PASSWORD_FIELD_PRESENT
            fi

            # 只打印用户名和上面判断出的状态。
            printf 'SHADOW_STATE %s %s\n' "$name" "$state"
        done < /etc/shadow
    else
        printf 'INCOMPLETE: shadow unreadable, missing or symlink\n'
    fi
}

# 检查项目 11：查看常见定时任务目录及 SOP 示例中的依赖文件。
# 不解析执行 Shell 命令，避免把任务中的恶意代码运行起来。
check_cron() {
    local dir
    local file
    title '定时任务检查'
    printf 'NOTE: check PROCESS lines for crond -c custom directories\n'
    for dir in /tmp/gl_crontabs /etc/gl_crontabs /etc/crontabs /etc/cron.d; do
        check_directory "$dir" "/rom$dir" text
    done
    # SOP 示例中的业务任务依赖文件；只比对，绝不 source 或执行。
    for file in /lib/functions/mptun.sh /lib/functions/modem.sh /usr/bin/traffic_statistics; do
        compare_file "$file" "/rom$file"
    done
    printf 'NOTE: other paths referenced by cron require separate review\n'
}

# 检查项目 12：设备、接口热插拔事件可能触发这里的脚本。
check_hotplug() {
    title '热插拔脚本检查'
    check_directory /etc/hotplug.d /rom/etc/hotplug.d text
}

# 检查项目 13：检查整个overlay可写层，不再只检查bin/sbin。
# 例如 /overlay/upper/etc/rc.local 对比 /rom/etc/rc.local。
# 文件只比较MD5；符号链接比较目标；特殊文件只记录信息，不读取内容。
check_overlay() {
    local root=/overlay/upper
    local file
    local relative
    local rom
    local result
    local total=0
    local same=0
    local changed=0
    local added=0
    local links=0
    local special=0
    local failed=0
    local skipped=0
    local differences=0

    title 'overlay文件差异检查'
    if [ -L "$root" ] || [ ! -d "$root" ]; then
        printf '检查未执行：/overlay/upper不存在或为符号链接，不能据此认为没有差异。\n'
        return
    fi

    # 管道右侧放在一组中，遍历结束后直接打印计数，不写临时文件。
    "$BB" find "$root" -print 2>&1 |
    {
        while IFS= read -r file; do
            if [ "$file" = "$root" ]; then
                continue
            fi
            case "$file" in
                "$root"/*)
                    ;;
                *)
                    printf '检查不完整：find输出异常 %s\n' "$file"
                    failed=$((failed + 1))
                    continue
                    ;;
            esac
            # 普通目录不计入文件数量；目录链接仍需要比较。
            if [ -d "$file" ] && [ ! -L "$file" ]; then
                continue
            fi

            # 去掉 /overlay/upper，保留 /etc/... 这样的路径。
            relative=${file#"$root"}
            rom="/rom$relative"
            # 报告是本次检查主动写的文件，不把它计为可疑改动。
            if [ "$relative" = "$LOG" ]; then
                printf '跳过本次报告文件：%s\n' "$file"
                skipped=$((skipped + 1))
                continue
            fi

            total=$((total + 1))
            compare_file "$file" "$rom"
            result=$?

            if [ "$result" -eq 0 ]; then
                if [ -e "$file" ] || [ -L "$file" ]; then
                    same=$((same + 1))
                else
                    printf '检查不完整：遍历后文件消失 %s\n' "$file"
                    failed=$((failed + 1))
                fi
            elif [ "$result" -eq 2 ]; then
                failed=$((failed + 1))
            elif [ -L "$file" ]; then
                links=$((links + 1))
            elif [ ! -f "$file" ]; then
                special=$((special + 1))
            elif [ ! -f "$rom" ] || [ -L "$rom" ]; then
                added=$((added + 1))
            else
                changed=$((changed + 1))
            fi
        done

        differences=$((changed + added + links + special))
        printf '\noverlay差异统计：\n'
        printf '已检查条目：%s\n' "$total"
        printf '相同：%s\n' "$same"
        printf 'MD5不同：%s\n' "$changed"
        printf '新增或无对应普通ROM文件：%s\n' "$added"
        printf '链接新增或目标不同：%s\n' "$links"
        printf '特殊条目（含可能的删除白化标记）：%s\n' "$special"
        printf '差异条目合计：%s\n' "$differences"
        printf '检查失败或遍历错误：%s\n' "$failed"
        printf '跳过本次报告：%s\n' "$skipped"
        printf '说明：统计文件、链接和特殊条目，不统计普通目录和权限差异；差异不等于感染。\n'
    }
}

# 初步结论：先快速检查已知落地路径和关键启动文件。
# 这里只给线索，不把“文件不同”直接判断成“已经确认感染”。
# 其他目录差异和进程细节继续看后面的逐项报告。
check_summary() {
    local file
    local current_md5
    local rom_md5
    local abnormal=0
    local failed=0

    title '初步异常结论'
    printf '检查时间（UTC）：'
    "$BB" date -u '+%Y-%m-%d %H:%M:%S'
    if [ "$BB" = /rom/bin/busybox ]; then
        printf '检查工具：/rom/bin/busybox，优先使用固件只读区工具。\n'
    else
        printf '工具警告：未找到ROM BusyBox，已回退到/bin/busybox，结果需额外核实。\n'
        failed=1
    fi

    # 已知可疑落地文件与隐藏目录。存在只能说明需要继续核实。
    for file in /bin/ulogd /usr/bin/monitord /usr/sbin/ujaild /tmp/ujaild /var/run/monitord.pid /etc/config/.gl /etc/glinet/.cache; do
        if [ -e "$file" ] || [ -L "$file" ]; then
            printf '待复核：发现已知可疑路径 %s\n' "$file"
            abnormal=$((abnormal + 1))
        fi
    done

    # 对关键文件先比一次MD5，保证报告开头就能看到主要差异。
    for file in /etc/init.d/urandom_seed /lib/upgrade/keep.d/netifd /etc/rc.local /etc/inittab; do
        if [ -L "$file" ] || [ -L "/rom$file" ]; then
            printf '待复核：关键文件存在符号链接 %s\n' "$file"
            abnormal=$((abnormal + 1))
            continue
        fi
        if [ ! -e "$file" ] && [ ! -e "/rom$file" ]; then
            continue
        fi
        if [ ! -f "$file" ] || [ ! -f "/rom$file" ]; then
            printf '待复核：关键文件或ROM基线缺失/类型异常 %s\n' "$file"
            abnormal=$((abnormal + 1))
            continue
        fi
        if ! current_md5=$(get_hash "$file"); then
            printf '检查不完整：无法计算MD5 %s\n' "$file"
            failed=1
            continue
        fi
        if ! rom_md5=$(get_hash "/rom$file"); then
            printf '检查不完整：无法计算ROM文件MD5 /rom%s\n' "$file"
            failed=1
            continue
        fi
        if [ "$current_md5" != "$rom_md5" ]; then
            printf '待复核：关键文件与ROM的MD5不同 %s\n' "$file"
            abnormal=$((abnormal + 1))
        fi
    done

    if [ -e /etc/mp_ip_mapping ] || [ -L /etc/mp_ip_mapping ]; then
        printf '待复核：存在/etc/mp_ip_mapping，时间见AW相关遗留文件检查；仅凭存在不能认定漏洞入口。\n'
        abnormal=$((abnormal + 1))
    fi

    if [ "$abnormal" -gt 0 ]; then
        printf '初步结果：发现%s处异常或待复核线索，详细信息及文件时间见下方报告。\n' "$abnormal"
    else
        printf '初步结果：未发现本次快速检查覆盖的已知异常线索。\n'
    fi
    if [ "$failed" -ne 0 ]; then
        printf '注意：存在工具回退或读取失败，初步结论不完整。\n'
    fi
    printf '说明：这里只快速核对已知路径和关键文件，不覆盖全部目录差异，也不代表设备一定安全。\n'
}

# 时间判断单独放在报告前面，避免关键证据埋在大量文件差异中。
check_seed_time() {
    local file
    local current_md5
    local rom_md5
    local mtime
    local earliest=0
    local latest=0
    local dir
    local command
    local stat_line
    local start_ticks
    local start_epoch
    local uptime_value
    local uptime_seconds
    local run_seconds
    local run_days
    local run_hours
    local run_minutes
    local run_remainder
    local boot_epoch=0
    local ticks_per_second=100
    local tick_source='假设100Hz，仅用于粗估，需核实本机CLK_TCK'
    local key
    local value
    local rest
    local tool
    local found=0

    title 'seed攻击时间初步判断'
    printf '采集时刻（设备本地时间及UTC偏移）：'
    "$BB" date '+%Y-%m-%d %H:%M:%S %z %Z'
    printf '首次入侵开始时间：尚不能仅凭这些文件确认。\n'
    printf '下面仅使用实际出现的seed持久化文件差异，ROM原始文件时间不参与推断。\n'

    # monitord可能在每次启动时重新复制，不能拿它的时间直接当首次感染时间。
    for file in /bin/ulogd /etc/init.d/urandom_seed /lib/upgrade/keep.d/netifd; do
        if [ -L "$file" ] || [ ! -f "$file" ]; then
            continue
        fi
        if [ -f "/rom$file" ] && [ ! -L "/rom$file" ]; then
            if ! current_md5=$(get_hash "$file"); then
                printf '无法比较关键文件：%s\n' "$file"
                continue
            fi
            if ! rom_md5=$(get_hash "/rom$file"); then
                printf '无法比较ROM基线：/rom%s\n' "$file"
                continue
            fi
            if [ "$current_md5" = "$rom_md5" ]; then
                continue
            fi
        fi
        printf '\n候选落地时间证据：%s\n' "$file"
        show_file_time "$file"
        if mtime=$("$BB" date -r "$file" +%s 2>/dev/null); then
            if [ "$earliest" -eq 0 ] || [ "$mtime" -lt "$earliest" ]; then
                earliest=$mtime
            fi
            if [ "$mtime" -gt "$latest" ]; then
                latest=$mtime
            fi
        fi
    done
    if [ "$earliest" -gt 0 ]; then
        printf '\n关键差异文件mtime最早候选（UTC）：'
        "$BB" date -u -d "@$earliest" '+%Y-%m-%d %H:%M:%S'
        printf '关键差异文件mtime最晚候选（UTC）：'
        "$BB" date -u -d "@$latest" '+%Y-%m-%d %H:%M:%S'
        printf '这只是当前文件时间范围，不等于首次入侵；复制、恢复备份、升级或改时间都可能影响它。\n'
    else
        printf '\n未取得可自动换算的seed差异文件时间；不能给出候选植入时间。\n'
    fi

    # btime是内核报告的本次开机时间（Unix秒）。
    if [ -r /proc/stat ]; then
        while read -r key value rest; do
            if [ "$key" = btime ]; then
                boot_epoch=$value
            fi
        done < /proc/stat
    fi
    # 有ROM getconf时使用真实CLK_TCK；没有时把100Hz假设明确写在报告里。
    for tool in /rom/usr/bin/getconf /rom/bin/getconf; do
        if [ -x "$tool" ]; then
            if value=$("$tool" CLK_TCK 2>/dev/null); then
                case "$value" in
                    ''|*[!0-9]*) continue ;;
                esac
                if [ "$value" -gt 0 ]; then
                    ticks_per_second=$value
                    tick_source=ROM工具读取的CLK_TCK
                    break
                fi
            fi
        fi
    done

    for dir in /proc/[0-9]*; do
        if [ ! -d "$dir" ]; then
            continue
        fi
        command=$("$BB" tr '\000' ' ' < "$dir/cmdline" 2>/dev/null)
        case "$command" in
            *monitord*|*ujaild*) ;;
            *) continue ;;
        esac
        found=1
        printf '\n采集时已在运行：%s %s\n' "$dir" "$command"
        if [ ! -r "$dir/stat" ]; then
            printf '缺少进程stat，无法估算该进程的启动时间。\n'
            continue
        fi
        stat_line=$("$BB" cat "$dir/stat")
        printf '原始进程stat：%s\n' "$stat_line"
        # 去掉PID和括号中的名称后，原第22字段starttime是剩余第20字段。
        start_ticks=$(printf '%s\n' "$stat_line" | "$BB" sed 's/.*) //' | "$BB" cut -d ' ' -f 20)
        case "$start_ticks" in
            ''|*[!0-9]*) printf '无法解析进程启动时钟值。\n'; continue ;;
        esac

        # 开机到现在的秒数，减去进程启动时的秒数，就是本次已运行时长。
        # uptime带小数，这里只取整数秒；不需要设备日期正确，也不需要btime。
        uptime_seconds=
        if [ -r /proc/uptime ]; then
            read -r uptime_value rest < /proc/uptime
            uptime_seconds=${uptime_value%%.*}
        fi
        case "$uptime_seconds" in
            ''|*[!0-9]*) printf '无法读取有效uptime，本次运行时长无法计算。\n' ;;
            *)
                run_seconds=$((uptime_seconds - start_ticks / ticks_per_second))
                if [ "$run_seconds" -ge 0 ]; then
                    run_days=$((run_seconds / 86400))
                    run_hours=$((run_seconds % 86400 / 3600))
                    run_minutes=$((run_seconds % 3600 / 60))
                    run_remainder=$((run_seconds % 60))
                    printf '可疑进程本次已运行约：%s天 %s小时 %s分钟 %s秒\n' "$run_days" "$run_hours" "$run_minutes" "$run_remainder"
                else
                    printf '进程启动值与uptime不一致，本次运行时长无法计算。\n'
                fi
                ;;
        esac
        printf '时长换算：CLK_TCK=%s（%s）；仅代表本次进程运行，不代表感染总时长。\n' "$ticks_per_second" "$tick_source"
        if [ "$boot_epoch" -le 0 ]; then
            printf '缺少本次开机时间btime，无法换算进程启动时间。\n'
            continue
        fi
        start_epoch=$((boot_epoch + start_ticks / ticks_per_second))
        printf '进程启动时间估算（UTC）：'
        "$BB" date -u -d "@$start_epoch" '+%Y-%m-%d %H:%M:%S'
        printf '换算依据：btime=%s starttime=%s CLK_TCK=%s（%s）。\n' "$boot_epoch" "$start_ticks" "$ticks_per_second" "$tick_source"
        printf '进程启动可能是重启或重新拉起，不代表首次入侵；换算还依赖设备时钟。\n'
    done
    if [ "$found" -eq 0 ]; then
        printf '\n采集时未找到monitord/ujaild命令行；不能据此排除历史感染或伪装进程。\n'
    fi
}

# 主流程：一项一项检查，各项的正常输出和错误信息都追加到 LOG。
# >> 追加，不清空旧报告；2>&1 把错误信息也写入同一个报告。
main() {
    check_summary >> "$LOG" 2>&1
    check_seed_time >> "$LOG" 2>&1
    check_system >> "$LOG" 2>&1
    #check_logs >> "$LOG" 2>&1
    check_processes >> "$LOG" 2>&1
    check_network >> "$LOG" 2>&1
    check_aw >> "$LOG" 2>&1
    check_seed >> "$LOG" 2>&1
    check_gl_helper >> "$LOG" 2>&1
    check_dhcp >> "$LOG" 2>&1
    check_startup >> "$LOG" 2>&1
    check_accounts >> "$LOG" 2>&1
    check_cron >> "$LOG" 2>&1
    check_hotplug >> "$LOG" 2>&1
    check_overlay >> "$LOG" 2>&1
    title '检查结束'
    printf 'COLLECTION_END\n'
    printf 'END_UTC '
    "$BB" date -u '+%Y-%m-%dT%H:%M:%SZ'
    printf 'No findings is not a clean-device verdict. Initial intrusion path is not established.\n'
}

# 生成便于区分设备的下载文件名，再通过当前rtty终端发送。
send_log() {
    local model
    local mac
    local download_log
    local tool

    # 型号只保留适合文件名的字符；MAC去掉冒号，避免Windows文件名含冒号。
    model=$("$BB" cat /proc/gl-hw-info/model 2>/dev/null | "$BB" tr -cd 'A-Za-z0-9._-')
    mac=$("$BB" cat /proc/gl-hw-info/device_mac 2>/dev/null | "$BB" tr -d ':\r\n')
    case "$mac" in
        ''|*[!A-Fa-f0-9]*) printf '无法读取有效设备MAC，报告保留在 %s\n' "$LOG"; return ;;
    esac
    if [ -z "$model" ] || [ "${#mac}" -ne 12 ]; then
        printf '缺少型号或MAC格式不正确，报告保留在 %s\n' "$LOG"
        return
    fi
    download_log="/var/log/${model}_${mac}.log"

    # 原始日志保留；同名的普通下载文件更新为本次完整报告。
    # 符号链接、管道或设备节点不覆盖。
    if [ -L "$download_log" ] || { [ -e "$download_log" ] && [ ! -f "$download_log" ]; }; then
        printf '下载文件路径类型异常，未复制：%s\n' "$download_log"
        return
    fi
    if ! "$BB" cp "$LOG" "$download_log"; then
        printf '复制下载日志失败，原报告在 %s\n' "$LOG"
        return
    fi
    printf '下载日志：%s\n' "$download_log"

    # 优先使用ROM中的rtty；没有时使用设备已安装的rtty。
    for tool in /rom/usr/bin/rtty /rom/bin/rtty /usr/sbin/rtty /bin/rtty; do
        if [ -x "$tool" ]; then
            if ! "$tool" -S "$download_log"; then
                printf 'rtty发送失败，可在rtty终端重试：rtty -S "%s"\n' "$download_log"
            fi
            rm "$download_log"
            /etc/init.d/log restart
            rm /var/log/system_log
            
            return
        fi
    done
    printf '未找到rtty，请手动下载上述日志文件。\n'
}

# 运行一次，不再做截取、数量限制或输出字符过滤。
# 开始/结束信息也追加到报告中，终端只显示报告位置。
main >> "$LOG" 2>&1
printf '检查结束，结果已追加到 %s\n' "$LOG"
send_log
