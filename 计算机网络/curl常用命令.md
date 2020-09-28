常规用法：*http请求地址的url要使用""括起来。当有存在多个参数使用&连接时可能会出错。*

---

- 获取请求头信息: curl -i url
- 显示请求的完整信息:curl -v url
- 跟踪url： curl -L url
- 下载资源: curl -O url 
- get请求: curl url
- post请求: curl -X Post 
- 上传文件: curl -F file=@filepath url
  常见参数:
- -d "name=name&pwd=pwd" 请求参数，默认为post
- -H "Content-Type: application/json" 
- H "Content-Type: multipart/form-data"
- -b "key1=val1;key2=val2"  发送cookie