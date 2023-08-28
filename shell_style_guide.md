# Shell Style Guide

Which Shell to use
#!/bin/bash

When to use Shell
小型实用程序、封装简单的脚本

Shell的一些准则：
1、调用其他实用程序并且数据操作相对较少的情况
2、如果注重性能问题，不要使用shell
3、脚本长度超过100行或使用非直接控制流逻辑，应用结构化语言进行重构
4、评估代码复杂度时，考虑代码是否易于他人维护

# Shell文件与解释器调用
文件扩展名
可执行文件应该没有扩展名或者只有.sh扩展名

## 环境
STDOUT vs STDERR
err() {
  echo "[$(date +'%Y-%m-%dT%H:%M:%S%z')]: $*" >&2
}

if ! do_something; then
  err "Unable to do_something"
  exit 1
fi

## 注释
文件头部
注释包括：内容概要、版权申明、作者信息
#!/bin/bash
#
# Perform hot backups of Oracle databases.

## 函数注释
库文件中所有函数都必须添加注释。
函数注释包含：
- 函数描述
- Globals：全局变量列表
- Arguments：参数列表
- Output：输出到STDOUT内容、输出到STDERR的内容
- Returns：返回值

例如：
#######################################
# 清理备份目录
# Globals:
#   BACKUP_DIR
#   ORACLE_SID
# Arguments:
#   None
#######################################
function cleanup() {
  …
}

#######################################
# 获取配置目录
# Globals:
#   SOMEDIR
# Arguments:
#   None
# Outputs:
#   Writes location to stdout
#######################################
function get_dir() {
  echo "${SOMEDIR}"
}

#######################################
# 使用复杂的方式删除文件
# Arguments:
#   File to delete, a path.
# Returns:
#   0 if thing was deleted, non-zero on error.
#######################################
function del_thing() {
  rm "$1"
}

实现注释
对复杂的、不清晰的、有趣的、重要的代码，进行注释。

TODO注释：
对临时、短期的解决方案、足够好但不完美的恶代码，进行TODO注释
TODO，接着是相关人员名称、邮件等
# TODO(mrmonkey): Handle the unlikely edge cases (bug ####)

## 格式化
缩进： 使用两个空格，不使用制表符
在代码块之间，使用空行以提高可读性

行长度与长字符串
最大行长度为80个字符

## 管道
# 将所有管道放在一行
command1 | command2

# 每个管道都换行
command1 \
  | command2 \
  | command3 \
  | command4

## 循环语句
将 ; do、 ; then 放在while、for、if同一行
# 如果是在函数体内，考虑使用 local 定义 dir 变量，避免修改了全局变量：
# local dir
for dir in "${dirs_to_cleanup[@]}"; do
  if [[ -d "${dir}/${ORACLE_SID}" ]]; then
    log_date "Cleaning up old files in ${dir}/${ORACLE_SID}"
    rm "${dir}/${ORACLE_SID}/"*
    if (( $? != 0 )); then
      error_message
    fi
  else
    mkdir -p "${dir}/${ORACLE_SID}"
    if (( $? != 0 )); then
      error_message
    fi
  fi
done

## Case语句
- 对分支缩进2个空格
- 但行分支）之后，；；之前，都需要添加一个空格
- 长命令、多行命令需要将匹配表达式、动作、；；分开为多行
case "${expression}" in
  a)
    variable="…"
    some_command "${variable}" "${other_expr}" …
    ;;
  absolute)
    actions="relative"
    another_command "${actions}" "${other_expr}" …
    ;;
  *)
    error "Unexpected expression '${expression}'"
    ;;
esac

## 变量扩展
使用双引号界定变量，推荐${var} 而不是$var
推荐规范
- 与现有代码风格保持一致
- 使用双引号包括变量
- 不要使用{}界定特殊参数、位置参数、除非强制要求、否则应避免此类情况

  # 特殊变量的首选样式：
echo "Positional: $1" "$5" "$3"
echo "Specials: !=$!, -=$-, _=$_. ?=$?, #=$# *=$* @=$@ \$=$$ …"

# 必要的大括号:
echo "many parameters: ${10}"

# 使用大括号避免代码混淆:
# 输出 "a0b0c0"
set -- a b c
echo "${1}0${2}0${3}0"

# 其他变量的首选样式：
echo "PATH=${PATH}, PWD=${PWD}, mine=${some_var}"
while read -r f; do
  echo "file=${f}"
done < <(find /tmp)

注意：对变量使用大括号，如：${var}，产生的结果是无引号格式。如果需要引号，则双引号必须明确添加

引号
始终对（包含：变量、命令替换、空格、Shell 元字符的）字符串使用双引号。
使用数组来安全地引用列表元素，尤其是命令行选项。
推荐，对 Shell 内部只读特殊变量：$？、$＃、$$、$! 使用双引号。对 Shell 内置变量使用双引号，例如：PPID 等，以确保不会错解析成别的变量。
引号包裹的字符串最好是单词。
不要对数字字面量使用引号。
注意对 [[ … ]] 中匹配规则使用引号。
如无特殊情况，使用 $@ 代替 $*，例如：将参数附加到消息或日志中。

## 功能与BUG
ShellCheck
检查脚本中常见的错误和警告
### 命令替换
# 正例
var="$(command "$(command1)")"

# 反例
var="`command \`command1\``"

### Test、 [...] 与[[...]]
建议使用 [[ ... ]]，而不是 [ ... ]、test、/usr/bin/[。

因为 [[ ... ]] 能够减少不必要的错误，比如：[[ ... ]] 没有路径名扩展、单词分割；[[ ... ]] 允许正则表达式匹配，而 [ ... ] 不允许。

# 确保左侧的字符串由：字母+name 组成。
# 注意，这里的 RHS 表达式不能使用引号。
if [[ "filename" =~ ^[[:alnum:]]+name ]]; then
  echo "Match"
fi

# 明确匹配条件为：等于 "f*" 字符串（此处，结果为不匹配）
if [[ "filename" == "f*" ]]; then
  echo "Match"
fiCopy
# f* 会扩展为当前目录文件列表，因此会提示错误："too many arguments"。
if [ "filename" == f* ]; then
  echo "Match"
fi

# 测试字符串

# 正例
if [[ "${my_var}" == "some_string" ]]; then
  do_something
fi

# -z（字符串长度为 0）
# -n (字符串长度不为 0)
# 推荐用此来测试字符串是否为空
if [[ -z "${my_var}" ]]; then
  do_something
fi

# 这样也可以判断空字符串，但不是首选的：
if [[ "${my_var}" == "" ]]; then
  do_something
fi

避免混淆，明确使用 -z、-n：
# 正例
if [[ -n "${my_var}" ]]; then
  do_something
fiCopy
# 反例
if [[ "${my_var}" ]]; then
  do_something
fi

# 正例
if [[ "${my_var}" == "val" ]]; then
  do_something
fi

if (( my_var > 3 )); then
  do_something
fi

if [[ "${my_var}" -gt 3 ]]; then
  do_something
fi

# 文件名的通配符扩展
由于文件名可以是 - 开头，因此使用 ./* 比 * 通配符更为安全。

使用引用扩展 "${array[@]}" 访问数组。

# 命名约定
## 函数名称
小写、下划线分隔单词，用 :: 分隔库。函数名称后必须跟随括号。关键字 function 是可选的，但是在整个项目中必须保持一致。
# 单一函数
my_func() {
  …
}

# 包函数
mypackage::my_func() {
  …
}

## 变量名称
循环体中的变量名称应与被遍历的变量命名相似。

for zone in "${zones[@]}"; do
  something_with "${zone}"
done

# 常量与环境变量名称
在文件顶部声明、全大写字母、使用下划线分隔单词。
# 常量
readonly PATH_TO_FILES='/some/path'

# 常量与环境变量
declare -xr ORACLE_SID='PROD'

建议使用 readonly、export 而不是 declare 命令。
VERBOSE='false'
while getopts 'v' flag; do
  case "${flag}" in
    v) VERBOSE='true' ;;
  esac
done
readonly VERBOSE

## 源文件名称
小写、按需使用下划线分隔单词。

这是为了与 Google 中的其他代码风格保持一致：maketemplate、make_template 而不是 make-template。

## 使用局部变量
用 local 声明函数中的变量。声明与赋值应该分别写在不同的行上。
声明与赋值必须是分开的两个语句；
my_func2() {
  local name="$1"

  # 声明与赋值分开两行：
  local my_var
  my_var="$(my_func)"
  (( $? == 0 )) || return

  …
}

## 函数位置
在函数声明之前，只能包含：set 语句、变量声明语句、常量声明语句。
main
如果脚本比较长，并且包含至少一个其他函数时，必须定义 main 函数。

main "$@"
对简短的脚本，main函数会显得有些冗余，

## 调用命令
检查返回值
始终检查返回值并提供有用的返回值

内置命令与外部命令
首先内置命令



