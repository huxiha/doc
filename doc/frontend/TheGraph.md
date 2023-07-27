## The Graph

- 可以帮助大家在不同区块链网络中跟踪智能合约触发的事件。
- 数据可以通过 GraphQL API 获取供前端展示

**基本原理**

1. dapp 发送交易触发事件
2. graph 持续扫描新块和数据，检查设定的 subgraph 是否存在
3. 如果发现了你在 subgraph 中定义的要寻找的事件，运行你设定的数据传输脚本，在 graph nodes 中创建或更新数据实体
4. dapp 可以使用 GraphQL 从 graph node 中查询数据

**创建部署 subgraph**

1. https://thegraph.com/hosted-service
2. My DashBoard
3. Add Subgraph
4. 简单填写基本信息创建 subgraph
5. 在后端合约 hardhat 文件夹外创建一个 abi.json
6. sudo npm install -g @graphprotocol/graph-cli
7. graph init --contract-name CONTRACT_NAME --product hosted-service GITHUB_NAME/SUBGRAPH_NAME --from-contract CONTRACT_DEPLOYED_ADDRESS --abi ./abi.json --network NETWOEK_NAME graph 按照合约网络类型填写问卷
8. 在 My DashBoard 中获取 access token
9. graph auth --product hosted-service access_token
10. cd graph
11. npm run deploy
12. 然后看 My DashBoard 应该可以看到部署的 subgraph 了

**配置 subgraph**

- 修改 graph 文件夹中的 schema.graphql，设置成你想跟踪的所有变量的 entity
- 在 graph 文件夹下运行 npm run codegen 创建 entity 和 events 跟踪匹配
- 可以根据自己的 entity 修改 src/\*.ts 里面的匹配
- 修改完成之后 npm run deploy

**前端使用 subgraph 抓取感兴趣的数据**

- 写查询语句： 例：

```
  query {
  games(orderBy:id, orderDirection:desc, first: 1) {
  id
  maxPlayers
  entryFee
  winner
  players
  }
  }
```

- 向 subgraph 的 api 发送 POST 请求获取数据: SUBGRAPH_URL 在 My Dashborad 的 subgraph 中的 QUERIES (HTTP)

```

  const _gameArray = await axios.post(SUBGRAPH_URL, { query,});
  const _game = _gameArray.games[0];
  _game.id;
  ...
```
