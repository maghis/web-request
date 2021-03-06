# web-request
> Simplifies making web requests with TypeScript async/await

This package makes it easier to perform web requests using [TypeScript](http://www.typescriptlang.org/) and [**async/await**](https://blogs.msdn.microsoft.com/typescript/2015/11/03/what-about-asyncawait/).
It wraps the popular [request](https://www.npmjs.com/package/request) package, extending it with an interface that facilitates async/await and strong-typing.

## Examples

Get web-page content as a string...
```js
var result = await WebRequest.get('http://www.google.com/');
console.log(result.content);
```

Get JSON data...
```js
var url = 'http://query.yahooapis.com/v1/public/yql?q=select+*+from+yahoo.finance.quotes+where+symbol+IN+(%22YHOO%22,%22AAPL%22)&format=json&env=http://datatables.org/alltables.env';
var data = await WebRequest.json<any>(url);
for (var quote of data.query.results.quote)
    console.log(quote.Symbol, quote.Bid, 'low='+quote.DaysLow, 'high='+quote.DaysHigh, 'vol='+quote.Volume);  
```

Get JSON data with a strongly typed result...
```js
interface QuoteResult {
    query: {
        results: {
            quote: Array<{
                Symbol: string;
                Bid: string;
                DaysHigh: string;
                DaysLow: string;
                Volume: string;
            }>
        }
    }
}    
var url = 'http://query.yahooapis.com/v1/public/yql?q=select+*+from+yahoo.finance.quotes+where+symbol+IN+(%22YHOO%22,%22AAPL%22)&format=json&env=http://datatables.org/alltables.env';
var data = await WebRequest.json<QuoteResult>(url);
for (var quote of data.query.results.quote)
    console.log(quote.Symbol, quote.Bid, 'low='+quote.DaysLow, 'high='+quote.DaysHigh, 'vol='+quote.Volume);  
```

Perform a series of REST operations, one-by-one...
```js
// Transfer all orders from customer #123 to customer #321 and then delete customer #123...
var orders = await WebRequest.json<Order[]>('http://www.example.com/customers/123/orders');
// Change status of all orders to backorder...
for (var order of orders)
    order.status = "backorder";
await WebRequest.post('http://www.example.com/customers/321/orders', null, orders);
await WebRequest.delete('http://www.example.com/customers/123');
// Flag order #98765 as shipped...
await WebRequest.patch('http://www.example.com/customers/321/orders/98765', null, {status: "shipped"});
```

## Getting Started

Make sure you're running Node v4 and TypeScript 1.7 or higher...
```
$ node -v
v4.2.6
$ npm install -g typescript tsd
$ tsc -v
Version 1.7.5
```

Install the *web-request* package and the typings definitions for Node.js...
```
$ npm install web-request
$ tsd install node
```

Write some code...
```js
import * as WebRequest from 'web-request';
(async function () {
    var result = await WebRequest.get('http://www.google.com/');
    console.log(result.content);
})();
```

Save the above to a file (index.ts), build and run it!
```
$ tsc index.ts typings/node/node.d.ts --target es6 --module commonjs
$ node index.js
<!doctype html><html ...
```

## Response Errors as Exceptions
The **throwResponseError** option will cause any response with a 400 or 500 level status to throw an exception. This option is disabled by default.

Throw an exception for a specific request.
```js
await WebRequest.get('http://xyzzy.com/123', {throwResponseError: true});
```

Throw an exception for any request that results in an error response.
```js
WebRequest.defaults({throwResponseErrors: true});
```

## Interface

```js
function get(uri: string, options?: RequestOptions): Promise<Response<string>>;
function post(uri: string, options?: RequestOptions, content?: any): Promise<Response<string>>;
function put(uri: string, options?: RequestOptions, content?: any): Promise<Response<string>>;
function patch(uri: string, options?: RequestOptions, content?: any): Promise<Response<string>>;
function head(uri: string, options?: RequestOptions): Promise<Response<void>>;
function delete(uri: string, options?: RequestOptions): Promise<Response<string>>;
function json<T>(uri: string, options?: RequestOptions): Promise<T>;
function create<T>(uri: string, options?: RequestOptions, content?: any): Promise<Response<T>>;
function stream(uri: string, options?: RequestOptions, content?: any): Promise<Response<void>>;
function defaults(options: RequestOptions): void;
function debug(value?: boolean): boolean;

interface Request<T> extends request.Request {
    options: RequestOptions;
    response: Promise<Response<T>>;
}

class Response<T> {
    request: Request<T>;
    message: http.IncomingMessage;
    get charset(): string;
    get content(): T;  
    get contentLength(): number; 
    get contentType(): string;
    get cookies(): Cookie[];
    get headers(): Headers;
    get httpVersion(): string;
    get lastModified(): Date;    
    get method(): string;
    get server(): string;
    get statusCode(): number;
    get statusMessage(): string;        
    get uri(): Uri;
}
```

Note the following interfaces are as defined by *request*...

* [RequestOptions](https://www.npmjs.com/package/request#requestoptions-callback)
* [http.IncomingMessage](https://nodejs.org/api/http.html#http_http_incomingmessage)

## More Examples

Setting defaults that apply for all requests is supported...
```js
WebRequest.defaults({baseUrl: 'https://example.com/'});
// now we can make requests without having to specify the root every time...
var orders = await WebRequest.json<Order[]>('/customers/123/orders');
await WebRequest.post('/customers/321/orders', null, orders);
await WebRequest.delete('/customers/123');
```

To make a request that requires authentication...
```js
await WebRequest.get('https://example.com/', {
  auth: {
    user: 'username',
    pass: 'password',
    sendImmediately: false
  }});
```

To make a request with custom headers...
```js
await WebRequest.get('https://example.com', {headers: {'User-Agent': 'request'}});
```

To enable cookies, set **jar** to true or specify a custom cookie jar...
```js
var response = await WebRequest.get('https://www.google.com/', {jar: true});
console.log(response.cookies);
```

Use the **stream** method to request a large resource efficiently with negligible memory impact...
```js
var request = WebRequest.stream('http://img15.hostingpics.net/pics/944021EarthHighRes.png'); // 4.3Mb
var w = fs.createWriteStream('earth.png');
request.pipe(w); // pipe content directly to a file
var response = await request.response; // wait for web-request to complete
await new Promise(resolve => w.on('finish', () => resolve())); // wait for file-write to complete
```

Stream a file up to a server... 
```js
var request = WebRequest.stream('http://example.com/data.json', {method:'post'});
fs.createReadStream('file.json').pipe(request);
await request.response;
```

Stream a file from one server to another...
```js
var request1 = WebRequest.stream('http://test.com/earth.png', {method:'get'});
var request2 = WebRequest.stream('http://example.com/earth.png', {method:'post'});
request1.pipe(request2);
await Promise.all([request1.response, request2.response]);
```
