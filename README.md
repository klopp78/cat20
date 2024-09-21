修改代码文件：cat-token-box/packages/cli/src/common/apis-rpc.ts
搜索出rpc_listunspent这个方法整体删除掉，改为上面上面文件内容

export const rpc_listunspent = async function (
  config: ConfigService,
  walletName: string,
  address: string,
): Promise<UTXO[] | Error> {
  const Authorization = `Basic ${Buffer.from(
    `${config.getRpcUser()}:${config.getRpcPassword()}`,
  ).toString('base64')}`;

  return fetch(config.getRpcUrl(walletName), {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      Authorization,
    },
    body: JSON.stringify({
      jsonrpc: '2.0',
      id: 'cat-cli',
      method: 'listunspent',
      params: [0, 9999999, [address]],
    }),
  })
    .then((res) => {
      if (res.status === 200) {
        return res.json();
      }
      throw new Error(res.statusText);
    })
    .then((res: any) => {
      if (res.result === null) {
        throw new Error(JSON.stringify(res));
      }
  
      const utxos: UTXO[] = res.result
      .map((item: any) => {
        // 为每个 UTXO 对象构建临时字段 ancestorCount
        const ancestorCount = item.ancestorcount !== undefined ? item.ancestorcount : 0;

        return {
          address: item.address, // 可选字段
          txId: item.txid,
          outputIndex: item.vout,
          script: item.scriptPubKey,
          satoshis: new Decimal(item.amount).mul(new Decimal(100000000)).toNumber(),
          ancestorCount: ancestorCount, // 添加此字段用于排序
        };
      })
      // 过滤出符合余额要求的 UTXO 
      .filter((utxo: any) => utxo.satoshis > 10000 && utxo.ancestorCount < 24)
      // 按 ancestorCount 从小到大排序 
      .sort((a: any, b: any) => a.ancestorCount - b.ancestorCount); 

    // 取排序后的第一个 UTXO 并去掉临时的 ancestorCount 字段
    if (utxos.length > 0) {
      const bestUtxo = utxos[0];
      return [{
        address: bestUtxo.address,
        txId: bestUtxo.txId,
        outputIndex: bestUtxo.outputIndex,
        script: bestUtxo.script,
        satoshis: bestUtxo.satoshis,
      }];
    }

    // 如果没有符合条件的 UTXO，则返回空数组
    return [];
    })
    .catch((e: Error) => {
      return e;
    });
};
