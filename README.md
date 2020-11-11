#####  消息中心bug
> 2020-11-11  
  *  message-kafka: full-gc  

     |问题描述|dealing|
     |:---|:---|
     |message-kafka压测出现少量full-gc|修改closeablehttpclient手动关闭|
     ---
  *  message-notice:  
  
     |问题描述|dealing|
     |:---|:---|
     |message-notice参数校验返回|修改对应返惨结构|
     |message-notice日志级别问题|修改对应error级别日志|
     |message-notice代码入参解析不全|修改入参解析|  
     
  >  代码:  
  
     1. message-kafka  
```java
public Map<Boolean, String> sendSmsByMobiles(Map<String, String> propMap, List<String> mobiles, String nonce, String templateId) {
        Map<Boolean, String> sendResult = new HashMap<>();
        // 请求体
        String requestJsonString = "";
        CloseableHttpResponse response = null;
        long start = System.currentTimeMillis();
        try (CloseableHttpClient httpclient = HttpClients.createDefault()) {
            HttpPost httpPost = new HttpPost(shortMsgUrl);
            httpPost.setHeader("Content-Type", "application/json");
            httpPost.setHeader("Accept", "application/json");
            // 构造请求体
            requestJsonString = constructMsg(propMap, mobiles, nonce, templateId);
            // 打印请求消息
            log.info("sendSmsByMobiles请求消息:{}", requestJsonString);
            HttpEntity responseEntity;
            StringEntity entity = new StringEntity(requestJsonString, "UTF-8");
            httpPost.setEntity(entity);
            long startTime = System.currentTimeMillis();
            // 设置代理
            if (StringUtils.isNotBlank(proxyAddr) && !"-1".equals(proxyAddr)) {
                RequestConfig config;
                HttpHost proxy =
                        new HttpHost(proxyAddr.split(":")[0], Integer.parseInt(proxyAddr.split(":")[1]), "http");
                RequestConfig.Builder defaultBuilder = getInitializedRequestConfig();
                config = defaultBuilder.setProxy(proxy).build();
                httpPost.setConfig(config);
            }
            response = httpclient.execute(httpPost);
            Long costTime = System.currentTimeMillis() - startTime;
            StatusLine status = response.getStatusLine();
            int state = status.getStatusCode();
            log.info("sendSmsByMobiles | content:{} HttpStatus:{},spend time:{}",response, state, costTime);
            if (state == HttpStatus.SC_OK) {
                responseEntity = response.getEntity();
                String respStr = EntityUtils.toString(responseEntity);

                JSONObject infoObject = (JSONObject) JSONObject.parse(respStr);
                log.error("sendSmsByMobiles | response:{}", JSON.toJSONString(infoObject));
                JSONObject resStatusObject = infoObject.getJSONObject("resStatus");
                if (null != resStatusObject) {
                    String resCode = resStatusObject.getString("resCode");

                    // 若存在回车换行等特殊字符则需要转译
                    String resMsg = resStatusObject.getString("resMsg").replaceAll("[\\t\\n\\r]", " ");
                    if ("0".equals(resCode)) {
                        sendResult.put(true, "");
                    } else {
                        sendResult.put(false, resMsg);
                    }
                }
            } else {
                String resMsg = status.getReasonPhrase();
                if (StringUtils.isNotBlank(resMsg)) {
                    resMsg = resMsg.replaceAll("[\\t\\n\\r]", " ");
                }
                sendResult.put(false, resMsg);
            }
            - httpclient.close();
            return sendResult;
        } catch (Exception e) {
            log.error("sendSmsByMobiles| 发送短信发生报错，sendSmsByMobiles请求消息 : {}, "
                            + "sendSmsByMobiles返回消息 : {}, reqBatchNo : {}, 异常为:{}",
                    requestJsonString, response, nonce, e);
            Long cost = System.currentTimeMillis() - start;
            //resultMsg = "error|reqBatchNo=" + nonce + "|" + cost;
            // 不确定此异常什么原因造成，如果是因为中断造成的需要重新设置中断标记
            if (e instanceof InterruptedException) {
                Thread.currentThread().interrupt();
            }
            sendResult.put(false, e.getMessage());
            return sendResult;
        }
    }

```
*  message-notice:  

     |问题描述|dealing|
     |:---|:---|
     |message-notice参数校验返回|修改对应返惨结构|
     |message-notice日志级别问题|修改对应error级别日志|
     |message-notice代码入参解析不全|修改入参解析|   
