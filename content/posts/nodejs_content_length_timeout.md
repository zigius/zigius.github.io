---
title: "The story of the content-length header"
date: 2022-12-06T15:59:12+02:00
draft: false
---

As part of my job, Me and my team are responsible for the data pipeline of our division.
Our collection layer of all the analytics events is written in typescript with an express server. It is responsible for handling 
and processing more than 3M requests every minute. 

As part of a rewrite of the collection layer, we noticed that we have requests that take more than a minute to progress. 
![Max response time](https://drive.google.com/uc?id=10FLQfEn645vsVBXOfvpZFlvfF5dvqjpm)

Our p99 response time is a lot better: 
![p99 response time](https://drive.google.com/uc?id=10GTx_K7hrtphNbJYV6bB-A9tqYbPEK3k)

So why do we have requests that take more than a minute to process? 
As part of the research of the max response time we found two issues in our code.

## ALB idle time
We expose the API behind an aws ALB with an idle timeout of 16 seconds. Why do we see requests that take more than 16 seconds? 
This issue still has no definitive solution, and we are working with aws to resolve it

## express body-parser middleware
To identify the requests that took a long time to process we added logs to our global express error handling middleware:
```ts
  app.use(async (err: any, req: express.Request, res: any, next: any) => {
    try {
      metrics.increment({ component: 'acidController', metric: `uncaught_server_error` });
      const msgId = v4();
      const serializedError = _.omit('body')(serializeError(err));
      logger.error(`uncaught server error msgId: ${msgId}. error: ${JSON.stringify(serializedError)}`);
      logger.error(`uncaught server error msgId: ${msgId}. query: ${JSON.stringify(req.query)}`);
      logger.error(`uncaught server error msgId: ${msgId}. method: ${req.method}`);
      logger.error(`uncaught server error msgId: ${msgId}. headers: ${JSON.stringify(req.headers)}`);
      logger.error(`uncaught server error msgId: ${msgId}. body: ${JSON.stringify(req.body)}`);
    } catch (error) {
      logger.error('error in uncaught server error middleware');
    } finally {
      res.status(500).json({ status: 'error' });
    }
  });
```

We noticed that the requests that take a long time are requests that had a mismatch between their `content-length` header and the actual size of the parsed body object. 
Here is an example log where we can see the issue:
```json
{
  "message": "request aborted",
  "code": "ECONNABORTED",
  "expected": 13548,
  "length": 13548,
  "received": 2048,
  "type": "request.aborted",
  "name": "BadRequestError",
  "stack": "BadRequestError: request aborted\n    at IncomingMessage.onAborted (/app/node_modules/raw-body/index.js:231:10)\n    at IncomingMessage.emit (node:events:513:28)\n    at IncomingMessage.emit (node:domain:489:12)\n    at IncomingMessage._destroy (node:_http_incoming:224:10)\n    at _destroy (node:internal/streams/destroy:102:25)\n    at IncomingMessage.destroy (node:internal/streams/destroy:64:5)\n    at abortIncoming (node:_http_server:642:9)\n    at socketOnClose (node:_http_server:636:3)\n    at Socket.emit (node:events:525:35)\n    at Socket.emit (node:domain:489:12)\n    at TCP.<anonymous> (node:net:301:12)"
}
```

After doing some research we found a github issue in express-json middleware that talks about the exact same thing: 
https://github.com/expressjs/body-parser/issues/329


We also found a similar issue in nodejs repo itself: 
https://github.com/nodejs/node/issues/17978

The reason was explained by one of the comments in the nodejs issue: 
https://github.com/nodejs/node/issues/17978#issuecomment-355283416
> This is pretty much a fundamental thing about HTTP/1.1, I’d say. One of the primary purposes of the Content-Length headers is so that the other side can tell when the body is finished.
So if the request size as transmitted in the Content-Length header is too large, the server doesn’t really know when the “real” request body is finished if it doesn’t match that length.
Which would be the behavior you are expecting?

>> A running service receives content-length while the client did not send a payload we wish to catch this case and send a correct HTTP response code which currently seems impossible. Any suggestions?
I’m not sure there’s much you can do besides relying on the timeout, or setting a shorter timeout. In what way is this causing trouble for you?

So, what happens? The content-length header of the request is larger than the actual body, so the middleware keeps waiting for newer chunks to arrive. Those new chunks will never arrive so the 
middleware never finishes its job and the request hangs until the load balancer cancels it.
example code snippet:

```js
const http = require('http');

const server = http.createServer((req, res) => {
    let chunks = [];
    req.on('data', (c) => chunks.push(c));
    req.on('end', () => {
        res.statusCode = 500;
        res.end();
    });
});

server.listen(3000);
server.setTimeout(200);

process.once('SIGINT', () => server.close());
```

The only way to fix this issue is to add a sane timeout to the server itself. 😞

Adding a timeout to the express server will be the subject of a different blog post 😎

