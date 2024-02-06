## pm2常用命令

1. **pm2  start -n myServerName node.js --watch --port 8080**   // 运行node server端口为8080并命名为myServerName，并且监听脚本变化
2. **pm2 start app.json** // 根据json配置启动node server
3. **pm2 restart/reload/stop/delete myServerName/id **       //  通过名称或者ID 重启/重新加载/暂停/删除 node 服务，后面可以跟多个ID或name
4. **pm2 list -a** // 列举出所有的node服务
5. **pm2 logs --lines 200** // 列出前200行日志
6. **pm2 monit**  // 监控
7. **pm2 show id/name** // 展示具体某一个监控

