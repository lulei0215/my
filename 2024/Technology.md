nodejs 实现 sse 请求
使用的是 nestjs 框架
很简单 主要实现的就是 请求第三方的流式 自己的接口也是流式返回 
其中异常还需要优化
controller 中 直接添加：@Sse()标识 即可

```typescript
@Post('demo')
  @PublicAPI()
  @Sse()
  async demo() {
    return this.userService.demo();
  }
```
service 中
```typescript
async demo() {
    const userInfo ={
      "messages": [
          {
              "role": "system",
              "content": " "
          },
          {
              "role": "user",
              "content": "帮我翻译下：我是一个AI助手，我可以帮助你完成各种任务。成英文"
          }
      ],
      "model": "ep-20240520100758-jj9tn",
      "stream": true,
      "temperature": 0.1,
      "top_p": 0.8,
      "top_k": 10,
      "max_tokens": 200,
      "frequency_penalty": 1.05
  };
  try {
    const response = await axios.post('https://ark.cn-beijing.volces.com/api/v3/chat/completions', userInfo, {
      headers: {
        'Content-Type': 'application/json',
        'Authorization':"Bearer ****"
      },
      responseType: 'stream'
    });
    return new Observable((subscriber) => {
        response.data.on("data", async (chunk: any) => {
        console.error("chunk=>",chunk.toString())
        const processedData = this.processChunkData(chunk);
        console.error("processedData=>",processedData,typeof(processedData))
        if (processedData != '') {
            subscriber.next(processedData);
        }
      });
      response.data.on('end', () => {
        subscriber.complete();
      });
      response.data.on('error', (err) => {
        subscriber.error(err);
      });
    });
  } catch (error) {
    console.error("Error:", error);
    return throwError(() => new Error("Failed to fetch data"));
  }
}
    processChunkData(chunk: any): string {
    const fa = chunk.toString();
    if (fa.includes("choices")) {
        let msg = "";
        let newStr = fa.replace(/data: |\[DONE\]|/g, '');

        let result = '';
        let lines = newStr.split('\n');

        console.error("lines0=>",lines)
        lines = lines.filter(element => element!= null && element!== "");
        lines.forEach(line => {
            let jsonStr = line;
            // if (line.startsWith('data: ')) {
            //     jsonStr = line.substring(6);
            // }
            try {
                console.error('jsonStr=>', jsonStr)
                if(jsonStr){
                const jsonObj = JSON.parse(jsonStr);
                console.error('jsonObj=>', jsonObj)
                if (jsonObj && jsonObj.choices && jsonObj.choices[0] && jsonObj.choices[0].delta && jsonObj.choices[0].delta.content) {
                    result += jsonObj.choices[0].delta.content;
                }
                }

            } catch (e) {
                console.error('Error parsing JSON:', e);
            }
        });
        return result;
    }
        return '';
    }
```

巨量广告接入
巨量广告转化融合归因优化方案接入手册V2.0
文档链接：https://bytedance.larkoffice.com/docx/CgYBdVzoBogND2xv7PhcSfFNnyc
主要使用了 巨量的实时归因查询，然后回传上传给巨量
```typescript
//获取抖音归因数据
  async getTiktokClientInfo(headers: any, userId: number,type: number) {
    console.log('获取抖音归因数据 gettiktokClientInfo', headers, userId,type)
    let res = null;
    const header = headers;
    const times_un = new Date().getTime() - 604800000;
    const dates_un = new Date(times_un);
    try {
      // 查询当前的 归因数据
      res = await this.getTiktokServer(headers, userId,type);
      console.log('归因数据成功--- ', res)
      if (res != null) {
        // 归因数据成功
        const sendInfo = await this.SendTiktokRep.findOne({
          where: {
            userId: userId,
            adId: res['adid'],
            dataType: type
          }
        })
        console.log('归因数据成功sendInfo ', sendInfo)
        // 查询之前是否 回传了
        if (!sendInfo) {
          console.log('未进行回传 ', sendInfo)
          console.log('未进行回传-res ', res)
          const sendStatus = await this.sendToTiktokServer(res, type);
          // 回传成功 保存数据
          if (sendStatus === 0) {
            console.log('回传成功 保存数据', sendStatus)
            const sendData = new SendTiktok();
            sendData['userId'] = userId;
            sendData['adId'] = res['adid'];
            sendData['dataType'] = type;
            sendData['type'] = 0;
            sendData['channel'] = 1;
            if (Number(headers['os']) === 0) {
              sendData['pkg'] = '*****';
              sendData['android_id'] = res['android_id'];
              if(headers['channel'] == 'HuaWei' || headers['channel'] == 'douyin'){
                sendData['pkg'] = '******';
              }
            } else {
              sendData['pkg'] = 'com.GuangDa.HiMoss';
              sendData['idfv'] = res['idfv'];
            }
            sendData['imei'] = headers['imei'];
            if(type == 2){
              sendData['payAmount'] = headers['amount'];
              sendData['imei'] = header['imei'];
              sendData['ouId'] = header['outer_event_id'];
            }
            sendData['clientIp'] = headers['gwip'];
            sendData['mac'] = headers['mac'];
            sendData['timestamp'] = new Date().getTime();
            console.log('回传成功 保存数据 -sendData', sendData)
            const saveStatus = await this.SendTiktokRep.save(sendData);

            console.log('SendTiktokRep 保存数据', saveStatus)
            // 数据同步
            const tiktokRecord = await this.tiktokRecordRep.findOne({
              where: {
                adid: res['adid'],
                cid: res['cid'],
              },
            });
            console.log('回传成功 同级数据增加 ', tiktokRecord)
            if (tiktokRecord) {
              let rRecordInfo = tiktokRecord['recordInfo'];
              rRecordInfo['register'] = tiktokRecord['register'] + 1;
              tiktokRecord['recordInfo'] = rRecordInfo;
              await this.tiktokRecordRep.save(tiktokRecord);
            } else {
              console.log('未找到 对应数据tiktokRecord')
            }
          } else {
          console.log('进行回传 失败 ', res)
            this.Logger.error(`getTiktokClientInfo catch error:`, res);
          }
        } else {
          console.log('已经进行回传 注册事件', res)
        }
      }
    } catch (error) {
      console.error('gettiktokClientInfo catch error1:', error);
    }
    console.log('getTiktokClientInfo---》', res)
    return res;
  }
/** 查询归因数据 */
  async getTiktokServer(
    headers: any, userId: number,type: number,
  ) {
    console.log('查询归因数据getTiktokServer:', headers, userId,type);
    try {
      let tiktok = null;
      if(type == 2){
        const userinfo  = await this.userAccountRep.findOne({
          where: { id: userId },
        });
        if(!userinfo){
          console.log('getTiktokServer - 用户不存在',headers, userId,type);
          return null;
        }
        const accountId = userinfo['accountId'];
        if (accountId && accountId.indexOf('tiktok&') > -1) {
          const list = accountId.split('&');
          if (list.length == 2 && list[0] == 'tiktok') {
            const tiktokId = list[1].replace('_no', '');
            headers = await this.TiktokRep.findOne({
              where: {
                id: Number(tiktokId)
              },
            });
            if (!headers) {
              console.log('tiktok - 用户不存在',headers, userId,type);
              return null;
            }
            headers['channel'] = userinfo['channel'];
          }
        }else{
          console.log('getTiktokServer-accountId不是来自巨量 - 用户',headers, userId,type);
          return null;
        }
      }
      console.log(
        '查询归因数据getTiktokServer 0:',
        JSON.stringify(headers),
        headers,
      );
     
      const url = 'https://analytics.oceanengine.com/sdk/app/attribution/';
      let body: any =
      {
        "platform": "android",
        "android_id": '', // 仅android需要
        "idfv": '', // 仅android需要
        "package_name": "*****"
      };
      if (Number(headers['os']) == 1) {
        body.platform = "ios";
        body.idfv = headers['idfv'];
        body.package_name = '********';
      } else {
        body['package_name'] = '*****';
        body['android_id'] = headers['android_id'];
        if(headers['channel'] == 'HuaWei' || headers['channel'] == 'douyin'){
          body['package_name'] = '******';
        }
      }
      console.log('noticeTiktokServer 1:', url, JSON.stringify(body));
      let res = await axios.post(url, body, {
        headers: {
          'Content-Type': 'application/json',
        },
      });
      console.log("归因结果请求返回:", res['data']['code'])
      const data = res['data'];
      // 归因成功 保存数据 进行 回传
      if (data['code'] === 0) {
        console.log("归因结果:", data)
        // 参数验证
        if (
          IsParamValid(data, 'aid') &&
          IsParamValid(data, 'cid') &&
          IsParamValid(data, 'advertiser_id') &&
          IsParamValid(data, 'callback_param')
        ) {
          if (Number(headers['os']) === 1) {
            tiktok = await this.TiktokRep.findOne({
              where: {
                adid: data['aid'],
                idfv: headers['idfv'],
                userId: userId,
              },
            });
          } else {
            tiktok = await this.TiktokRep.findOne({
              where: {
                adid: data['aid'],
                android_id: headers['android_id'],
                userId: userId,
              },
            });
          }
          if (tiktok) {
            tiktok['callback'] = data['callback_param'];
            const result = await this.TiktokRep.update(
              { id: tiktok['id'] },
              { callback: data['callback_param'] },
            )
            if (result) return tiktok;
          } else {
            const tiktokRecord = new Tiktok();
            tiktokRecord['userId'] = userId;
            tiktokRecord['adid'] = data['aid'];
            tiktokRecord['callback'] = data['callback_param'];
            tiktokRecord['os'] = Number(headers['os']);
            tiktokRecord['timestamp'] = headers['os'];
            tiktokRecord['android_id'] = data['adv_android_id'];
            tiktokRecord['idfv'] = data['adv_idfv'];
            tiktokRecord['ip'] = headers['gwip'];
            tiktokRecord['ua'] = headers['ua'];
            tiktokRecord['oaid'] = headers['oaid'];
            tiktokRecord['imeiMd5'] = headers['imei'];
            tiktokRecord['callback'] = data['callback_param'];
            const result = await this.TiktokRep.save(tiktokRecord);
            if (result) return tiktokRecord;
          }
        }
      } else {
        this.Logger.error(`getTiktokServer data['code']查询失败:`, body, data['message'], userId);
      }

    } catch (error) {
      this.Logger.error(`getTiktokServer catch error:`, error);
    }
    return null;
  }
  /** 发送抖音服务器 */
  async sendToTiktokServer(config: any, type: number) {
    console.log("发送抖音服务器-sendToTiktokServer", config, type)
    try {
      // 行为列表
      const event_type = {
        0: 'active',
        1: 'active_register',
        2: 'active_pay',
        6: 'next_day_open',
        25: 'game_addiction',
      };
      let device = null;
      // 设备
      if (Number(config['os']) == 0) {
        device = {
          android_id: config['android_id'],
          platform: "android"
        };
        
      } else if (Number(config['os']) == 1) {
        device = {
          idfv: config['idfv'],
          platform: "ios"
        };
      }
      console.log('回传sendTiktokServer device:', device);
      // 地址
      const url = 'https://analytics.oceanengine.com/api/v2/conversion';
      const body = {
        event_type: event_type[type],
        context: {
          ad: {
            callback: config['callback'],
          },
          device: device,
        },
        timestamp: new Date().getTime(),
      };
      console.log('回传sendTiktokServer 0:', url, JSON.stringify(body));
      const res = await axios.post(url, body, {
        headers: {
          'Content-Type': 'application/json',
        },
      });
      console.log("回传结果:", res)
      if (res && res['status'] == 200 && res['data']) {
        return res['data']['code'];
      }
    } catch (error) {
      this.Logger.error(`sendToTiktokServer catch error:`, error);
    }
    return 500;
  }
```