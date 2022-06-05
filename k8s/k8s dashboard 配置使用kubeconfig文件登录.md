# [k8s dashboard 配置使用kubeconfig文件登录 ](https://www.cnblogs.com/eddie1127/p/15152496.html)

# 找到admin secrets

```
kubectl -n kubernetes-dashboard get secrets
```

#### 获取token

```ini
DASH_TOCKEN=$(kubectl -n kubernetes-dashboard get secrets kubernetes-dashboard-token-8mz8k -o jsonpath={.data.token} |base64 -d)
```

#### 设置 kubeconfig 文件中的一个集群条目

```javascript
kubectl config set-cluster kubernetes --server=192.168.255.100:30443 --kubeconfig=/usr/local/src/dashbord-admin.conf
```

#### 设置 kubeconfig 文件中的一个用户条目

```ruby
kubectl config set-credentials kubernetes-dashboard --token=$DASH_TOCKEN --kubeconfig=/usr/local/src/dashbord-admin.conf
```

#### 设置 kubeconfig 文件中的一个上下文条目

```ruby
kubectl config set-context kubernetes-dashboard@kubernetes --cluster=kubernetes --user=kubernetes-dashboard --kubeconfig=/usr/local/src/dashbord-admin.conf
```

#### 设置 kubeconfig 文件中的当前上下文

```verilog
kubectl config use-context kubernetes-dashboard@kubernetes --kubeconfig=/usr/local/src/dashbord-admin.conf
```

#### 保存配置文件到本地

```bash
sz /usr/local/src/dashbord-admin.conf
```