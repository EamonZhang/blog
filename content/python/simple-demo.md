---
title: "python 简单示例"
date: 2021-12-03T13:36:56+08:00
draft: false
toc: false
categories: ['python']
tags: []
---

## 快速启动一个本地服务

可临时充当文件下载服务
```
# python2
python -m SimpleHTTPServer 8888

# python3
python3 -m http.server 8888
```

## flask 服务

```
from flask import Flask

app = Flask(__name__)

@app.route("/")
def hello_world():
    result = find_result() 
    return result

@app.route('/login', methods=['GET', 'POST'])
def login():
    if request.method == 'POST':
        pass 
    else:
        pass

if __name__ == '__main__':
    app.run(host='0.0.0.0',port='8000')
```

`python app.py`

`gunicorn -w 1 -b 127.0.0.1:4000 app:app`

## 操作PG数据库
```
import psycopg2
import time

conn = None
try:
  conn = psycopg2.connect(database="postgres", user="postgres", host="192.168.*.*", port="5432",password='*****')
  print("Opened database successfully")
  cur = conn.cursor()
  cur.execute("select * from api")
  rows = cur.fetchall()
  result = ''
  all_field = cur.description
  for filed in all_field:
     print(filed[0])
  for row in rows:
     result += str(row[0])
     result += "\n"
  print(result)
  print("Operation done successfully")
  conn.close()
except Exception as e:
   print("ERROR ： ")
   print(e)
finally:
   print("close connction")
   conn.close()
```
`pip install psycopg2-binary==2.8.5`


