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
