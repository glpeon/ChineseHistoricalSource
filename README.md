# ChineseHistoricalSource
This is my personal stash of Chinese Historical Source

声明：这是个个人项目，不要用于商业用途。

# 项目目的

我一直想用做一个史料查询系统，这样的话就能想用Google一样搜索不同史料了。目前在线能找到的各种史料大概只有txt和pdf版本，这两种都不是标准化数据，不能喂到索引系统里面。

史料的电子版

史料的文白对照不在本项目的范畴里。如果需要文白对照，我个人推荐一个移动应用"读典籍"：https://dudianji.com/mobile/

史料的电子化现在基本上弄得差不多了，这个项目就是想把电子史料标准化。

# 项目内容

这个小项目就是把txt版本的史料做两个操作：
1. 把不同汉字编码的史料标准化成utf8版本。
2. 把utf8版本的txt史料变成统一json格式

所以内容主要是三个：
1. 原始数据。
这个数据来源是"猪猪书网"：http://www.zzs5.com/txt/7871.html
（如有侵权请联系作者，我务必在第一时间删除）
2. 转码数据。
它是把上面的数据做成utf8格式的。
我用了一个在线工具"Subtitle Tools": https://subtitletools.com/convert-text-files-to-utf8-online
如果有史料上的不准确，我会在转码数据里改，而不会去改原始数据。
转码数据我也会做一下格式上的预处理。
3. json数据。这个数据就是可以放到索引系统里的标准化数据
4. 标准化算法。我用Ruby写一些Script，把utf8格式的txt史料parse成json。

# json格式
目前支持三个Attribute：史料、章节和原文。原文的每一句话都是一个json object。
范例：
```json
{
    "source": "史记",
    "chapter": "陈涉世家",
    "text": "陈胜者，阳城人也，字涉。"
}
```

# 工程进度
raw和utf8是纯手工操作，所以已完成。

Ruby Parsing已完成：
史记

# 工程笔记

## GCP Instance Setup

今天的云服务各种各样，但因为吸引的客户是企业而不是个人，所以没有多少是免费的。如果想免费只有两个办法，一种是短期试用。短期试用虽然自由，但难以持久。一种是长期但受限制。
限制的方法有限制容量和限制待机时间。
长期免费但受限制的服务我找到的只有两个：Heroku和Google Cloud Platform（GCP）。

Heroku目前不能做一个通用的Instance，只能做Application。我记得Heroku有一个免费的PostgreSQL，但有一个很紧的数量限制：10,000行。对于史料来说，这远远不够。
另一个原因是我本人不是很熟悉用SQL做索引系统，所以还是选择ElasticSearch。Heroku有一个ElasticSearch的Application，但不是免费的，所以只能放弃Heroku了。

GCP的选择就很多了。谷歌这个平台比较开放，support也很多。因为GCP基本就是谷歌版本的AWS，所以主打的业务就是通用的计算单元。AWS的计算单元叫EC2，谷歌叫Compute Engine。

谷歌虽然说是免费，但是把免费的方法藏得很深。稍有不慎就被谷歌收钱了。Medium有一篇文章讲了一下如何把Instance搞成免费的：https://medium.com/@hbmy289/how-to-set-up-a-free-micro-vps-on-google-cloud-platform-bddee893ac09
简单来说就是，机子的类型必须是f1-micro VPS，区域必须是us-central1 (Iowa), us-east1 (South Carolina)或者us-west1 (Oregon)。磁盘容量可以调成30G。

谷歌的ssh比AWS人性化多了，直接用浏览器当Terminal，非常爽。不像AWS，还要搞一个Private Key File

## 安装ElasticSearch
我用的操作系统是Debian，所以安装ElasticSearch需要下列Command（https://www.elastic.co/guide/en/elasticsearch/reference/current/deb.html）

```bash
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo apt-key add -
sudo apt-get install apt-transport-https
echo "deb https://artifacts.elastic.co/packages/7.x/apt stable main" | sudo tee -a /etc/apt/sources.list.d/elastic-7.x.list
sudo apt-get update && sudo apt-get install elasticsearch
```

然后就是启动了。用`ps -p 1`查启动方式。我试了两次，第一次查出来是systemd，于是

```bash
sudo systemctl start elasticsearch.service # 启动
sudo systemctl stop elasticsearch.service # 关闭
sudo journalctl -f # Tail log
```

第二次竟然变了，变成init，于是

```bash
sudo update-rc.d elasticsearch defaults 95 10 # 设成自动启动
sudo -i service elasticsearch start # 启动
sudo -i service elasticsearch stop # 关闭
```

顺便说一下，我2014年也搞过一个ElasticSearch的个人项目，启动方式当年是类似`elasticsearch server start`。当时大版本号好像是3，现在都是7了。
这几年ElasticSearch变化快，用之前建议查一下变没变。

## ElasticSearch 网络设置
经过了各种找，发现ElasticSearch的Configuration在这里(https://www.elastic.co/guide/en/elasticsearch/reference/current/settings.html)
```
/etc/elasticsearch/elasticsearch.yml
```
然后要把network host改成GCP的Internal IP

## ElasticSearch内存设置
GCP的免费Tier只有0.6GB的内存（不给力啊老湿），而ElasticSearch的默认设置是1GB或者2GB，非常土豪。所以需要调一下JVM的设置。(https://www.elastic.co/guide/en/elasticsearch/reference/current/jvm-options.html)

```
/etc/elasticsearch/jvm.options
```

设置一下-Xmx和-Xms，都搞成300m，主要前面不要有空格。

## GCP的弃坑

GCP的免费版内存实在太小了，就算是把Heap Size变小，还是会让Instance直接死掉。我现在想先试试在Mac上搞一个Demo版本，以后估计真的要**众筹**买AWS、GCP、Heroku或者Bonsai了。

![crowd-funding](http://filmdope.com/wp-content/uploads/2009/11/oliver-twist01.jpg)

## Mac上安装ElasticSearch

经过GCP的折磨以后，本地Mac上安装就非常爽了：

```bash
brew install elasticsearch
brew services start elasticsearch # Background service running
elasticsearch # Test in shell
brew install kibana
brew services start kibana # Background service running
kibana # Test in shell
```

Elasticsearch默认port是9200，Kibana默认port是5601

Log和Config什么的可以在这儿找：https://www.elastic.co/guide/en/elasticsearch/reference/current/brew.html

## 史记
史记这个txt比较整齐，从第五行开始是正文（为了方便我直接把前四行手工删了），所有章节都以"●卷XXX第XXX"开始。

为了以后便于搜索，章节名称我做一下简化。忽略卷目和章节数，直接写章节名称。比如，"●卷一·五帝本纪第一"，简化为"五帝本纪"。

史记包括太史公自序一共130篇，不是很长。

需要注意的是，这个txt换行并不是一句话的终点。也就是说，虽然换行了，但这句话还没完。所以不能通过换行来做分界。
如果分段的话，我可以用中文的分段标记，也就是前面空两格。（也就是四个拉丁字母空格）

## 汉书
汉书末尾有一些关于汉书的介绍，还有目录，不应该做索引，所以删去。

汉书这个史料格式比较奇怪，它每一章是一行，然后用空格分行。

卷十一以后的章节名都是乱码，我手工给改过来了。

汉书里的表就不索引了，系文倒是可以索引，有兴趣的同学可以来这里查：https://ctext.org/han-shu/

班固把王莽放到那么靠后，估计是故意的吧。

发现“律历志第一下”里面空格乱加。我不熟悉“统母”、“五步”这类的术语，只能当做独立的一段了。

第一章这几个字“卷一上  高帝纪第一上”有一个找不到的特殊字符"﻿"，我不知道怎么把它搞掉。最简单的一个Hack就是加一个空行，然后把出来的json的第一个元素删掉。

章节还是把第XX上和第XX下合一下吧，不然可视性太差。

## 后汉书

后汉书这个史料注解太多了，虽然对理解史料有帮助，但对索引系统不友好。好在所有的注都是以“注【XXX】”开头。

我找到了另一个格式的后汉书电子版：http://www.gdwxmz.com/txt/shishu/houhanshu.html

虽然是章节顺序是乱的，但格式非常工整，每一行都是一段，也可以用正则表达式来找出章节名称。

每一个章节底下会重复一下章节名，

这个版本有一个问题，还是乱码，比如“贼帅常山人张燕，轻勇EC39捷”。“矫”字就成了乱码。我就先不一个一个改了，以后可以找一个更好的电子版。

这个电子版还夹杂广告，有八次“本电子书由《中国古典文学名著(www.gdwxmz.com)_古代小说下载_电子书txt在线阅读》（www.gdwxmz.com）整理并提供下载，本站只提供好看的小说txt电子书下载！”需要删掉。

关于内容，有这么个问题：《续汉书》八志（律历、礼仪、祭祀、天文、五行、郡国、百官、舆服）是后来司马彪续上的，所以章节名的格式就不一样了。为了兼容这个，我统一把格式变成“.*第.*”。

其余的就跟汉书的办法一样了。

跟汉书一样，章节里面有特殊字符。正则表达式不要用^和$

有些章节下面一两行有些重复，需要手工删掉。

## 三国志

三国志这个史料虽然有注解，但是是裴松之的，我个人认为裴松之的注史学价值非常高，不应该删掉。因此，所有注解的位置和内容都按原文的标准索引。

突然发现后面的史料一下子就没有注了，这不统一，应该找新的电子版。

找到了一个乱序版，虽然有乱码，但至少可以读。可惜没有裴松之的注了：http://www.gdwxmz.com/txt/shishu/sanguozhi.html

这个版本和上面后汉书那个格式简直一模一样，毕竟同一个源。这样工作就简单多了。

孙坚那一章标题错了：“孙破虏讨逆传弟一”，“弟”应为“第”。不要辜负文台兄。

## 晋书

晋书这个源乱码比较少。章节名称是两行，所以需要改一下代码。让人再次抓狂的是，第二行开始又出现了看不到的特殊字符，需要手工改。

需要把中文空格改成英文，不然不统一。

## 宋书

宋书和晋书一样，略。

## 南齐书

格式和宋书一样，略。

## 梁书

梁书竟然没有，需要这个资源：http://www.gdwxmz.com/txt/shishu/liangshu.html

这个资源处理方法和后汉书一样。

# 陈书

终于到了吕思勉先生的偶像陈武帝陈霸先了！耶！

格式和南齐书一样，略。

# 魏书

我发现http://www.gdwxmz.com 因为没有向国家备案，于是被阿里云关了服务器。摔！

找到了一个白话文的版本：https://www.bhzw.cc/info/9738.html，并没有卵用。

魏书的作者魏收字伯起，这个字我喜欢。

发现了另一个源：https://www.bhzw.cc/info/6103.html。这个是编码不是utf8，而且是繁体，需要转换一下。

没想到我有生之年真的会用到hao123：http://www.hao123.com/haoserver/jianfanzh.htm

有时候一章分成两行写，需要编辑一下。

卷四章节名少个个字，“帝纪第四 世祖纪下  宗纪”，应该是“帝纪第四 世祖纪下  恭宗纪  ”