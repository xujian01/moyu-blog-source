---
title: GitLab的Webhook配置和开发
date: 2020-10-10 10:33:29
tags: GitLab,WebHook
---
> 本文主要介绍如何使用gitlab的webhook来打通企业微信消息提醒。

## 前提准备
### 企业微信消息发送接口
根据企业微信开发者文档得到一个消息发送的接口url，参照：[企业微信群机器人配置说明](https://work.weixin.qq.com/api/doc/90000/90136/91770?st=1827E5EC947C81BBE563C4273E6222968B82B70397A45ABAB41E979B16DB5221F65A83AB012969397CE2E51EF7A8997AE3CAF92F6473A0C5384DA17EB2C41D5055835355B3A37373AB6A5EA28D9743396E44D225D674E85D803E33055AC20872593657E4998C36DCBC6268A8B1DA1DA5F7BB21628385E8B9B9BDE5ED2B4BF206D79ACE6F5791B721D0C94CEABA281EA5&vid=1688850402739477&cst=88A970BD24E6CBCD6844008FFC70E464C4A86B63784323A03A26A74FAA3F8509FFF6A079F601C9567A620B81CA87647E&deviceid=2027bf5d-2885-4f64-880a-c241c5b30782&version=3.0.36.2330&platform=mac)；
### gitlab（账号，用户组，项目）
* 生成gitlab账号token
![在这里插入图片描述](https://img-blog.csdnimg.cn/20201207154748852.png)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20201207154821440.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzE4NTE1MTU1,size_16,color_FFFFFF,t_70)

* 获取项目的project_id
参考[gitlab如何查询项目ID](https://git.jarry.com/help/api/api_resources.md)
* 获取用户组的group_id
方法类似于上面project_id的获取

### gitlab开放API文档
[开放API文档](https://git.jarry.com/help/api/api_resources.md)
![在这里插入图片描述](https://img-blog.csdnimg.cn/2020120715484219.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzE4NTE1MTU1,size_16,color_FFFFFF,t_70)
## webhook配置和开发
### 配置webhook
![在这里插入图片描述](https://img-blog.csdnimg.cn/20201207154858922.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzE4NTE1MTU1,size_16,color_FFFFFF,t_70)
> `Secret Token`和`Enable SSL verification`配置项可以先不配置。

在这里配置wenhook，我这里先配置两个触发事件，`Tag push events`（tag新增/删除事件）和`Merge request events`（MR新增/删除事件）。
### gitlab的webhook原理
上面的配置中有一个URL配置项还没有配置。
想知道这里应该配什么，首先应该了解gitlab的webhook工作原理。
> 这里还是以发送通知到企业微信为例。

![在这里插入图片描述](https://img-blog.csdnimg.cn/2020120715501273.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzE4NTE1MTU1,size_16,color_FFFFFF,t_70)
1. 项目代码变动往gitlab上推送相应的事件，例如代码push，新建tag，创建merge request等等；
2. gitlab收到相应事件，触发对应的webhook，设置HTTP请求的header以及request body，然后发送HTTP请求到配置的webhook的URL；
3. HTTP请求到达对应的处理服务器以后，对request body和header进行解析，包装通知内容；
4. 将通知的内容通过企业微信的消息发送接口发送到企业微信；
具体参考[webhook使用指南](https://git.jarry.com/help/user/project/integrations/webhooks)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20201207154955502.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzE4NTE1MTU1,size_16,color_FFFFFF,t_70)
**接下来所有的重点就是这个URL是什么？他应该是一个接口，用来处理gitlab的事件。**
## 项目实战
前提准备做好之后，就可以开发处理事件的服务端了（基于SpringBoot项目）。
以下是一些核心代码。
**`GitLabApiUtils.java`** 
```java
/**
     * 获取所以项目master成员
     * @param projectId
     * @return
     */
    public static List<String> getAllProjectMembers(Integer projectId) {
        getProjectMembersUrl = getProjectMembersUrl.replace("$",""+projectId);
        getGroupMembersUrl = getGroupMembersUrl.replace("$","800");
        List<String> projectMasterMembers = getGitLabMasterMembers(getProjectMembersUrl);
        List<String> groupMasterMembers = getGitLabMasterMembers(getGroupMembersUrl);
        return Stream.of(projectMasterMembers,groupMasterMembers).flatMap(Collection::stream).distinct().collect(Collectors.toList());
    }
```
```java
/**
     * 获取master成员
     * @param url
     * @return
     */
    private static List<String> getGitLabMasterMembers(String url) {
        List<String> result = new ArrayList<>();
        OkHttpClient okHttpClient = new OkHttpClient();
        Request request = new Request.Builder().url(url).header("PRIVATE-TOKEN",token).build();
        Response response = null;
        try {
            response = okHttpClient.newCall(request).execute();
            String body = response.body().string();
            JSONArray jsonArray = JSONArray.parseArray(body);
            for (int i = 0;i < jsonArray.size();i++) {
                JSONObject jsonObject = jsonArray.getJSONObject(i);
                if (jsonObject.getInteger("access_level") == 40) {
                    result.add(jsonObject.getString("name"));
                }
            }
        } catch (IOException e) {
            log.error("调用GitLab API失败！",e);
        } finally {
            if (response != null) {
                response.close();
            }
        }
        return result;
    }
```
这个例子中只做了Tag Push Event和Merge Request Event的处理，主要是根据不同的事件构建不同的企业微信消息内容。其他的可以自己扩展。
**`MessageStrategy.java`**
```java
/**
 * 构建消息内容
 */
public interface MessageStrategy {
    String produceMsg(JSONObject jsonObject);
}
```
**`TagMessageStrategy.java`**

```java
@Override
    public String produceMsg(JSONObject jsonObject) {
        String operateType = OPERATE_TYPE_ADD;
        if ("0000000000000000000000000000000000000000".equals(jsonObject.getString("after"))) {
            operateType = OPERATE_TYPE_DELETE;
        }
        Integer projectId = jsonObject.getInteger("project_id");
        JSONObject prObject = jsonObject.getJSONObject("project");
        String repo = prObject.getString("name");
        String operator = jsonObject.getString("user_name");
        String tag = jsonObject.getString("ref");
        String[] tagArr = tag.split("/");
        tag = tagArr[tagArr.length-1];
        String detailUrl = prObject.getString("web_url")+"/tags/"+tag;
        String commitInfo = "";
        if (OPERATE_TYPE_ADD.equals(operateType)) {
            String newTagMsg = jsonObject.getString("message");
            JSONObject latestCommit = jsonObject.getJSONArray("commits").getJSONObject(0);
            String latestCommitMsg = latestCommit.getString("message").replaceAll("\r\n"," ");
            String latestCommitUser = latestCommit.getJSONObject("author").getString("name");
            commitInfo = "\n>Tag描述："+newTagMsg
                    +"\n>最近一次提交信息："+latestCommitMsg
                    +"\n>最近一次提交人："+latestCommitUser;
        }
        List<String> members = GitLabApiUtils.getAllProjectMembers(projectId);
        String alertUsers = members.stream().map(s -> "@"+s+" ").collect(Collectors.joining());
        String alertContent = alertUsers+"<font color=\\\"info\\\">【"+repo+"】</font>"+operator+"<font color=\\\"info\\\">"+operateType+"</font>了一个Tag！"
                +"\n>Tag名称："+tag
                +commitInfo
                +"\n>[查看详情]("+detailUrl+")";
        return alertContent;
    }
```
**`MRMessageStrategy.java`**

```java
@Override
    public String produceMsg(JSONObject jsonObject) {
        String operator = jsonObject.getJSONObject("user").getString("name");
        JSONObject objectAttributes = jsonObject.getJSONObject("object_attributes");
        String operateType = "变更";
        String state = objectAttributes.getString("state");
        if ("closed".equals(state)) {
            operateType = "关闭";
        } else if ("opened".equals(state)) {
            operateType = "新增";
        } else if ("merged".equals(state)) {
            operateType = "审核通过";
        }
        String source = objectAttributes.getString("source_branch");
        String target = objectAttributes.getString("target_branch");
        Integer projectId = objectAttributes.getInteger("target_project_id");
        String title = objectAttributes.getString("title");
        String description = objectAttributes.getString("description");
        JSONObject lastCommit = objectAttributes.getJSONObject("last_commit");
        String lastCommitMsg = lastCommit.getString("message").replaceAll("\n","");
        String lastCommitUser = lastCommit.getJSONObject("author").getString("name");
        String repo = objectAttributes.getJSONObject("target").getString("name");
        String url = objectAttributes.getString("url");
        List<String> members = GitLabApiUtils.getAllProjectMembers(projectId);
        String alertUsers = members.stream().map(s -> "@"+s+" ").collect(Collectors.joining());
        String alertContent = alertUsers+"<font color=\\\"info\\\">【"+repo+"】</font>"+operator+"<font color=\\\"info\\\">"+operateType+"</font>了一个Merge Request！"
                +"\n>标题："+title
                +"\n>描述："+description
                +"\n>Source Branch："+source
                +"\n>Target Branch："+target
                +"\n>最近一次提交信息："+lastCommitMsg
                +"\n>最近一次提交人："+lastCommitUser
                +"\n>[查看详情]("+url+")";
        return alertContent;
    }
```
**`AlertController.java`**

```java
@PostMapping("/alert")
    public String alert(@RequestBody JSONObject jsonObject, HttpServletRequest request) {
        String bodyContext = "发送成功";
        String objectKind = jsonObject.getString("object_kind");
        MessageStrategy messageStrategy = null;
        if("tag_push".equals(objectKind)) {
            messageStrategy = new TagMessageStrategy();
        } else if("merge_request".equals(objectKind)) {
            messageStrategy = new MRMessageStrategy();
        }
        MessageStrategyContext messageStrategyContext = new MessageStrategyContext(messageStrategy);
        String alertContent = messageStrategyContext.buildMessage(jsonObject);
        log.info("消息内容："+alertContent);
        String[] cmds={"curl",weChatSendUrl,"-H"
                ,"Content-Type: application/json","-d","{\"msgtype\": \"markdown\",\"markdown\": {\"content\": \""+alertContent+"\"}}"};
        ProcessBuilder process = new ProcessBuilder(cmds);
        try {
            process.start();
        } catch (Exception e) {
            bodyContext = "发送失败";
        }
        return bodyContext;
    }
```
完整代码请查看[gitlab-to-企业微信](https://github.com/xujian01/webhook)

---
**经过开发之后webhook配置项里的URL自然也就有了，那就是`http://{服务ip}:{服务端口}/alert`**
## 总结
其实hook这种设计在很多地方都有，且不说一些开源中间件，JDK本身就提供了ShutdownHook，最重要的还是了解hook的工作原理，才能更好的使用hook，感受它带来的扩展和便捷。