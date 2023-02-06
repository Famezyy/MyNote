# 第07章_DashBoard

之前在 kubernetes 中完成的所有操作都是通过命令行工具`kubectl`完成的。其实，为了提供更丰富的用户体验，kubernetes 还开发了一个基于 web 的用户界面（Dashboard）。用户可以使用 Dashboard 部署容器化的应用，还可以监控应用的状态，执行故障排查以及管理 kubernetes 中各种资源。

https://kubernetes.io/zh-cn/docs/tasks/access-application-cluster/web-ui-dashboard/

1. 下载 yaml，并运行 Dashboard

   ```bash
   # 下载 yaml
   [root@k8s-master01 ~]# wget  https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml
   
   # 修改 kubernetes-dashboard 的 Service 类型
   kind: Service
   apiVersion: v1
   metadata:
     labels:
       k8s-app: kubernetes-dashboard
     name: kubernetes-dashboard
     namespace: kubernetes-dashboard
   spec:
     type: NodePort  # 新增
     ports:
       - port: 443
         targetPort: 8443
         nodePort: 30009  # 新增
     selector:
       k8s-app: kubernetes-dashboard
   
   # 部署
   [root@k8s-master01 ~]# kubectl create -f recommended.yaml
   
   # 查看 namespace 下的 kubernetes-dashboard 下的资源
   [root@k8s-master01 ~]# kubectl get pod,svc -n kubernetes-dashboard
   NAME                                            READY   STATUS    RESTARTS   AGE
   pod/dashboard-metrics-scraper-c79c65bb7-zwfvw   1/1     Running   0          111s
   pod/kubernetes-dashboard-56484d4c5-z95z5        1/1     Running   0          111s
   
   NAME                               TYPE       CLUSTER-IP      EXTERNAL-IP  PORT(S)         AGE
   service/dashboard-metrics-scraper  ClusterIP  10.96.89.218    <none>       8000/TCP        111s
   service/kubernetes-dashboard       NodePort   10.104.178.171  <none>       443:30009/TCP   111s
   ```

2. 创建访问账户，获取 token

   https://github.com/kubernetes/dashboard/blob/master/docs/user/access-control/creating-sample-user.md

   ```yaml
   apiVersion: v1
   kind: ServiceAccount
   metadata:
     name: admin-user
     namespace: kubernetes-dashboard
     
   ---
   
   apiVersion: rbac.authorization.k8s.io/v1
   kind: ClusterRoleBinding
   metadata:
     name: admin-user
   roleRef:
     apiGroup: rbac.authorization.k8s.io
     kind: ClusterRole
     name: cluster-admin
   subjects:
   - kind: ServiceAccount
     name: admin-user
     namespace: kubernetes-dashboard
   ```

   ```bash
   kubectl -n kubernetes-dashboard create token admin-user
   ```

3. 通过浏览器访问 Dashboard 的 UI（https://192.168.11.101:30009），在登录页面上输入上面的 token

删除时执行以下两条命令：

```bash
kubectl -n kubernetes-dashboard delete serviceaccount admin-user
kubectl -n kubernetes-dashboard delete clusterrolebinding admin-user
```

