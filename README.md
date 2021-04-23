# Exploit Title:
SQL injection vulnerability exists in Mybatis-Plus

# Description:
MyBatis-Plus is an powerful enhanced toolkit of MyBatis for simplify development. This toolkit provides some efficient, useful, out-of-the-box features for MyBatis, use it can effectively save your development time. It is very popular among Chinese software developers, Github Star 10.9k.

# Date:
2021-04-22

# Exploit Author:
kw0ng

# Vendor Homepage: 
https://baomidou.com/
https://mybatis.plus/
https://github.com/baomidou/mybatis-plus

# Software Link: 
https://github.com/baomidou/mybatis-plus/releases

# Version: 
All versions

# Tested on:
Mac OS 11.2.3
Windows 10

# POC:
?ascs=extractvalue(1,concat(char(126),md5(123)))&ascs=1

Use the paging Demo code officially provided by Mybatis-Plus, and the database uses Mysql, The Demo code can be seen in this project, use Idea to open it to run.
The paging principle of Mybatis-Plus is to intercept the SQL generated by Mybatis, splicing ORDER BY / LIMIT and other statements, and these statements are derived from the Page entity passed in by the user, if you use the paging controller, you must substitute the Page entity. 

1. Access Demo code selectPage interface

http://127.0.0.1:8081/user/selectPage

![image](https://user-images.githubusercontent.com/40931609/115806523-80faf480-a419-11eb-908e-035d5a9c3603.png)

2. Use the sql injection payload

http://127.0.0.1:8081/user/selectPage?ascs=extractvalue(1,concat(char(126),md5(123)))&ascs=1

![image](https://user-images.githubusercontent.com/40931609/115806581-9f60f000-a419-11eb-81ee-1d6c85c3e3d6.png)

3. debugging 

com/baomidou/mybatisplus/extension/plugins/pagination/Page.java
line 255
![image](https://user-images.githubusercontent.com/40931609/115806724-e222c800-a419-11eb-844c-54c0ba8b5434.png)

The ascs is a List<String> parameter,if you only send one ascs parameter, you can see that springMVC will divide the ascs parameter into 3 parts with commas, which will cause the subsequent SQL splicing statement syntax error:

![image](https://user-images.githubusercontent.com/40931609/115806838-17c7b100-a41a-11eb-84ce-aa5539812647.png)

But, If we use two ascs parameters, Ascs parameters will not be split :

![image](https://user-images.githubusercontent.com/40931609/115806902-2f9f3500-a41a-11eb-93c3-6f0643626e64.png)

View Page Pagination Blocker:

com/baomidou/mybatisplus/extension/plugins/PaginationInterceptor.java

line 127:
![image](https://user-images.githubusercontent.com/40931609/115807179-accaaa00-a41a-11eb-9bd9-d127bec25c28.png)

This interceptor will intercept the SQL statement generated by Mybatis and splice the ascs parameters we passed in to ORDER BY to generate the final SQL statement.

Main vulnerability code:

```
com/baomidou/mybatisplus/extension/plugins/PaginationInterceptor.java
line 126:
plainSelect.setOrderByElements(orderByElementsReturn);
line 127:
return plainSelect.toString();
```

orderByElementsReturn is the ascs array we passed in, plainSelect is the original sql statement in mybatis, here first set orderByElementsReturn to the properties of plainSelect, and then rewrite the toString method.

Add the ascs to the ORDER BY string and append to the original sql:
jsqlparser-4.0.jar!/net/sf/jsqlparser/statement/select/PlainSelect.class
line 387:
![image](https://user-images.githubusercontent.com/40931609/115807705-b7397380-a41b-11eb-8cea-119260abde96.png)

orderByToString:
jsqlparser-4.0.jar!/net/sf/jsqlparser/statement/select/PlainSelect.class
line 437:
![image](https://user-images.githubusercontent.com/40931609/115807804-df28d700-a41b-11eb-8c11-eafd3dffd4ef.png)

