---
title: 利用NGINX反向代理避免跨域
date: 2016-06-07
updated: 2016-06-07
---

在慕课网上看了高并发的课程，准备用`spring`+`Mybaits`来开发新的项目。遇到了前端跨域请求的问题。

服务器上`nginx`+`tomcat`，其中`nginx`监听`80`端口，`tomcat`监听`8080`端口。

因为对前端不熟悉，以为用`ajax`就可以不需要`callback`，然而前端的同学说不跨域的情况下才不需要`callback`，让我在返回的`json`里加上。可是我刚刚学会了最基本的`spring-mvc`用法，根本不知道怎么加上`callback` :joy: 

网上到时找到一些可行的代码，差不多这个样子：

来源：http://quarterlifeforjava.iteye.com/blog/2218530

```java
@RequestMapping(method=RequestMethod.GET,value="getProjectStatusList",produces="text/html;charset=UTF-8")
@ResponseBody
public String getProjectStatusList(HttpServletRequest request, 
					 HttpServletResponse response){
	
	
	Map<String,Object> map = new HashMap<String,Object>();
	try{
		String callback = request.getParameter("callback");
		//System.out.println("token:"+request.getHeader("token"));
		List<String> list = ss.getProjectStatusList();
		map.put("status", "success");
		map.put("data", list);
		ObjectMapper mapper = new ObjectMapper();
		//这个拼接是重点。。。
	        String result = callback+"("+mapper.writeValueAsString(map)+")";
		//String result = mapper.writeValueAsString(map);
		return result;
	}catch(Exception e){
		JSONObject jo = new JSONObject();
		jo.put("status", "fail");
		jo.put("data", e.getMessage());
		return jo.toString();
	}
}
```

然而这样改动对我来说简直是伤筋动骨，因为我有太多的`URL`映射，修改的成本太大。

所以机智的我想到了`nginx`，这家伙不就是拿来搞反向代理的吗？真是机智如我 :sunglasses:

有了这个思路，做起来就简单了。直接在监听`80`端口的`server`中添加一个`location`：

```conf
location /myApp {
  proxy_pass  http://localhost:8080/myApp;
}
```

重新加载`nginx`：

```shell
{NGINX_HOME}/sbin/nginx -s reload
```

然后就把之前`http://site:8080/myApp`的跨域请求变成了`http://site/myApp`的非跨域请求。

---

好吧，都是我猜的，等前端同学来验证我的想法了 :smile: 

---

下午实验课自己写了个`ajax`请求试了下，这个思路是没有问题的

```javascript
$.ajax({url : "/myApp/list", async : true}).success( function (data) {
  console.log(data);
});
```

打印结果：

```json
Object {data: Array[8], success: true, errorMsg: null}
```
