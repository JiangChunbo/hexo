---
title: Spring 请求参数处理实践
date: 2022-08-17 15:37:26
tags:
---


场景描述：请求 Content Type 为 `application/x-www-form-urlencoded`，即参数以键值对形式传递，其中有简单类型，如：数字，字符串，也有 JSON 类型，需要绑定到 Spring 的模型中，完成校验。

参数格式参考如下，为了便于显示，进行了换行: 

```bash
lesson_date=2022-08-08
&lesson_period_id=1
&subject_id=1
&grade_id=1
&class_id=1
&teacher_id=1
&student_list=[{"x":0,"y":0,"speak_num":0,"distract_num":0,"is_special":0}]
```


如何支持下划线传参映射到 Model 模型? 装饰器模式 Request

```java
public class MultiParamCaseRequest extends HttpServletRequestWrapper {

    private final Map<String, String[]> additionalParams;

    public MultiParamCaseRequest(HttpServletRequest request, Map<String, String[]> additionalParams) {
        super(request);
        this.additionalParams = additionalParams;
    }

    @Override
    public Enumeration<String> getParameterNames() {
        final HashSet<String> parameterNames = new HashSet<>();
        parameterNames.addAll(getRequest().getParameterMap().keySet());
        parameterNames.addAll(additionalParams.keySet());
        return Collections.enumeration(parameterNames);
    }

    @Override
    public String getParameter(String name) {
        final String parameter = super.getParameter(name);
        if (parameter != null) {
            return parameter;
        }
        final String[] parameterValues = additionalParams.get(name);
        if (parameterValues != null) {
            if (parameterValues.length == 0) {
                return "";
            }
            return parameterValues[0];
        }
        return null;
    }

    @Override
    public String[] getParameterValues(String name) {
        final String[] parameterValues = super.getParameterValues(name);
        if (parameterValues != null) {
            return parameterValues;
        }
        return additionalParams.get(name);
    }
}
```

```java
@Component
public class ParamNameExtensionFilter extends HttpFilter {

    private final Pattern underLinePattern = Pattern.compile("_(\\w)");

    @Override
    protected void doFilter(HttpServletRequest request, HttpServletResponse response, FilterChain chain) throws IOException, ServletException {
        Map<String, String[]> additionalParams = new HashMap<>();
        final Enumeration<String> parameterNames = request.getParameterNames();
        final Map<String, String[]> parameterMap = request.getParameterMap();
        while (parameterNames.hasMoreElements()) {
            final String parameterName = parameterNames.nextElement();
            final String camelCaseParameterName = this.underLineToCamel(parameterName);
            if (parameterName.equals(camelCaseParameterName)) {
                continue;
            }
            final String[] values = parameterMap.get(parameterName);
            additionalParams.put(camelCaseParameterName, values);
        }
        final MultiParamCaseRequest multiParamCaseRequest = new MultiParamCaseRequest(request, additionalParams);
        chain.doFilter(multiParamCaseRequest, response);
    }

    private String underLineToCamel(final String value) {
        final StringBuffer sb = new StringBuffer();
        Matcher m = this.underLinePattern.matcher(value);
        while (m.find()) {
            m.appendReplacement(sb, m.group(1).toUpperCase());
        }
        m.appendTail(sb);
        return sb.toString();
    }
}
```

如果 Model 对象包含 `Collection`，需要定制 DataBinder:

```java
    @InitBinder
    public void initBinder(WebDataBinder dataBinder) {
        dataBinder.registerCustomEditor(List.class, "studentList", new PropertyEditorSupport() {
            @Override
            public void setAsText(String text) throws IllegalArgumentException {
                if (StringUtils.isEmpty(text)) {
                    setValue(Collections.EMPTY_LIST);
                } else {
                    final ObjectMapper objectMapper = new ObjectMapper();
                    try {
                        setValue(objectMapper.readValue(text, new TypeReference<List<EvaluationVO.Student>>() {
                        }));
                    } catch (JsonProcessingException e) {
                        throw new RuntimeException(e);
                    }
                }
            }
        });
    }
```