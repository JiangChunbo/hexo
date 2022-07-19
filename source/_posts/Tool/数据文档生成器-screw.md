---
title: 数据文档生成器 screw
date: 2022-07-18 08:59:38
categories:
- 工具
tags:
- screw
---
# 数据文档生成器 screw

## 相关链接

https://github.com/yihr/screw

[参考文章](https://mp.weixin.qq.com/s?__biz=MzU0OTE4MzYzMw==&mid=2247490367&idx=4&sn=33457d828191fc8717bf90e29d02aa5b&chksm=fbb292c1ccc51bd776a938e2a0aca306363c8ea526362c83a1ed0e7d2522d136a5482f529481&mpshare=1&scene=1&srcid=08075ho06D4MoWdvYI6HZGn2&sharer_sharetime=1596771975108&sharer_shareid=00de337ccf971170dff621a18a7fdda8&key=bbcde1cc2908d6bbca75f9126db113cba3e65212579f0c49d238ed98e48a07667ecea3754647040875027233e7254e03354cbcb58a82ff9c1b4f865e3e510b1759f8f013ff094835f46a0f3809a473f4&ascene=1&uin=MjE2Mjg4NzYz&devicetype=Windows+10+x64&version=62090529&lang=zh_CN&exportkey=A98TRXcX38ZaCSWR2WlBPus%3D&pass_ticket=%2Fq1C0cKlfjbdKpTIM9MtXaZTfIIIRMMDAPgn%2FJ8FuXo%3D)



## maven 依赖

```xml
<dependency>
	<groupId>cn.smallbun.screw</groupId>
	<artifactId>screw-core</artifactId>
	<version>1.0.3</version>
</dependency>
```

## JUnit 生成

```java
@SpringBootTest
public class DBDocument {

    @Autowired
    DataSource dataSource;

    @Test
    public void contextLoads() {
        // 生成文件配置
        EngineConfig engineConfig = EngineConfig.builder()
            /* 生成文件路径，本地路径 */
            .fileOutputDir("D:/")
            /* 是否打开输出的目录 */
            .openOutputDir(true)
            /* 文件类型 */
            .fileType(EngineFileType.HTML)
            /* 生成模板实现 */
            .produceType(EngineTemplateType.freemarker)
            .build();
        // 生成文档配置（包含以下自定义版本号、描述等配置连接）
        Configuration config = Configuration.builder()
            .version("1.0.0")
            .description("数据库文档")
            .dataSource(dataSource)
            .engineConfig(engineConfig)
            .produceConfig(getProcessConfig())
            .build();
        // 执行生成
        new DocumentationExecute(config).execute();
    }

    /**
     * 配置想要生成的表 + 配置想要忽略的表
     * 
     * @return
     */
    public static ProcessConfig getProcessConfig() {
        return ProcessConfig.builder()
            .build();
    }
}
```