<!-- YAML
added: v0.9.11
changes:
  - version: v8.0.0
    pr-url: https://github.com/nodejs/node/pull/11608
    description: The `chunk` argument can now be a `Uint8Array` instance.
-->

* `chunk` {Buffer|Uint8Array|string|any} 数据块移动到可读队列底部。对于不以对象模式运行的流，`chunk` 必须是字符串， `Buffer` 或者 `Uint8Array`。对于对象流， `chunk` 任何非`null`的值。

`readable.unshift()` 方法会把一块数据压回到`Buffer`内部。
这在如下特定情形下有用：
代码正在消费一个数据流，已经"乐观地"拉取了数据。
又需要"反悔-消费"一些数据，以便这些数据可以传给其他人用。

*注意*: [`'end'`][] 事件已经触发或者运行时错误抛出后，`stream.unshift(chunk)` 方法不能被调用。

使用 `stream.unshift()` 的开发者一般需要换一下思路，考虑用一个[Transform][] 流替代. 
更多信息请查看[API for Stream Implementers][]部分。

```js
// Pull off a header delimited by \n\n
// use unshift() if we get too much
// Call the callback with (error, header, stream)
const { StringDecoder } = require('string_decoder');
function parseHeader(stream, callback) {
  stream.on('error', callback);
  stream.on('readable', onReadable);
  const decoder = new StringDecoder('utf8');
  let header = '';
  function onReadable() {
    let chunk;
    while (null !== (chunk = stream.read())) {
      const str = decoder.write(chunk);
      if (str.match(/\n\n/)) {
        // found the header boundary
        const split = str.split(/\n\n/);
        header += split.shift();
        const remaining = split.join('\n\n');
        const buf = Buffer.from(remaining, 'utf8');
        stream.removeListener('error', callback);
        // remove the readable listener before unshifting
        stream.removeListener('readable', onReadable);
        if (buf.length)
          stream.unshift(buf);
        // now the body of the message can be read from the stream.
        callback(null, header, stream);
      } else {
        // still reading the header.
        header += str;
      }
    }
  }
}
```

*注意*： 不像 [`stream.push(chunk)`][stream-push]，`stream.unshift(chunk)`在重置流的内部读取状态时是不会结束读取过程。 如果在读取过程中调用 `readable.unshift()` 则会导致异常 (例如：即来自自定义流上的 [`stream._read()`][stream-_read]内部方法上的实现)。 应该在调用 `readable.unshift()`方法之后适当调用 [`stream.push('')`][stream-push] 来重置读取状态，执行读取的过程中最好避免调用 `readable.unshift()`方法。

