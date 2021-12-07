---
layout: post
title: 'Apk Permissions Check'
subtitle: '利用ElementTree库解析apk检查AndroidManifest'
description: '用python的Elementree库解析安卓apk中的Androidanifest库，编写一些处理工具。检查apk声明的权限列表，删除重复声明的安卓组件'
date: 2018-11-06
author: 周子博
categories: 技术
cover: 'https://i.loli.net/2019/02/12/5c626fbda2e1c.jpg'
tags: Android Python
---

<h2>目的</h2>
<p>反编译apk并解析xml文件，检查其中是否存在设定白名单外的权限声明。该工具可以作为分发安卓包前工具链的一部分。</p>
<h2>使用说明</h2>
<ol>
<li>使用前请配置apkPermissionsCheck.py中的必要权限集合:NECESSARYPERMISSIONSSET。集合外的权限将会被警告打印到控制台</li>
<li>需要将apktool.jar放置在同目录下</li>
<li>
执行脚本
<blockquote>
<p><font size="3">python apkPermissionsCheck.py -path [ directory ] <br />
# -path/--path 检查某个路径下的所有apk <br />
或 <br />
python apkPermissionsCheck.py -apkname [ apk1 ] ( [apk2] ... ) <br />
# -apkname/--apkname 检查若干当前路径下的apk文件</font></p>
</blockquote>
</li>
</ol>

<h2>apkPermissionsCheck Code</h2>
<p><a href="https://github.com/7ippo/ApkPermissionsCheck">☆Github Repo☆</a></p>
<pre><code class="language-python">
import os
import re
import argparse
import xml.etree.ElementTree as ET

parser = argparse.ArgumentParser()
parser.add_argument('--path', '-path', required=False,
                    help='Optional, Directory path of a group of apk files')
parser.add_argument('--apkname', '-apkname', required=False,
                    help='Optional, Several apk files\' names', nargs='+')

NECESSARYPERMISSIONSSET = {'android.permission.RECEIVE_USER_PRESENT',
                           'android.permission.MOUNT_UNMOUNT_FILESYSTEMS',
                           'android.permission.CAMERA',
                           'android.permission.WRITE_EXTERNAL_STORAGE',
                           'android.permission.READ_EXTERNAL_STORAGE',
                           'android.permission.RECORD_AUDIO'}


def deCompile(filename):
    apktool_command = "java -jar apktool.jar d -f " + filename
    os.system(apktool_command)


def checkPermissionsList(filename):
    manifest_path = os.path.join("./" + filename + "/AndroidManifest.xml")
    ET.register_namespace(
        'android', 'http://schemas.android.com/apk/res/android')
    print(manifest_path)
    manifest_tree = ET.parse(manifest_path)
    manifest_root = manifest_tree.getroot()
    uses_permission_list = manifest_root.findall("uses-permission")

    warnings_permissions = []
    for permission in uses_permission_list:
        permission_name = permission.attrib[
            '{http://schemas.android.com/apk/res/android}name']
        if(re.search(r'^(android\.permission)\..*', permission_name)):
            if permission_name not in NECESSARYPERMISSIONSSET:
                if(permission_name not in warnings_permissions):
                    warnings_permissions.append(permission_name)
    warnings_permissions.append(filename)
    return warnings_permissions


def reportPermissionsWarnings(warnings_permissions):
    warning_nums = len(warnings_permissions)-1
    if warning_nums == 0:
        print()
        print('========{}========'.format("Checking Result..."))
        print()
        print('**CLEAR ~**\n')
        print('There is 0 WARNING Android permission in : {}'.format(
            warnings_permissions[0]))
    else:
        print()
        print('========{}========'.format("Checking Result..."))
        print()
        print('**WARNING !**\n')
        print('{} unnecessary permissions founded in : {}'.format(
            warning_nums, warnings_permissions[warning_nums]))
        for i in range(warning_nums):
            print('        {}\n'.format(warnings_permissions[i]))
    return


if __name__ == '__main__':
    args = parser.parse_args()
    if not(args.path) and not(args.apkname):
        parser.print_help()
        exit(0)
    if args.path:
        for apks in os.listdir(args.path):
            if(re.search(r'apk$', apks)):
                print()
                deCompile(apks)
                folder_name = re.sub(r'\.apk$', '', apks)
                warnings_permissions = checkPermissionsList(folder_name)
                reportPermissionsWarnings(warnings_permissions)
    elif args.apkname:
        for apks in args.apkname:
            deCompile(apks)
            folder_name = re.sub(r'\.apk$', '', apks)
            warnings_permissions = checkPermissionsList(folder_name)
            reportPermissionsWarnings(warnings_permissions)

</code>
</pre>
<h2>Another</h2>
<p>介于公司对AndroidManifest合并的逻辑太过粗暴导致重复权限声明&lt;uses-permission&gt;多达四五十个，遂顺手写了另一个小工具removeRedundantPermissions，同样解析AndroidManifest.xml，删除重复的权限声明</p>
<blockquote>
<p><font size="3">python removeRedundantPermissions.py -xml/--xml [ AndroidManifest1.xml ] ( [AndroidManifest2.xml] ...)<br />
# -xml/--xml 删除若干AndroidManifest.xml中的重复权限声明 </font></p>
</blockquote>
<h2>removeRedundantPermissions Code</h2>
<p><a href="https://github.com/7ippo/ApkPermissionsCheck">☆Github Repo☆</a></p>
<pre><code class="language-python">
import argparse
import xml.etree.ElementTree as ET

parser = argparse.ArgumentParser()
parser.add_argument('--xml', '-xml', required=True,
                    help='Several XML files', nargs='+')

if __name__ == '__main__':
    args = parser.parse_args()
    if args.xml:
        for xml in args.xml:
            ET.register_namespace(
                'android', 'http://schemas.android.com/apk/res/android')
            manifest_tree = ET.parse(xml)
            manifest_root = manifest_tree.getroot()
            uses_permission_list = manifest_root.findall("uses-permission")
            permission_list = []
            for permission in uses_permission_list:
                permission_name = permission.attrib[
                    '{http://schemas.android.com/apk/res/android}name']
                if permission_name not in permission_list:
                    permission_list.append(permission_name)
                else:
                    manifest_root.remove(permission)
            manifest_tree.write(xml)
</code>
</pre>