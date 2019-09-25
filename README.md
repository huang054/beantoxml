# xml与javabean互转

在群里有个小伙伴询问json与xml互转怎么实现，我以为有工具类，叫他去搜，他说没有，我也搜了下，发现网上的demo都不靠谱，于是想自己看看有没上面办法解决，想到springmvc有个返回xml格式数据，心想看能不解决

## springmvc结果转换

demo

```java
@PostMapping("/user")
@ResponseBody
public User userValidation(@Valid @RequestBody User user){
    System.out.println(service1);

    return user;
}
```

```java
@XmlRootElement
public class User {

    @Max(1000)
    private long id;

    @NotNull
    private String name;

    @MyCardsValidation
    private String carsNo;

    public long getId() {
        return id;
    }

    public void setId(long id) {
        this.id = id;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public String getCarsNo() {
        return carsNo;
    }

    public void setCarsNo(String carsNo) {
        this.carsNo = carsNo;
    }
}
```

在请求头里设置Accept ：application/xml 

就返回



```xml
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>

<user>

    <carsNo>huang-123</carsNo>

    <id>124</id>

    <name>lilei</name>

</user>

```


这里说下http请求的请求头metatype，spring会根据其格式接受返回相应格式数据

## 跟踪源码

```java
@Override
protected void writeToResult(Object o, HttpHeaders headers, Result result) throws Exception {
   try {
      Class<?> clazz = ClassUtils.getUserClass(o);
      Marshaller marshaller = createMarshaller(clazz);
      setCharset(headers.getContentType(), marshaller);
      marshaller.marshal(o, result);
   }
   catch (MarshalException ex) {
      throw ex;
   }
   catch (JAXBException ex) {
      throw new HttpMessageConversionException("Invalid JAXB setup: " + ex.getMessage(), ex);
   }
}
```

发现用得是Marshaller实现的，这个Marshaller是什么东西？发现是jdk的类，其实就是用jdk的实现

，但是有个缺陷，不能返回集合，于是我用一个公共类包装一个集合对象就可以了

## 自定义实现

```java
package com.netty;

import org.apache.xpath.objects.XNull;

import javax.xml.bind.annotation.XmlElement;
import javax.xml.bind.annotation.XmlElementWrapper;
import javax.xml.bind.annotation.XmlRootElement;
import java.util.List;

/**
 * @author huangjun
 * @version V1.0
 * @Description: TODO
 * @Date Create in 10:10 2019/09/25
 */
@XmlRootElement(name = "body")
public class Body {
   // private ReqHeader reqHeader;
    private List<User> smsBodys;


//    @XmlElement(name = "REQHEADER")
//    public ReqHeader getReqHeader() {
//        return reqHeader;
//    }
//
//    public void setReqHeader(ReqHeader reqHeader) {
//        this.reqHeader = reqHeader;
//    }

    /**
     * 在JAXB标准中，@XmlElementWrapper注解表示生成一个包装 XML 表示形式的包装器元素。
     * 此元素主要用于生成一个包装集合的包装器 XML 元素。因此，该注释支持两种形式的序列化。
     * @XmlElementWrapper 仅允许出现在集合属性上
     * @return
     */
    @XmlElementWrapper(name = "users")
    @XmlElement(name = "user")
    public List<User> getSmsBodys() {
        return smsBodys;
    }

    public void setSmsBodys(List<User> smsBodys) {
        this.smsBodys = smsBodys;
    }

}
```

```java
package com.netty;

/**
 * @author huangjun
 * @version V1.0
 * @Description: TODO
 * @Date Create in 15:55 2019/09/24
 */


import javax.xml.bind.JAXBContext;
import javax.xml.bind.JAXBException;
import javax.xml.bind.Marshaller;
import javax.xml.bind.Unmarshaller;
import java.io.FileNotFoundException;
import java.io.StringReader;
import java.io.StringWriter;
import java.util.ArrayList;
import java.util.Arrays;
import java.util.List;

public class JsonToXml {

    public static <T> T xmlToBean(String xml, T t) throws JAXBException {
        JAXBContext context = JAXBContext.newInstance(t.getClass());
        Unmarshaller um = context.createUnmarshaller();
        StringReader sr = new StringReader(xml);
        t = (T) um.unmarshal(sr);
        return t;
    }

    public static <T> StringWriter beanToXml(T t) throws JAXBException, FileNotFoundException {
        JAXBContext context = JAXBContext.newInstance(t.getClass());
        Marshaller m = context.createMarshaller();
        StringWriter sw = new StringWriter();
        m.marshal(t, sw);
        m.setProperty(Marshaller.JAXB_FORMATTED_OUTPUT, true);
       // m.marshal(t, new FileOutputStream("/test.xml"));
       //m.marshal(t, System.out);
        return sw;
    }

    public static void main(String[] args) throws JAXBException, FileNotFoundException {
        List<User> users = new ArrayList<>();
       User user = new User();
       user.setId(1);
       user.setName("lilei");
        User user1 = new User();
        user1.setId(2);
        user1.setName("lilei1");
        users.addAll(Arrays.asList(user,user1));
        Body body = new Body();
        body.setSmsBodys(users);
       System.out.println(beanToXml(body));
    }
}
```

其实有时候不是spring非常牛比，而是spring开发者对jdk太熟悉了，而我们却对jdk不知道的太多