---
layout: post
title: 'Log for Shell'
subtitle: '以及Shell脚本规范'
date: 2018-09-28
categories: 技术
cover: 'https://7ippo.github.io/assets/img/Manhattan.jpg'
tags: Shell 脚本规范 日志
---

<h2>目标</h2>
<p>Shell脚本面向内部技术人员，编写目标如下：   
</p>
<ul>
<li>
<p>脚本编写简单小巧，结构清晰，便于维护</p>
<ol>
<li>脚本内容需要分为若干步骤，同时打印步骤序号和步骤名称</li>
<li>一个脚本做的事情不超过5步，脚本内容过多则进行相应拆分,避免某一个脚本过于复杂</li>
<li>脚本的每个步骤以及函数实现遵循单一功能原则，代码长度控制在50行以内（50行大概是整个桌面窗口最大化后能全部展示的行数）</li>
<li>文件部署尽量清晰，及时清理废弃脚本和废弃文件夹，被注释掉的无用代码及时删除</li>
<li>入口脚本应该提供<strong>-help</strong>，<strong>-version</strong>功能
<br/><br/></li>
</ol>
</li>
<li>
<p>日志输出可以快速定位问题，打印格式规范统一</p>
<ol>
<li>日志止于出错位置，并且能够明确指出错误位置</li>
<li>能够看到执行脚本的脚本名与执行时的具体参数</li>
<li>关键日志和非关键日志区分开来</li>
<li>不打印看起来没有意义或没有人会看的日志</li>
<li>调试脚本时打印的辅助日志，调试完成后请<strong>务必</strong>删掉</li>
</ol>
</li>
</ul>
<h2>文件部署</h2>

{% highlight python %}
>build_scripts
    >bin  业务脚本
    >lib  引用库
        >tools  内部工具
        ...
    >etc  配置文件，包括通用配置与本地配置
        >local_etc  本地配置
            >default  默认配置
            >using_etc*  本地使用的环境配置
        ...
    >tmp*  临时文件
        >log*  日志
        ...
    >server  服务脚本
{% endhighlight %}
<ul>
<li>local_etc目录下存放着default与using_etc两个配置目录，default纳入版本控制系统，下存放着默认的环境配置。using_etc为default的拷贝，为机器最终使用的环境配置，方便做动态增加机器，环境变量的特殊修改和还原操作。</li>
<li>带星号*的目录不会纳入版本控制系统</li>
</ul>
<h2>规约</h2>
<h3>命名风格统一</h3>
<ul>
<li>文件名、内部变量名使用&nbsp;<strong><em>lowerCamelCase</em></strong>&nbsp;小写驼峰形式，如:&nbsp; buildServerConsole.sh，resultNum</li>
<li>函数名使用&nbsp;<strong><em>UpperCamelCase</em></strong>&nbsp;大写驼峰形式，如:&nbsp; ReportFailure，AfterShell</li>
<li>常量名、路径名使用以下划线连接的全大写单词形式，如:&nbsp;QUIET_RUN，SCRIPTS_AUTOBUILD_DIR</li>
<li>命名时，除非某个单词有业内通用的缩写简写，否则不要使用单词缩写简写命名，<strong>特别不要把自己平时的缩写简写写入公共脚本</strong></li>
</ul>
<h3>日志输出规范</h3>
<p>编写了一套Shell的日志工具脚本:[☆logForShell代码☆](https://github.com/7ippo/logForShell)。目前能够实现：统一格式打印，带TAG输出，分步执行，时间戳，出错后停止输出日志，出错后打印Shell堆栈，区分关键日志与非关键日志。编写Shell脚本时请按照以下规范调用日志方法。</p>
<ol>
<li><strong>入口脚本</strong>中，在调用任何日志方法之前，请先调用&nbsp;<strong><em>CleanFlagBeforeStart</em></strong>&nbsp;函数。该函数会清除上一次可能因执行失败而留下的标志。</li>
<li>
<p>脚本开始时，调用&nbsp;<strong><em>BeforeShell</em></strong>&nbsp;函数，传入所有执行参数$@。该函数会打印执行脚本名称与所有参数：</p>
{% highlight python %}
BeforeShell $@

2018-09-05 [17:52:21]
2018-09-05 [17:52:21]    +   test.sh  BEGIN   +
2018-09-05 [17:52:21]    +   test.sh -n -r -android 90   +
2018-09-05 [17:52:21]
{% endhighlight %}
</li>
<li>
<p>脚本执行步骤时，打印步骤序号和步骤名称。调用&nbsp;<strong><em>Step</em></strong>&nbsp;函数，传入两个参数，第一个为步骤序号，第二个为步骤名称。该函数会打印步骤标志：</p>
{% highlight python %}
Step 1 "Copy Unity Resource"

2018-09-05 [17:52:21]
2018-09-05 [17:52:21] ==================== test.sh  STEP 1 : Copy Unity Resource ====================
2018-09-05 [17:52:21]
{% endhighlight %}
</li>
<li>
<p>脚本结束时，调用&nbsp;<strong><em>AfterShell</em></strong>&nbsp;函数。该函数会打印脚本结束标志：</p>
{% highlight python %}
AfterShell

2018-09-05 [17:52:21]
2018-09-05 [17:52:21]    +   test.sh  FINISH   +
{% endhighlight %}
</li>
<li>
<p>需要打印关键步骤成功的标志时，调用&nbsp;<strong><em>ReportSuccess</em></strong>&nbsp;函数(默认为空)，可以传入关键步骤名称参数。该函数会打印成功结果标志：</p>
{% highlight python %}
ReportSuccess "Build Dynamic Framework Project"	

****     Build Dynamic Framework Project SUCCESS      ****

{% endhighlight %}
</li>
<li>
<p>需要打印错误标志时，调用&nbsp;<strong><em>ReportFailure</em></strong>&nbsp;函数。该函数会打印调用者的行号、脚本名，并阻止之后的日志输出，并通过内建命令caller输出一个Shell的调用堆栈：</p>
{% highlight python %}
ReportFailure		

****     line 10 test.sh REPORTED FAILURE     ****


    CALLER LIST
 - line 10 a test.sh
 - line 14 b test.sh
 - line 17 main test.sh

{% endhighlight %}
</li>
<li>
<p>非关键命令，如svn操作，cp命令执行前请调用&nbsp;<strong><em>BlankLine</em></strong>&nbsp;打印一个空行</p>
</li>
<li>
<p>如果需要打印带TAG的Shell日志，可以在source logForShell.sh时传入TAG参数(默认为空)，则日志打印时均会带上该TAG输出：</p>
{% highlight python %}
source ./logForShell.sh [Shell]
Show "Using TAG to LOG"

2018-09-11 [10:08:26] [Shell] Using TAG to LOG
{% endhighlight %}
</li>
</ol>
<h3>其他规范</h3>
<ol>
<li>脚本设计编写时应考虑按功能模块拆分成独立脚本，执行时均使用sh或source执行，去掉分配脚本执行权限的过程</li>
<li>
<p>变量使用时要加花括号</p>
<blockquote>
<code>$Ditch => ${Ditch}</code>
</blockquote>
</li>
<li>
规范脚本的错误码，调用脚本后需要对脚本返回值$?进行相应处理，推荐每个调用其他脚本的脚本内部写一个处理返回值的函数。该函数最少应具备处理错误返回值的情况，如面对错误返回值不继续执行后续操作，需要清除相应标志位：</li>
{% highlight go %}
function HandleError(){
returnVal=$?
if [ $retVal -ne 0 ]; then
    Show " error exit status "$returnVal" last command"
    rm -f ${MARK_FILE}
    exit 0
fi
}
{% endhighlight %}
<li>
<code>引用命令输出时，使用$()而不用反引号``</code>
<blockquote>
<code><font size="3">`pwd` => $(pwd)</font></code>
</blockquote>
</li>
<li>
<p>打印变量时，把变量名和变量值一起打印</p>
<blockquote>
<code><font size="3">如Show "$PLATFORM",至少应该写成Show "PLATFORM:$PLATFORM"</font></code>
</blockquote>
</li>
<li>
定义函数时在函数名前加上<em>function</em>:</li>
{% highlight go %}
function UpperCamelCase(){
	...
}
{% endhighlight %}
<li>
if-else流程控制，then和条件判断请写在同一行:</li>
{% highlight go %}
if [ ... ];then
  ...
elif [ ... ]; then
  ...
else
  ...
fi
{% endhighlight %}

<li>区分$0与$BASH_SOURCE的区别，需要打印当前执行脚本名时使用$BASH_SOURCE
<br/>
<br/></li>
</ol>
## [☆logForShell代码☆](https://github.com/7ippo/logForShell)
{% highlight python %}
#!/bin/bash

# 全局执行结果标志位，有一步出错后日志不会继续输出
# 考虑到子Shell与父Shell进程间通信，使用与该文件同目录下的tag文件作为标志位进行信息传递和判断
export GLOBAL_FAIL_FLAG="$(cd $(dirname ${BASH_SOURCE[0]}); pwd )/globalFailFlag.tag"

# 是否加入标签TAG标志位，默认为default
# 如： . ./logForShell.sh default
# 需要加入TAG时，调用该脚本时传入TAG参数
# 如： . ./logForShell.sh [Shell]
if [ "${1}" != "default" ];then
	TAG_FOR_SHELL=${1}
else
	TAG_FOR_SHELL=""
fi

# 核心打印函数：用于区分关键与非关键日志
# 正常log前加时间戳，这样如svn cp等命令打印的内容没有时间戳
# 打印日志前加入自定义标签，方便过滤检索

function Show(){
	dateStr=`echo $(date +%Y-%m-%d) $(date +[%H:%M:%S])`
	if [ ! -f ${GLOBAL_FAIL_FLAG} ]; then
		echo "${dateStr} ${TAG_FOR_SHELL} $@"
	fi
}

# 空行打印函数：请在非关键日志如svn,cp,wget等命令前打印一个空行

function BlankLine(){
	echo ""
}

# 纯净打印函数：无时间戳无标签无格式

function NoFormatShow(){
	echo "$@"
}

# 步骤打印函数：需要传入$1：步骤序号，$2：步骤描述
# 如： Step 1 "Copy Unity Resource"

function Step(){
	sourceFileName=`caller`
	Show ""
	Show "==================== ${sourceFileName##*/}  STEP $1 : $2 ===================="
	Show ""
}

# 进入脚本函数：在关键脚本开始时执行，调用时需要传入执行脚本的所有参数
# 如： BeforeShell $@

function BeforeShell(){
	sourceFileName=`caller`
	Show ""
	Show "   +   ${sourceFileName##*/}  BEGIN   +"
	Show "   +   ${sourceFileName##*/} $@   +"
	Show ""
}

# 退出脚本函数：在脚本结束时执行
# 如： AfterShell

function AfterShell(){
	sourceFileName=`caller`
	Show ""
	Show "   +   ${sourceFileName##*/}  FINISH   +"
}

# 错误报告函数：在某一步结束后判断执行结果，若出错则调用该函数
# 如： ReportFailure
# 该函数会改变执行结果标志位，阻止出错后日志继续输出

function ReportFailure(){
	touch ${GLOBAL_FAIL_FLAG}
	echo ""
	echo ""
	echo "****     line `caller` REPORTED FAILURE     ****"
	echo ""
	echo ""
	# 通过循环判断caller返回，如果不为空则持续打印，最多打印5行
	echo "    CALLER LIST    "
	echo " - line `caller 0`"
	for loop in 1 2 3 4
	do
		if [ "`caller ${loop}`" != "" ];then
			echo " - line `caller ${loop}`"
		else
			echo ""
			break
		fi
	done
}

# 成功报告函数：在关键步骤结束后判断执行结果，成功则调用该函数
# 一般步骤不用调用该函数输出
# 如： ReportSuccess "Init and check related params"

function ReportSuccess(){
	echo ""
	echo "****     $1 SUCCESS      ****"
	echo ""
}

# 清理标志位函数：在加载logForShell函数后，务必在主入口脚本所有语句执行前调用

function CleanFlagBeforeStart(){
	if [ -f ${GLOBAL_FAIL_FLAG} ]; then
		rm -f ${GLOBAL_FAIL_FLAG}
	fi
}
{% endhighlight %}
