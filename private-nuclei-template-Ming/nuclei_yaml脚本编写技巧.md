# nuclei yaml脚本编写技巧

## nuclei yaml脚本简介

本文主要是面向第一次编写nuclei yaml脚本的人。nuclei yaml主要分为两部分，第一部分为漏洞信息介绍，第二部分为漏洞检测。完整的脚本编写文档请查阅[官方文档](https://docs.nuclei.sh/template-guide/introduction)

## nuclei yaml脚本第一部分

首先来看看第一部分，第一部分主要包含了漏洞信息：

```yml
id: secgate_3600_uploadfile # 漏洞在nuclei中的显示名称
info:
  name: SecGate_3600_uploadfile # 漏洞实际名称
  author: joyboy # 作者
  severity: critical # 严重程度(critical/high/medium/low/info)
  description: http://xxx.xxx.xxx/?g=obj_app_upfil # 漏洞描述
  reference:
    - https://labs.watchtowr.com/yet-more-unauth-remote-command-execution-vulns-in-firewalls-sangfor-edition/ # 漏洞来源
  metadata: # 元数据
    max-request: 1 # 最大请求数(基本无用，可不写)
    fofa: title="SecGate 3600",fid="1Lh1LHi6yfkhiO83I59AYg==" # 在网络空间搜索引擎中的指纹，支持shodan\censys\fofa\hunter\quake等，可以配合uncover使用，同一搜索引擎下的多个指纹使用","隔开，
    verified: true # 是否验证
  tags: upload,qax # 描述漏洞类型以及其他关键词
```

### 注意事项

1. 所有的nuclei yaml脚本文件必须包含第一部分
2. `id`中不能包含空格，务必使用下划线连接，所有字符推荐使用英文小写
3. 在标注危险程度`severity`时，请务必按照危害来定级，一般情况下可以shell的漏洞必须给`critical`，可以获取敏感文件的漏洞给`high`，可以获得敏感数据的漏洞给`medium`，可以获取普通数据的漏洞给`low`，指纹识别给`info`
4. `description`尽量简单描述漏洞信息，可不写
5. `reference`可以给出POC、EXP的来源或者漏洞通报文章的地址，可不写
6. `metadata`中必须至少给出一个网络空间搜索指纹
7. `tags`类型必须包含以下的一种类型：`fileread`、`rce`、`sqli`、`unauthorized`、`upload`，`weakpasswd`，`xss`

## nuclei yaml脚本第二部分

目前nuclei有两种方式针对web漏洞的检测流程，一种是使用`base`格式，一种是使用`raw`格式，`base`格式只需填写漏洞关键利用部分，包内的其余数据为nuclei随机填写，`raw`格式是讲漏洞数据包直接通过nuclei发送，不经过nuclei处理。一般情况下，`base`格式适合简单的漏洞利用，`raw`适合复杂漏洞利用

### `base`格式yaml脚本

首先来看看`base`格式的yaml脚本

```yml
http: # 表示http请求，在web漏洞中，该值不可变
    - method: POST # 请求类型，支持常见的请求方式(GET/POST/PUT/DELETE)等
      path: 
        - "{{RootURL}}/bic/ssoService/v1/applyCT" # 请求地址，其中{{RootURL}}下文讲解
      headers: # 如果包含必须的请求头，可使用该参数指定请求头，内容为必须的请求头，无需编码
        Testcmd: ipconfig
        Content-Type: application/json
      body: '略' # 如果该请求包含body，可在此填写，如POST和PUT请求中的参数
      redirects: true # 如果该数据包发送后会被重定向，请包含此参数，否则可删去
      max-redirects: 3 # 如果需要重定向，可以使用此参数控制最大重定向次数，默认为10，否则可删去
      # 下面为漏洞匹配，如果数据包发送成功，可使用下面的内容匹配是否攻击成功，成功的可显示到nuclei
      matchers-condition: and # 如果包含多个匹配项，可使用该参数，默认为or
      matchers:
        - type: word # 匹配类型，支持字符匹配(word)，返回状态匹配(status)，超本文匹配(xpath)，正则匹配(regex)，复杂类型匹配(dsl)等
          words:
            - Windows IP
        - type: status
          status:
            - 200
        - type: xpath # 匹配xml和html中的内容
          xpath:
            - "/html/head/title[contains(text(), 'Example Domain')]"
        - type: regex # 使用正则匹配返回包中的内容
          regex:
            - "root:[x*]:0:0"
        - type: dsl # 复杂类型匹配器可以使用逻辑关系进行多次匹配
          dsl:
            - "len(body)<1024 && status_code==200"
            - "status_code_1 == 404 && status_code_2 == 200 && contains((body_2), 'secret_string')" # 如果包含多个请求包，可以接下划线和数字来区分
```
