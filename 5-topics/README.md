# Topics

In the previous tutorial we improved our logging system. Instead of using a fanout exchange only capable of dummy broadcasting, we used a direct one, and gained a possibility of selectively receiving the logs.

Although using the direct exchange improved our system, it still has limitations - it can't do routing based on multiple criteria.

In our logging system we might want to subscribe to not only logs based on severity, but also based on the source which emitted the log. You might know this concept from the syslog unix tool, which routes logs based on both severity (info/warn/crit...) and facility (auth/cron/kern...).

That would give us a lot of flexibility - we may want to listen to just critical errors coming from 'cron' but also all logs from 'kern'.

To implement that in our logging system we need to learn about a more complex topic exchange.

## Terminology

| Term          | Description                                                                        |
|---------------|------------------------------------------------------------------------------------|
| Exchange      | An exchange is responsible for routing the messages to the different queues with the help of attributes, bindings, and routing keys. |
| Binding       | A binding is a "link" that you set up to bind a queue to an exchange.                |
| Binding Key   | The binding key is the key you bind the queue to the exchange with, which is compared in the exchange to the routing key of the message. |
| Routing Key   | The routing key is a message attribute. The exchange might look at this key when deciding how to route the message to queues (depending on exchange type). The routing key is like an address for the message. |

## Routing Key vs Topics

| Feature        | Routing Key Binding                                       | Topics (Topic Exchange)                                                                                                                                   |
|------------------|-----------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------|
| Exchange Type   | Direct, Fanout, Headers, Topic                            | Topic                                                                                                                                                       |
| Routing Key     | Exact string match required for binding and routing       | Structured with dots (.), allows for wildcards (*) and hashes (#) for flexible matching                                                                       |
| Wildcards       | Not supported                                            | Supports * for single-word matches and # for multi-word matches                                                                                            |
| Flexibility     | Less flexible, requires exact matches                    | More flexible, allows for broader matching patterns using wildcards                                                                                          |
| Use Cases       | Simple routing scenarios, direct message delivery          | Complex routing scenarios, content-based routing, publish-subscribe patterns                                                                                       |
| Examples        | Order processing, task assignment, logging                       | Log routing based on severity levels (e.g., "logs.info", "logs.error"), stock market updates based on sector and company (e.g., "stocks.us.tech.apple")       |

## Topic exchange

Messages sent to a `topic` exchange can't have an arbitrary `routing_key` - it must be a list of words, delimited by dots. The words can be anything, but usually they specify some features connected to the message. A few valid routing key examples: "`stock.usd.nyse`", "`nyse.vmw`", "`quick.orange.rabbit`". There can be as many words in the routing key as you like, up to the limit of 255 bytes.

The binding key must also be in the same form. The logic behind the `topic` exchange is similar to a `direct` one - a message sent with a particular routing key will be delivered to all the queues that are bound with a matching binding key. However there are two important special cases for binding keys:

- `*` (star) can substitute for exactly one word.
- `#` (hash) can substitute for zero or more words.

It's easiest to explain this in an example:

![Alt text](image.png)

In this example, we're going to send messages which all describe animals. The messages will be sent with a routing key that consists of three words (two dots). The first word in the routing key will describe speed, second a colour and third a species: "`<speed>.<colour>.<species>`".

We created three bindings: Q1 is bound with binding key "`*.orange.*`" and Q2 with "`*.*.rabbit`" and "`lazy.#`".

These bindings can be summarised as:

- Q1 is interested in all the orange animals.
- Q2 wants to hear everything about rabbits, and everything about lazy animals.

A message with a routing key set to "`quick.orange.rabbit`" will be delivered to both queues. Message "`lazy.orange.elephant`" also will go to both of them. On the other hand "`quick.orange.fox`" will only go to the first queue, and "lazy.brown.fox" only to the second. "`lazy.pink.rabbit`" will be delivered to the second queue only once, even though it matches two bindings. "`quick.brown.fox`" doesn't match any binding so it will be discarded.

What happens if we break our contract and send a message with one or four words, like "`orange`" or "`quick.orange.new.rabbit`"? Well, these messages won't match any bindings and will be lost.

On the other hand "`lazy.orange.new.rabbit`", even though it has four words, will match the last binding and will be delivered to the second queue.

---
### Sidenote: Topic exchange

Topic exchange is powerful and can behave like other exchanges.

When a queue is bound with "`#`" (hash) binding key - it will receive all the messages, regardless of the routing key - like in `fanout` exchange.

When special characters "`*`" (star) and "`#`" (hash) aren't used in bindings, the topic exchange will behave just like a `direct` one.

---

## When to not use the topic exchange

You can use the binding key `#` to catch all messages published to a topic exchange, but that is not recommended and you should probably look at fanout exchange instead.
If none of the binding keys uses any wildcards, the exchange behaves just like the direct exchange so that might be a better choice.


## Putting it all together

We will use a topic exchange instead of the direct one. Our `routing_key`s will be two words (two dots `.` in the key), the first defining the severity, the second defining the source. E.g. "`<severity>.<source>`".

## Ways to run 

To receive all the logs:

```
go run receive_logs_topic/main.go "#"
```

To receive all logs from the facility "kern":

```
go run receive_logs_topic/main.go "kern.*"
```

Or if you want to hear only about "critical" logs:

```
go run receive_logs_topic/main.go "*.critical"
```

You can create multiple bindings:

```
go run receive_logs_topic/main.go "kern.*" "*.critical"
```

And to emit a log with a routing key "kern.critical" type:

```
go run emit_log_topic/main.go "kern.critical" "A critical kernel error"
```