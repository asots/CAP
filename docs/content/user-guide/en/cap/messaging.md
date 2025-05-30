# Message

The data sent by using the `ICapPublisher` interface is called `Message`.

!!! WARNING "TimeoutException thrown in consumer using HTTPClient"
    By default, if the consumer throws an `OperationCanceledException` (including `TaskCanceledException`), it is considered normal user behavior, and the exception is ignored. However, if you use `HttpClient` in the consumer method and configure a request timeout, you may need to handle exceptions separately and re-throw non-`OperationCanceledException` exceptions due to a [design issue](https://github.com/dotnet/runtime/issues/21965) in `HttpClient`. Refer to issue #1368 for more details.

## Compensating transaction

Wiki :
[Compensating transaction](https://en.wikipedia.org/wiki/Compensating_transaction)

In some cases, consumers need to return an execution result to inform the publisher, enabling the publisher to perform compensation actions. This process is commonly referred to as message compensation.

Typically, you can notify the upstream system by republishing a new message in the consumer code. CAP simplifies this process by allowing you to specify the `callbackName` parameter when publishing a message. This feature is generally applicable to point-to-point consumption. Below is an example.

For instance, in an e-commerce application, the initial status of an order is "pending." The status is updated to "succeeded" when the product quantity is successfully deducted; otherwise, it is marked as "failed."

```C#
// =============  Publisher =================

_capBus.Publish("place.order.qty.deducted", 
    contentObj: new { OrderId = 1234, ProductId = 23255, Qty = 1 }, 
    callbackName: "place.order.mark.status");    

// publisher using `callbackName` to subscribe consumer result

[CapSubscribe("place.order.mark.status")]
public void MarkOrderStatus(JsonElement param)
{
    var orderId = param.GetProperty("OrderId").GetInt32();
    var isSuccess = param.GetProperty("IsSuccess").GetBoolean();
    
    if(isSuccess){
        // mark order status to succeeded
    }
    else{
       // mark order status to failed
    }
}

// =============  Consumer ===================

[CapSubscribe("place.order.qty.deducted")]
public object DeductProductQty(JsonElement param)
{
    var orderId = param.GetProperty("OrderId").GetInt32();
    var productId = param.GetProperty("ProductId").GetInt32();
    var qty = param.GetProperty("Qty").GetInt32();

    //business logic 

    return new { OrderId = orderId, IsSuccess = true };
}
```

### Controlling callback response

You can inject the `CapHeader` parameter in the subscription method using the `[FromCap]` attribute and utilize its methods to add extra headers to the callback context or terminate the callback.

Example:

```cs
[CapSubscribe("place.order.qty.deducted")]
public object DeductProductQty(JsonElement param, [FromCap] CapHeader header)
{
    var orderId = param.GetProperty("OrderId").GetInt32();
    var productId = param.GetProperty("ProductId").GetInt32();
    var qty = param.GetProperty("Qty").GetInt32();

    // Add additional headers to the response message
    header.AddResponseHeader("some-message-info", "this is the test");
    // Or add a callback to the response
    header.AddResponseHeader(DotNetCore.CAP.Messages.Headers.CallbackName, "place.order.qty.deducted-callback");

    // If you no longer want to follow the sender's specified callback and want to modify it, use the RewriteCallback method.
    header.RewriteCallback("new-callback-name");

    // If you want to terminate/stop, or no longer respond to the sender, call RemoveCallback to remove the callback.
    header.RemoveCallback();

    return new { OrderId = orderId, IsSuccess = true };
}
```

## Heterogeneous system integration

In version 3.0+, we reconstructed the message structure. We used the Header in the message protocol in the message queue to transmit some additional information, so that we can do it in the Body without modifying or packaging the user’s original The message data format and content are sent.

This approach facilitates better integration with heterogeneous systems. Compared to previous versions, users no longer need to understand the internal message structure used by CAP to complete integration tasks.

Now we divide the message into Header and Body for transmission.

The data in the body is the content of the original message sent by the user, that is, the content sent by calling the Publish method. We do not perform any packaging, but send it to the message queue after serialization.

In the Header, we need to pass some additional information so that the CAP can extract the key features for operation when the message is received.

The following is the content that needs to be written into the header of the message when sending a message in a heterogeneous system:

 | Key           | DataType | Description                                                    |
 | ------------- | -------- | -------------------------------------------------------------- |
 | cap-msg-id    | string   | Message Id, Generated by snowflake algorithm, can also be guid |
 | cap-msg-name  | string   | The name of the message                                        |
 | cap-msg-type  | string   | The type of message, `typeof(T).FullName`(not required)        |
 | cap-senttime  | string   | sending time (not required)                                    |
 | cap-kafka-key | string   | Partitioning by Kafka Key                                      |

### Custom headers

To consume messages sent without CAP headers, both AzureServiceBus, Kafka and RabbitMQ consumers can inject a minimal set of headers using the `CustomHeadersBuilder` property as shown below (RabbitMQ example):
```C#
container.AddCap(x =>
{
    x.UseRabbitMQ(z =>
    {
        z.ExchangeName = "TestExchange";
        z.CustomHeadersBuilder = (msg, sp) =>
        [
            new(DotNetCore.CAP.Messages.Headers.MessageId, sp.GetRequiredService<ISnowflakeId>().NextId().ToString()),
            new(DotNetCore.CAP.Messages.Headers.MessageName, msg.RoutingKey)
        ];
    });
});
```

After adding `cap-msg-id` and `cap-msg-name`, CAP consumers receive messages sent directly from any external system, like the RabbitMQ management tool when using RabbitMQ as a transport.

To publish messages with CAP headers 
```C#
var headers = new Dictionary<string, string?>()
{
    {"cap-kafka-key", request.OrderId }
};
_publisher.Publish<OrderRequest>("OrderRequest", request,headers);
```


## Scheduling

After CAP receives a message, it sends the message to Transport(RabitMq, Kafka...), which is transported by transport.
 
When you send message using the `ICapPublisher` interface, CAP will dispatch message to the corresponding Transport. Currently, bulk messaging is not supported.

For more information on transports, see [Transports](../transport/general.md) section.

## Storage 

CAP will store the message after receiving it. For more information on storage, see the [Storage](../storage/general.md) section.

## Retry

Retrying is a crucial aspect of the CAP architecture. CAP retries messages that fail to send or execute, employing several retry strategies throughout its design.

### Send retry

During the message sending process, when the broker crashes or the connection fails or an abnormality occurs, CAP will retry the sending. Retry 3 times for the first time, retry every minute after 4 minutes (FallbackWindowLookbackSeconds), and +1 retry. When the total number of retries reaches 50, CAP will stop retrying.

You can adjust the total number of retries by setting [FailedRetryCount](configuration.md#failedretrycount) in CapOptions Or use [FailedThresholdCallback](configuration.md#failedthresholdcallback) to receive notifications when the maximum retry count is reached.

It will stop when the maximum number of times is reached. You can see the reason for the failure in Dashboard and choose whether to manually retry.

### Consumption retry

The consumer method is executed when the Consumer receives the message and will retry when an exception occurs. This retry strategy is the same as the send retry.

We introduced database-based distributed locks in version 7.1.0 to deal with the problem of concurrent data acquisition of database retries under multiple instances, you need to explicitly configure `UseStorageLock` option to true.

Whether sending fails or consumption fails, we will store the exception message in the cap-exception field within the message header. You can find it in the Content field's JSON in the database table.

## Data Cleanup

There is an `ExpiresAt` field in the database message table indicating the expiration time of the message. When the message is sent successfully, status will be changed to `Successed`, and `ExpiresAt` will be set to **1 day** later. 

Consuming failure will change the message status to `Failed` and `ExpiresAt` will be set to **15 days** later (You can use [FailedMessageExpiredAfter](configuration.md#failedmessageexpiredafter) configuration items to custom).

By default, the data of the message in the table is deleted every **5 minutes** to avoid performance degradation caused by too much data. The cleanup strategy `ExpiresAt` is performed when field is not empty and is less than the current time. 

That is to say, the message with the status Failed (by default they have been retried 50 times), if you do not have manual intervention for 15 days, it will **also be** cleaned up.

You can use [CollectorCleaningInterval](configuration.md#collectorcleaninginterval) configuration items to custom the interval time.
