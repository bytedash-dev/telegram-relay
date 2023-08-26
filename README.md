# Telegram Relay

Telegram Relay is a [SignalR](https://dotnet.microsoft.com/en-us/apps/aspnet/signalr) [Hub](https://learn.microsoft.com/en-us/aspnet/core/signalr/hubs) that is able to relay [Telegram](https://telegram.org/) Database Library ([TDLib](https://core.telegram.org/tdlib)) API to-and-fro the Telegram Server. This eliminates the needs to dabble in the intricacies of building the binaries and hooking it up. Telegram Relay is doing all the heavy lifting to integrate with TDLib while providing an easy to use interface for clients to connect with. Clients will be connecting with Telegram Relay mostly with SignalR and also REST at a minimal capacity. Telegram Relay will then be interacting with TDLib via the [JSON interface](https://core.telegram.org/tdlib/docs/td__json__client_8h.html). As such, clients will be sending/receiving data/payloads to/from Telegram Relay in JSON string.

Currently Telegram Relay is using TDLib version `1.8.12`

The Telegram Relay Client API is very easy to implement and has only a handful of methods. 99% of the API that you will be consuming are TDLib's which you can refer to on their documentations in their website [here](https://core.telegram.org/tdlib/docs/classtd_1_1td__api_1_1_function.html) and [here](https://core.telegram.org/tdlib/docs/classtd_1_1td__api_1_1_object.html).

## Usages

### Prerequisite

 1. You will need knowledge in ASP.NET Core SignalR Client API in order to be connected to the Telegram Relay SignalR Hub (Server). There are several [clients](https://learn.microsoft.com/en-us/aspnet/core/signalr/client-features) available but we will be using the [JavaScript client](https://learn.microsoft.com/en-us/aspnet/core/signalr/javascript-client) in the examples in this document. In order to follow the examples, set up the client first.
 2. You will need to apply for the Telegram's API access from [https://my.telegram.org/](https://my.telegram.org/).
 3. You will need to understand the concepts used in TDLib. The [Getting Started Guide](https://core.telegram.org/tdlib/getting-started) is an excellent place for that. It is important to understand what does it mean by sending a request `synchronously` and `asynchronously`.

### Connecting to the SignalR Hub
First and foremost, we will be connecting to the SignalR Hub using the SignalR JavaScript Client. Connect to the SignalR Hub via the url `https://[host_name]/signalr-telegram`.
```
const connection = new window.signalR.HubConnectionBuilder()
    .withUrl("https://telegram.bytedash.com.my/signalr-telegram")
    .configureLogging(signalR.LogLevel.Information)
    .withAutomaticReconnect({
        nextRetryDelayInMilliseconds: retryContext => {
            return 5 * 1000; // attempt to retry every 5 seconds until successful
        }
    })
    .build();

await connection.start(); // connect to the Hub
```
For more information on the JavaScript API, please refer to its [JavaScript API reference](https://learn.microsoft.com/en-us/javascript/api/@microsoft/signalr).

### Initialise the connection
Once connected to the Hub, you will need to initialise the connection for Telegram Relay to set up the necessary folders on the server. This will allow different Telegram accounts to be connected at the same time and their files and data to be hosted in their own respective space.

The `Init` API accepts a single param `accountName` of type `string`.
```
const accountName = "+60123456789";
await connection.send("Init", accountName);
```
The `accountName` can be any ASCII characters that can uniquely identify the Telegram accounts used in Telegram Relay. It can be a Telegram account phone number, email address or user's name. After the authentication with TDLib using this `accountName` is successful, subsequent initialisation with the same `accountName` doesn't require authentication anymore and the Telegram account is ready to be accessed.

### TDLib "td_send" API
This is the Telegram Relay SignalR API which is equivalent to TDLib's **td_send** API outlined in [JSON interface of TDLib](https://core.telegram.org/tdlib/docs/td__json__client_8h.html). You'll be using this to send request to TDLib asynchronously. You will most often be using this API interacting with TDLib.

The `Send` API accepts a single param `data` of type `string` which is a JSON serialised string. You will need to refer to TDLib's documentation mentioned above on what APIs that can be consumed here.

The example below shows how the TDLib's [getChatHistory](https://core.telegram.org/tdlib/docs/classtd_1_1td__api_1_1get_chat_history.html) is consumed.
```
const payload = {
    "@type": "getChatHistory",
    "@extra": null,
    "chat_id": 123456789,
    "offset": 0,
    "limit": 10,
    "only_local": false
};

const data = JSON.stringify(payload);
await connection.send("Send", data);
```

You will notice that TDLib API reference, in this case [getChatHistory](https://core.telegram.org/tdlib/docs/classtd_1_1td__api_1_1get_chat_history.html) uses an **underscore** to end the name of their fields such as **chat_id_**, **from_message_id_**, etc. However, the JSON interface doesn't require that as shown in the example above.

### TDLib "td_receive" API
This is the Telegram Relay SignalR API which is equivalent to TDLib's **td_receive** API outlined in [JSON interface of TDLib](https://core.telegram.org/tdlib/docs/td__json__client_8h.html). You'll be using this to receive response from TDLib asynchronously. TDLib will most often be using this API to interact with your application.

The `ClientReceive` API accepts a single param `data` of type `string` which is a JSON serialised string. You will need to refer to TDLib's documentation mentioned above on what APIs that are received from TDLib here.

```
connection.on("ClientReceive", async data => {
var result = JSON.parse(data);
    switch (result["@type"]) {
        case "updateAuthorizationState":
            break;

        case "updateNewChat":
            break;
    }
});
```

### TDLib "setTdlibParameters" API
The TDLib [setTdLibParameters](https://core.telegram.org/tdlib/docs/classtd_1_1td__api_1_1set_tdlib_parameters.html) is required to be consumed very early in the application for TDLib initialisation. It must be implemented when the authorisation state is [authorizationStateWaitTdlibParameters](https://core.telegram.org/tdlib/docs/classtd_1_1td__api_1_1authorization_state_wait_tdlib_parameters.html).
```
connection.on("ClientReceive", async data => {
    var result = JSON.parse(data);
    switch (result["@type"]) {
        case "updateAuthorizationState":
            {
                switch (result["authorization_state"]["@type"]) {
                    case "authorizationStateWaitTdlibParameters":
                        {
                            const payload = {
                                "@type": "setTdlibParameters",
                                "api_id": 12345678,
                                "api_hash": "place_your_api_hash_here",
                                "device_model": "Web",
                                "system_language_code": "en",
                                "application_version": "1.0.0"
                            };

                            const data = JSON.stringify(payload);
                            await connection.send("Send", data);
                        }
                        break;

                    case "authorizationStateWaitPhoneNumber":
                        break;
                }
            }
            break;

        case "updateNewChat":
            break;
    }
});
```

A few of the [fields](https://core.telegram.org/tdlib/docs/classtd_1_1td__api_1_1tdlib_parameters.html) indicated in this API, do not need to be supplied because Telegram Relay will be furnishing them. Even if supplied, Telegram Relay will overwrite them. The fields that should be omitted are:

 1. system_version
 2. device_model
 3. database_directory
 4. files_directory
