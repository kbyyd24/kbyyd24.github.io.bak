---
title: SpringMVC中Controller的测试
date: 2017-06-04
updated: 2017-06-04
description: 记录如何在 SpringMVC 中测试接口
---

> 参考 [http://www.jianshu.com/p/ad7995332dd9](http://www.jianshu.com/p/ad7995332dd9)

<!-- more -->

controller:

```java
@Controller
@RequestMapping("/system")
public class SysMapController {

    @Autowired
    private FirstSysMapService firstSysMapService;//注入Service

    @RequestMapping(value = "/first/map",
            method = RequestMethod.GET,
            produces = {"application/json;charset=UTF-8"})
    @ResponseBody//响应类型为`json`
    public List<FirstSysMap> getMap() {
        List<FirstSysMap> map = firstSysMapService.findMap();
        return map;
    }
}
```

test:

```java
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration({"classpath:spring/spring-dao.xml",
        "classpath:spring/spring-service.xml",
        "classpath:spring/spring-web.xml"})
@WebAppConfiguration//没搞懂，应该是注入`web`上下文的意思
public class SysMapControllerTest {

    @Autowired
    private SysMapController mapController;

    @Autowired
    private ServletContext context;

    private MockMvc mockMvc;//`springMVC`提供的`Controller`测试类，不需要再去单独依赖`mockito`

    @Before
    public void setup() {
        //应该是把`Controller`加入测试环境的意思
        mockMvc = MockMvcBuilders.standaloneSetup(mapController).build();
    }

    @Test
    public void getMap() throws Exception {
        //发起请求，`accpet`设定接收的数据类型
        ResultActions actions = mockMvc.perform(MockMvcRequestBuilders.get("/system/first/map")
                .accept(MediaType.APPLICATION_JSON_UTF8)
        );
        //获取`response`
        MvcResult result = actions.andReturn();

        String res = result.getResponse().getContentAsString();

        System.out.println("-----------response:" + res);
    }

}
```
