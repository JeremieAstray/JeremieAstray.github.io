## 关于velocity shiro 的整合问题：velocity-tools2.0该如何配置

springmvc4.1.0,shiro1.2.3,velocity1.7,velocity-tools2.0  
velocityToolBox.xml的配置方式如下(与[shiro-velocity整合](https://github.com/eduosi/shiro-velocity-support)不一致，与网络上大多不一样，此配置参考了velocity-tools包里面的配置文件而来)：

```
<?xml version="1.0" encoding="UTF-8"?>
<tools>
    <toolbox scope="application">
        <tool class="com.jeremie.commons.shiro.support.Permission"/>
    </toolbox>
</tools>
```

如果使用原配置方式，会使velocity-tools无法正常加载此类

另附[Permission.java](https://github.com/eduosi/shiro-velocity-support/blob/master/src/main/java/org/apache/shiro/web/support/velocity/Permission.java)

springmvc-servlet中velocity的配置

```
    <bean class="org.springframework.web.servlet.view.velocity.VelocityViewResolver" p:order="2">
           <property name="toolboxConfigLocation" value="/WEB-INF/velocityToolBox.xml"/>
           <property name="viewClass" value="com.jeremie.commons.velocity.MyVelocityToolboxView" />
           <property name="cache" value="true" />
           <property name="prefix" value="" />
           <property name="suffix" value=".vm" />
           <property name="exposeSpringMacroHelpers" value="true" /><!--是否使用spring对宏定义的支持-->
           <property name="exposeRequestAttributes" value="true" /><!--是否开放request属性-->
           <!--<property name="requestContextAttribute" value="rc"/><!–request属性引用名称–>-->
           <property name="contentType" value="text/html;charset=UTF-8"/>
    </bean>
    <bean id="velocityConfigurer" class="org.springframework.web.servlet.view.velocity.VelocityConfigurer">
           <property name="resourceLoaderPath" value="/" />
           <property name="velocityProperties">
               <props>
                   <prop key="input.encoding">utf-8</prop>
                   <prop key="output.encoding">utf-8</prop>
               </props>
           </property>
    </bean>
```

MyVelocityToolboxView.java

```
public class MyVelocityToolboxView extends VelocityToolboxView {

    @Override
    protected Context createVelocityContext(Map model, HttpServletRequest request, HttpServletResponse response) {
        ViewToolContext ctx;

        ctx = new ViewToolContext(getVelocityEngine(), request, response, getServletContext());

        ctx.putAll(model);

        if (this.getToolboxConfigLocation() != null) {
            ToolManager tm = new ToolManager();
            tm.setVelocityEngine(getVelocityEngine());
            tm.configure(getServletContext().getRealPath(getToolboxConfigLocation()));
            if (tm.getToolboxFactory().hasTools(Scope.REQUEST)) {
                ctx.addToolbox(tm.getToolboxFactory().createToolbox(Scope.REQUEST));
            }
            if (tm.getToolboxFactory().hasTools(Scope.APPLICATION)) {
                ctx.addToolbox(tm.getToolboxFactory().createToolbox(Scope.APPLICATION));
            }
            if (tm.getToolboxFactory().hasTools(Scope.SESSION)) {
                ctx.addToolbox(tm.getToolboxFactory().createToolbox(Scope.SESSION));
            }
        }
        return ctx;
    }

}
```