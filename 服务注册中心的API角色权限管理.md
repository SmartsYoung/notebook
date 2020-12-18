**背景**：ServiceCenter是一个具有微服务实例注册/发现能力的微服务组件，提供一套标准的RESTful API对微服务元数据进行管理。管理员可以对用户或服务账号的角色进行更精确的资源访问控制。在 RBAC 中，权限与角色相关联，用户通过成为某种角色或者多种角色而得到这些角色的权限。简化了注册中心对权限的管理。

## 1 API与资源概述

本文介绍的API就是ServiceCenter中的restful-api接口，按照其类型可以分为多种资源。如account、service、instance、role、shema等。每种资源包含不同的操作。关于API列表所属资源的详细划分可参考[链接](https://github.com/apache/servicecomb-service-center/blob/master/server/service/rbac/resource.go)。ServiceCenter可通过次函数查询得到api所对应的资源。

```
func GetResource(api string) string {
   r, ok := resourceMap[api]
   if !ok {
      return resourceMap["*"]
   }
   return r
}
```

## 2 角色与权限

- **角色（Role)** 与权限（Permission）相关联，一个角色对应多个权限

- **用户（User）**与角色（Role）相关联，一个用户对应多个角色

- **权限（Permission）**包含资源，或者与操作组合方式相结合

![role](https://github.com/SmartsYoung/notebook/blob/main/img/servicecomb/role.png)

ServiceCenter默认提供了两种角色，"admin"管理员角色和"developer"开发者角色，管理员角色拥有系统的所有操作角色，开发者角色拥有除了账户信息之外的大多数权限。      

管理员角色对应的资源（列表仅展示部分资源及api）以及相应的操作：

| role  | resource |                             api                              |                 verbs                 |
| :---: | :------: | :----------------------------------------------------------: | :-----------------------------------: |
| admin | account  |       /v4/token、/v4/account、<br />/v4/account/{name}       | ["get", "create", "update", "delete"] |
|       |   role   |                /v4/role、/v4/role/{roleName}                 | ["get", "create", "update", "delete"] |
|       | service  | /v4/{project}/registry/microservices、<br />/v4/{project}/registry/microservices/{serviceId} | ["get", "create", "update", "delete"] |

当我们新建一个角色时，需要给角色分配资源和授予相应的权限，我们可以这样定义资源

```json
{
 "name": "tester",
 "perms": [
         { 
            "resources": ["service","instance"],
            "verbs":     ["get", "create", "update"]
         },
         { 
             "resources": ["rule"],
             "verbs":     ["get"]
         }
    ]
}
```

对于一个普通的角色，可以精细控制到它能访问哪些资源，允许的操作是什么？上图例子所示的“tester”角色只能新增、修改、查询服务，没有权限删除服务。

|  role  | resource |                       api                        |            verbs            |
| :----: | :------: | :----------------------------------------------: | :-------------------------: |
| tester | service  | /v4/{project}/registry/microservices/{serviceId} | ["get", "create", "update"] |



## 3 注册中心如何接入权限管理系统

### 3.1路由初始化

模块是以插件的形式注入到ServiceCenter中，默认未开启rbac模块。开启时需要配置环境变量和设置管理员密码，系统检测到rbac模块开启时，初始化路由：

```go
if rbac.Enabled() {
   roa.RegisterServant(&v4.AuthResource{})
   roa.RegisterServant(&v4.RoleResource{})
}
```

其中，AuthResource结构体实现了rbac的创建用户、用户管理等路由的功能，RoleResource模块实现了添加角色、角色管理等路由功能。

### 3.2 API资源映射

当rbac模块启动之后，访问所有的API需要授权（除白名单之外），ServiceCenter通过将所有的API映射到资源表中， 函数initResourceMap将API资源映射到相应的类别中。

```go
//MapResource save the mapping from api to resource
func MapResource(api, resource string) {
   resourceMap[api] = resource
}

func initResourceMap() {
	rbacframe.MapResource(APIAccountList, ResourceAccount)
	rbacframe.MapResource(APIUserAccount, ResourceAccount)
	rbacframe.MapResource(APIUserPassword, ResourceAccount)
    ...
}
```

### 3.3 初始化默认角色权限

接下来查询数据库中是否存在管理员账号，然后初始化"admin"、"developer"角色资源权限。当系统初始化完成后，访问API需要相应的权限：

```go
func Allow(ctx context.Context, roleList []string, project, resource, verbs string) (bool, error) {
   if ableToAccessResource(roleList, "admin") {
      return true, nil
   }
   // allPerms combines the roleList permission
   var allPerms = make([]*rbacframe.Permission, 0)
   for i := 0; i < len(roleList); i++ {
      r, err := datasource.Instance().GetRole(ctx, roleList[i])
      if err != nil {
         log.Error("get role list errors", err)
         return false, err
      }
      if r == nil {
         log.Warnf("role [%s] has no any permissions", roleList[i])
         continue
      }
      allPerms = append(allPerms, r.Perms...)
   }

   if len(allPerms) == 0 {
      log.Warn("role list has no any permissions")
      return false, nil
   }
   for i := 0; i < len(allPerms); i++ {
      if ableToAccessResource(allPerms[i].Resources, resource) && ableToOperateResource(allPerms[i].Verbs, verbs) {
         return true, nil
      }
   }

   log.Warn("role is not allowed to operate resource")
   return false, nil
}
```

从Allow函数可以看出，决定用户是否有权限访问资源的是角色列表、project、resource、verbs等参数，当从token中解析出角色是admin时，将拥有所有权限，直接返回；若是其它角色，将角色列表拥有的资源汇总，相应资源允许的操作也进行汇总，再与当前访问的API资源与操作进行对比决定是否拥有权限。

## 4功能验证

### 4.1创建角色

本文的重点是分析如和通过角色分配资源并进行相应的操作，认证以及账户管理请参考[链接](https://service-center.readthedocs.io/en/latest/user-guides/rbac.html)。  
为了更好的理解流程，下面我们将通过客户端示例代码，实现基于Service-Center的API资源角色权限管理。  

调用Client的RegisterRole方法注册角色。下面仅展示部分RBAC客户端代码，完整客户端代码请参考：[https://github.com/hityc2019/sc-client]()

```go
// RegisterRole register the role from the service-center
func (c *Client) RegisterRole(role *rbacframe.Role, token string) error {
    if role == nil {
        return errors.New("invalid request role parameter")
    }
    body, err := json.Marshal(role)
    if err != nil {
        return NewJSONException(err, string(body))
    }

    registerURL := c.formatURL(RolePath, nil, nil)
    resp, err := c.httpDo(http.MethodPost, registerURL, c.addTokenToHeader(token), body)
    if err != nil {
        return err
    }

    if resp.StatusCode != http.StatusOK {
        body, err := ioutil.ReadAll(resp.Body)
        if err != nil {
            return NewIOException(err)
        }
        return NewCommonException("result: %d %s", resp.StatusCode, string(body))
    }
    return nil
}
```
通过Role结构体传入角色的资源以及相应的权限。

### 4.2 用户关联角色

创建一个用户peter，并给用户关联刚创建的角色“tester”，即通过Account结构体关联用户和角色等信息。若创建用户时并未关联角色，则该用户没有任何权限。

```go
// RegisterAccount register the account from the service-center
func (c *Client) RegisterAccount(account *rbacframe.Account, token string) error {
    if account == nil {
        return errors.New("invalid request account parameter")
    }
    body, err := json.Marshal(account)
    if err != nil {
        return NewJSONException(err, "parse the account info failed")
    }
    registerURL := c.formatURL(AccountPath, nil, nil)
    resp, err := c.httpDo(http.MethodPost, registerURL, c.addTokenToHeader(token), body)
    if err != nil {
        return err
    }
    if resp == nil {
        return fmt.Errorf("RegisterAccount failed, response is empty, AccountName: %s", account.Name)
    }
	if resp.StatusCode != http.StatusOK {
		body, err := ioutil.ReadAll(resp.Body)
		if err != nil {
			return NewIOException(err)
		}
		return NewCommonException("result: %d %s", resp.StatusCode, string(body))
	}
	return nil
}
```

### 4.3 生成token并访问资源

为刚创建的用户生成token，若成功则返回token。

```go
// GetToken generate token according to user-password
func (c *Client) GetToken(username, password string) (string, error) {
    request := rbacframe.Account{
        Name:     username,
        Password: password,
    }
    body, err := json.Marshal(request)
    if err != nil {
        return "", NewJSONException(err,"parse the username or password failed")
    }

    tokenUrl := c.formatURL(TokenPath, nil, nil)
    resp, err := c.httpDo(http.MethodPost, tokenUrl, nil, body)
    if err != nil {
        return "", err
    }
    if resp == nil {
        return "", fmt.Errorf("user %s generate token failder: ", username)
    }
    body, err = ioutil.ReadAll(resp.Body)
    if err != nil {
        return "", NewIOException(err)
    }

    if resp.StatusCode == http.StatusOK {
        var response rbacframe.Token
        err = json.Unmarshal(body, &response)
        if err != nil {
            return "", NewJSONException(err, string(body))
        }
        return response.TokenStr, nil
    }
    return "", fmt.Errorf("user %s generate token failed, response StatusCode: %d", username, resp.StatusCode)
}
```

用户携带此token访问服务，例如：该用户所拥有的角色拥有访注册微服务API的权限，，会返回200显示注册成功。
```shell script
curl -X POST \
  http://127.0.0.1:30100/v4/default/registry/microservices \
  -H 'Accept: */*' \
  -H 'Authorization: Bearer {peter_token}' \
  -d '{
        "service": {
          "serviceId": "11111-22222-33333",
          "appId": "test",
          "serviceName": "test",
          "version": "1.0.0"
        }
}'
```
但此API未授权删除微服务API的权限，携带此token删除刚创建的微服务返回401未授权。

```shell script
curl -X DElETE \
  http://127.0.0.1:30100/v4/default/registry/microservices/11111-22222-33333 \
  -H 'Accept: */*' \
  -H 'Authorization: Bearer {peter_token}' 
```
