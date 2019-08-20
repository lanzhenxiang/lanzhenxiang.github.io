---
title: HDX SDK for HTML5
date: 2019-08-19 15:37:19
tags: HTML5
---

### Introduction

Citrix Receiver for HTML5 enhances support for HDX and SDK sessions by enabling you to customize your delivery model for Citrix hosted apps and desktops through your website. This feature is particularly useful for building a rich app experience in your Enterprise portals. It can be used to provide a rich app experience for users as a service when hosting Citrix Receiver for HTML5 on your web server while launching Citrix hosted apps and desktops from your website.

### Getting Started

1. Copy CitrixHTML5SDK.js, HDXLauncher.js, HDXEngine.html files to the same folder as the parent HTML page.
2. Include CitrixHTML5SDK.js in parent HTML page.
3. Set the full path of HTML5 Client.
4. Set the connection parameters for launching the session.
5. Attach the events if required.
6. Start the session by passing ICA.You can refer to StoreFront Web APIs to fetch ICA.
   Click here to access full API documentation.

### 开发实践

以上的文档内容为 Citrix 提供的官方对接文档，下面主要讲到的是我在对接文档时遇到的疑问，以及解决办法。Citrix SDK 提供的文件为

1. CitrixReceiverHTML5Hosting_2.6.9.4033.zip
2. CitrixReceiverHTML5SDK_2.6.9.4033.zip
3. CitrixReceiverHTML5SDK_2.6.9.4033_Doc.zip
4. CitrixReceiverHTML5SDK_2.6.9.4033_Example.zip
   SDK 文件请到 Citrix 官网下载

#### 所遇问题 SDK 在第三方 web 应用使用时，请求接口不支持跨域（CORS）。

在对接 Citrix SDK 时，我用到的场景是第三方 Web 应用接入 Citrix SDK，如果直接拷贝 CitrixHTML5SDK.js, HDXLauncher.js, HDXEngine.html 文件到 web 目录，通过 JS 直接请求 StoreWeb API,由于浏览器的同源策略，以及服务端没有设置允许跨域请求，会直接报错（Cross-Origin Read Blocking (CORB) blocked cross-origin response）。我在这里的解决办法是通过服务端代理接口请求，获取 ICA 文件，前端直接获取 ICA 文件调用 SDK 进行加载。获取 ICA 主要步骤：

1. 获取 configuration，获取 crsfToken，并保证后续的请求必须携带 crsfToken,获取后续请求的接口地址，返回值为 XML 文件。
2. 获取登录认证的方法，需要验证 PostCredentials 方法是否存在，如果不存在说明 HTTP Basic 认证未开启。
3. 使用用户密码登录，获取会话 Cookie，并在后续的请求带上会话的 cookie。
4. 获取资源列表 listResource,判断 ICA 文件类型，并指定的 ICA 资源信息。
5. 获取指定 ICA 文件的 ICA launch 状态。
6. 通过 ICA 资源文件的 URL，获取最终的 ICA 文件。
7. 返回 ICA 文件供前端使用。

后端代码部分实现如下：

```
  /**
     * 获取crsfToken
     *
     * crsfToken configuration后面的每次请求必须带上
     */
    protected function getCrsfToken()
    {
        return $this->cookies->getCookieByName("CsrfToken") ?
        $this->cookies->getCookieByName("CsrfToken")->getValue() : null;
    }

    /**
     * 获取配置信息
     */
    protected function configuration()
    {

        $configuration_url = $this->storeWebUri . $this->homeConfigurationUri;

        $res = $this->request($configuration_url);
        if ($res) {
            $res_data = $res->getBody()->getContents();
            return true;
        }
        return false;
    }
    /**
     * 获取登录认证方法
     *
     * 需要验证PostCredentials方法是否存在，如果不存在说明HTTP Basic认证未开启
     *
     * Open the StoreFront admin console, select the Authentication node and click "Add/Remove Methods". Enable "HTTP Basic".
     */
    protected function getAuthMethods()
    {
        $auth_method_url = $this->storeWebUri . $this->authMethodsUri;

        $res = $this->request($auth_method_url);
        if ($res) {
            $res_data = $res->getBody()->getContents();
            $methods_info = $this->xmlToArray($res_data);
            $url = null;

            foreach ($methods_info['method'] as $method) {
                if ('PostCredentials' === $method['@attributes']['name']) {
                    $url = $method['@attributes']['url'];
                    break;
                }
            }

            if (!$url) {
                Log::error('Open the StoreFront admin console, select the Authentication node and click "Add/Remove Methods". Enable "HTTP Basic".');
                return false;
            }
            $this->loginAttemptUri = $url;
            return true;
        }
        return false;
    }

    /**
     * 用户登录
     */
    protected function login($username, $password)
    {
        $data = [
            "username" => $username,
            "password" => $password,
        ];

        $login_url = $this->storeWebUri . $this->loginAttemptUri;
        $res = $this->request($login_url, $data);
        if ($res) {
            $res_data = $res->getBody()->getContents();
            $login_info = $this->xmlToArray($res_data);
            if ('success' === $login_info['Result']) {
                return true;
            } else {
                Log::error('Login failed - try again');
            }

        }
        return false;
    }

    /**
     * 获取ica资源列表
     */
    protected function listResources($hostname)
    {
        $list_resources_url = $this->storeWebUri . $this->listResourcesUri;
        $res = $this->request($list_resources_url);
        if ($res) {
            $res_data = $res->getBody()->getContents();
            $resources = json_decode($res_data, true);
            if ($resources && $resources["resources"]) {
                foreach ($resources['resources'] as $resource) {
                    if (!in_array('ica30', $resource['clienttypes'])) {
                        continue;
                    }
                    // 获取当前用户机器的ICA resource
                    if ($resource['name'] === $hostname) {
                        return $resource;
                    }
                }
            }
        }
        return false;
    }

    /**
     * 获取ICA  launch 状态
     */
    protected function launchStatus($resource)
    {
        $launch_status_url = $this->storeWebUri . $resource['launchstatusurl'];

        $res = $this->request($launch_status_url);
        if ($res) {
            $res_data = $res->getBody()->getContents();
            $launch_data = json_decode($res_data, true);
            if ('success' === $launch_data['status']) {
                return true;
            }
        }
        return false;
    }

    /**
     * 获取Ica 文件
     */
    protected function performLaunch($resource)
    {
        $ica_file_url = $this->storeWebUri . $resource['launchurl'];

        $query = [
            'launchId' => time(),
            'CsrfToken' => $this->getCrsfToken(),
        ];
        $res = $this->request($ica_file_url, [], $query);
        if ($res) {
            $ica_file = $res->getBody()->getContents();
            return $ica_file;
        }

        return false;
    }
```

前端通过接口请求 ICA 文件，然后加载 ICA 文件。

```
//icaData 为后端返回的ICA文件，文件格式为text
function sessionCreation(sessionObject) {
         var handler = function (eventObj) {
             console.log("clicked", eventObj);
         };
         console.log(sessionObject);
         sessionObjs[sessionObjs.length] = sessionObj = sessionObject;

         var launchData = {};
         launchData["type"] = "ini"; //document.getElementById("icaType").value;
         launchData["value"] = icaData; //(document.getElementById("icaType").value == "JSON" )? icaData : text;
         function connectionHandler(event) {
             console.log("Event Received : " + event.type);
             console.log(event.data);
             window.opener = null;
             window.close();
         }
         if (document.getElementById("connecting").checked) {
             sessionObject.addListener("onConnection", connectionHandler);
         }

         function connectionClosedHandler(event) {
             console.log("Event Received : " + event.type);
             console.log(event.data);
             window.opener = null;
             window.close();
         }
         //  if (document.getElementById("connectingClosed").checked) {
         //  }
         sessionObject.addListener("onConnectionClosed", connectionClosedHandler);

         function onErrorHandler(event) {
             console.log("Event Received : " + event.type);
             console.log(event.data);
         }
         if (document.getElementById("error").checked) {
             sessionObject.addListener("onError", onErrorHandler);
         }

         function onURLRedirectionHandler(event) {
             console.log("Event Received : " + event.type);
             console.log(event.data);
         }
         if (document.getElementById("urlredirection").checked) {
             sessionObject.addListener("onURLRedirection", onURLRedirectionHandler);
         }

         var cst = [{
             "id": "custom3",
             "config": {
                 "isPrimary": true,
                 "imageUrl": "http://<path_of_the_image>/back.png",
                 "toolTip": "back",
                 "position": "front"
             },
             "handler": function (e) {
                 sessionObj.sendSpecialKeys(["ALt", "ArrowLeft"]);
             }
         }, {
             "id": "custom4",
             "config": {
                 "isPrimary": true,
                 "imageUrl": "http://<path_of_the_image>/fwd.png",
                 "toolTip": "forward",
                 "position": "front"
             },
             "handler": function (e) {
                 sessionObj.sendSpecialKeys(["ALt", "ArrowRight"]);
             }
         }];

         sessionObject.addToolbarBtns(cst);
         sessionObject.start(launchData);
     }
```

**注意:连接函数这里有一个变量 HTML5ClientPath，这个值为 SDK 存放的 web 访问路径。如:
http://localhost:9999/citrix/**

```
 function Connect() {
         try {
             citrix.receiver.setPath(HTML5ClientPath);
             var cp = getConnectionParams();
             if (cp !== null) {
                 citrix.receiver.createSession("", getConnectionParams(), sessionCreation);
             }
         } catch (e) {
             //console.log(e);
             alert(e.description);
         }
     }
```
