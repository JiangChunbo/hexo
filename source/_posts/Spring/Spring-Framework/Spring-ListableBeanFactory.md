---
title: Spring ListableBeanFactory
date: 2022-10-28 09:00:32
tags:
---




```java
<T> Map<String, T> getBeansOfType(Class<T> type, boolean includeNonSingletons, boolean allowEagerInit)
        throws BeansException;
```


策略设计模式的使用: 

目前你需要实现一些邮件正文的生成，令他们由不同的业务类去实现，这些业务类提供一个统一的接口，比如: `EmailBodyProvider`。这些 `EmailBodyProvider` 必须提供一个标识符以供查询。

```java
public interface EmailBodyProvider {

    Long getId();

    String getEmailBody(Long applyId);

    /**
     * 根据操作类型获取具体实现类
     *
     * @param formId 表单ID
     * @return IJsbAssetEmailApplyInfoQueryBiz
     */
    static EmailBodyProvider getInstance(Long id) {
        Map<String, EmailBodyProvider> map = ContextUtil.getContext().getBeansOfType(EmailBodyProvider.class, false, false);
        for (Map.Entry<String, EmailBodyProvider> entry : map.entrySet()) {
            if (Objects.equals(formId, entry.getValue().getId())) {
                return entry.getValue();
            }
        }
        throw new IllegalArgumentException("can not find IJsbAssetEmailApplyInfoQueryBiz implement instance");
    }

}
```

假如现在有一些实现类: `Business1EmailBodyProvider`、`Business2EmailBodyProvider`