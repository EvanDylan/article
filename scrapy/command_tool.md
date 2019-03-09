## 命令行工具

### 命令总览

运行`scrapy --help`

```
Scrapy 1.5.1 - project: tutorial

Usage:
  scrapy <command> [options] [args]

Available commands:
  bench         Run quick benchmark test
  check         Check spider contracts
  crawl         Run a spider
  edit          Edit spider
  fetch         Fetch a URL using the Scrapy downloader
  genspider     Generate new spider using pre-defined templates
  list          List available spiders
  parse         Parse URL (using its spider) and print the results
  runspider     Run a self-contained spider (without creating a project)
  settings      Get settings values
  shell         Interactive scraping console
  startproject  Create new project
  version       Print Scrapy version
  view          Open URL in browser, as seen by Scrapy

Use "scrapy <command> -h" to see more info about a command
```



### startproject命令

用于创建新的项目，命令行语法如下`scrapy startproject <project_name> [project_dir]`

### genspider命令

在已有的项目中创建一个新的爬虫，命令行语法如下

```
Usage
=====
  scrapy genspider [options] <name> <domain>

Generate new spider using pre-defined templates

Options
=======
--help, -h              show this help message and exit
--list, -l              List available templates
--edit, -e              Edit spider after creating it
--dump=TEMPLATE, -d TEMPLATE
                        Dump template to standard output
--template=TEMPLATE, -t TEMPLATE
                        Uses a custom template.
--force                 If the spider already exists, overwrite it with the
                        template
```

运行`scrapy genspider --list`查看已有的模板有哪些：

- basic
- crawl
- csvfeed
- xmlfeed

使用模板创建爬虫类命令如下：`scrapy genspider--template=crawl spider http://www.baidu.com`

### crawl命令

用于运行已存在的爬虫，命令格式如下：

```
Usage
=====
  scrapy crawl [options] <spider>

Run a spider

Options
=======
--help, -h              show this help message and exit
-a NAME=VALUE           set spider argument (may be repeated)
--output=FILE, -o FILE  dump scraped items into FILE (use - for stdout)
--output-format=FORMAT, -t FORMAT
                        format to use for dumping items with -o
```

### check命令

用于检查爬虫语法错误

```
Usage
=====
  scrapy check [options] <spider>

Check spider contracts

Options
=======
--help, -h              show this help message and exit
--list, -l              only list contracts, without checking them
--verbose, -v           print contract tests for all spiders
```

### view命令

通过命令行将指定URL网页下载下来，并在浏览器中打开。

```
Usage
=====
  scrapy view [options] <url>

Fetch a URL using the Scrapy downloader and show its contents in a browser

Options
=======
--help, -h              show this help message and exit
--spider=SPIDER         use this spider
--no-redirect           do not handle HTTP 3xx status codes and print response
```

运行`scrapy view http://www.taobao.com`，下载完成后浏览器自动展现了下载了的淘宝首页。