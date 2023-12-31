# Telegram Relay

**Telegram Relay** is a [SignalR](https://dotnet.microsoft.com/en-us/apps/aspnet/signalr) [Hub](https://learn.microsoft.com/en-us/aspnet/core/signalr/hubs) that is able to relay [Telegram](https://telegram.org/) Database Library ([TDLib](https://core.telegram.org/tdlib)) API to-and-fro the Telegram Server. This eliminates the needs to dabble in the intricacies of building the binaries and hooking it up. **Telegram Relay** is doing all the heavy lifting to integrate with TDLib while providing an easy to use interface for clients to connect with. Clients will be connecting with **Telegram Relay** mostly with SignalR and also REST at a minimal capacity. **Telegram Relay** will then be interacting with TDLib via the [JSON interface](https://core.telegram.org/tdlib/docs/td__json__client_8h.html). As such, clients will be sending/receiving data/payloads to/from **Telegram Relay** in JSON string.

Currently **Telegram Relay** is using TDLib version `1.8.12`

The **Telegram Relay** Client API is very easy to implement and has only a handful of methods. 99% of the API that you will be consuming are TDLib's which you can refer to on their documentations in their website [here](https://core.telegram.org/tdlib/docs/classtd_1_1td__api_1_1_function.html) and [here](https://core.telegram.org/tdlib/docs/classtd_1_1td__api_1_1_object.html).

## Usages

### Prerequisite

 1. You will need knowledge in ASP.NET Core SignalR Client API in order to be connected to the **Telegram Relay** SignalR Hub (Server). There are several [clients](https://learn.microsoft.com/en-us/aspnet/core/signalr/client-features) available but we will be using the [JavaScript client](https://learn.microsoft.com/en-us/aspnet/core/signalr/javascript-client) in the examples in this document. In order to follow the examples, set up the client first.
 2. You will need to apply for the Telegram's API access from [https://my.telegram.org/](https://my.telegram.org/).  
	&nbsp;&nbsp;&nbsp;&nbsp; Demo APIId: `94575`  
	&nbsp;&nbsp;&nbsp;&nbsp; Demo ApiHash: `a3406de8d171bb422bb6ddf3bbd800e2`
 3. You will need to understand the concepts used in TDLib. The [Getting Started Guide](https://core.telegram.org/tdlib/getting-started) is an excellent place for that. It is important to understand what does it mean by sending a request `synchronously` and `asynchronously`.

### Connecting to the SignalR Hub
First and foremost, we will be connecting to the SignalR Hub using the SignalR JavaScript Client. Connect to the SignalR Hub via the url `https://[host_name]/signalr-telegram`.
```javascript
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
The `connection` in the example above is SignalR Client's [`HubConnection`](https://learn.microsoft.com/en-us/javascript/api/@microsoft/signalr/hubconnection) that represents a connection to a SignalR Hub.
For more information on the JavaScript API, please refer to its [JavaScript API reference](https://learn.microsoft.com/en-us/javascript/api/@microsoft/signalr).

### Initialise the connection
Once connected to the Hub, we will need to initialise the connection for **Telegram Relay** to set up the necessary folders on the server. This will allow different Telegram accounts to be connected at the same time and their files and data to be hosted in their own respective space.

The `Init` API accepts a single param `accountName` of type `string`.
```javascript
const accountName = "+60123456789";
await connection.invoke("Init", accountName);
```
The `accountName` is the assigned Account Name that can uniquely identify the Telegram accounts used in **Telegram Relay** and can be any ASCII characters. It can be a Telegram account phone number, email address or user's name. After the authentication with TDLib using this `accountName` is successful, subsequent initialisation with the same `accountName` doesn't require authentication anymore and the Telegram account is ready to be accessed.  
**Note:** The following characters will be removed from the `accountName`: `;`, `/`, `\`, `?`, `:`, `"`, `@`, `&`, `*`, `=`, `+`, `$`, `,`, `<`, `>`, `|`, `[space]`.

### TDLib "td_send" API
This is the **Telegram Relay** SignalR API which is equivalent to TDLib's **td_send** API outlined in [JSON interface of TDLib](https://core.telegram.org/tdlib/docs/td__json__client_8h.html). We will be using this to send request to TDLib _asynchronously_ and most often be using this API to interact with TDLib.

The `Send` API accepts a single param `data` of type `string` which is a JSON serialised string. Refer to TDLib's documentation mentioned above on what APIs that can be consumed here.

The example below shows how the TDLib's [`getChatHistory`](https://core.telegram.org/tdlib/docs/classtd_1_1td__api_1_1get_chat_history.html) is consumed.
```javascript
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
The above example is calling the [`send`](https://learn.microsoft.com/en-us/javascript/api/@microsoft/signalr/hubconnection#@microsoft-signalr-hubconnection-send) method on SignalR Client's [`HubConnection`](https://learn.microsoft.com/en-us/javascript/api/@microsoft/signalr/hubconnection). This does not wait for a response from the server aka _calling a method asynchronously_.

Notice that TDLib API reference (in this case [`getChatHistory`](https://core.telegram.org/tdlib/docs/classtd_1_1td__api_1_1get_chat_history.html)) uses an **underscore** to end the name of their fields such as **chat_id_**, **from_message_id_**, etc. However, the JSON interface doesn't require that as shown in the example above.

### TDLib "td_execute" API
This is the **Telegram Relay** SignalR API which is equivalent to TDLib's **td_execute** API outlined in [JSON interface of TDLib](https://core.telegram.org/tdlib/docs/td__json__client_8h.html). We will be using this to send request to TDLib _synchronously_. Only a handful of APIs can be used _synchronously_, most of the them are called _asynchronously_ using the above **td_send** API. A request can be executed synchronously, only if it is documented with **_Can be called synchronously_**.

The example below shows how the TDLib's [`getFileMimeType`](https://core.telegram.org/tdlib/docs/classtd_1_1td__api_1_1get_file_mime_type.html) is consumed.
```javascript
const payload = {
    "@type": "getFileMime",
    "@extra": null,
    "file_name": "Telegram/uploads/cats.mp4"
};

const data = JSON.stringify(payload);
const result = await connection.invoke("Execute", data);
console.log(result.text); // video/mp4
```
Note that the above example is calling the [`invoke`](https://learn.microsoft.com/en-us/javascript/api/@microsoft/signalr/hubconnection#@microsoft-signalr-hubconnection-invoke) method on SignalR Client's [`HubConnection`](https://learn.microsoft.com/en-us/javascript/api/@microsoft/signalr/hubconnection). This is required to wait and receive a result from the server aka _calling a method synchronously_.

### TDLib "td_receive" API
This is the **Telegram Relay** SignalR API which is equivalent to TDLib's **td_receive** API outlined in [JSON interface of TDLib](https://core.telegram.org/tdlib/docs/td__json__client_8h.html). We will be using this to receive response from TDLib _asynchronously_. TDLib will most often be using this API to interact with our application.

The `ClientReceive` API accepts a single param `data` of type `string` which is a JSON serialised string. Refer to TDLib's documentation mentioned above on what APIs that are received from TDLib here.

```javascript
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
The TDLib [`setTdLibParameters`](https://core.telegram.org/tdlib/docs/classtd_1_1td__api_1_1set_tdlib_parameters.html) is required to be consumed very early in the application for TDLib initialisation. It must be implemented when the authorisation state is [`authorizationStateWaitTdlibParameters`](https://core.telegram.org/tdlib/docs/classtd_1_1td__api_1_1authorization_state_wait_tdlib_parameters.html).
```javascript
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

A few of the [fields](https://core.telegram.org/tdlib/docs/classtd_1_1td__api_1_1tdlib_parameters.html) indicated in this API do not need to be supplied because **Telegram Relay** will be furnishing them. Even if supplied, **Telegram Relay** will overwrite them. The fields that should be omitted are:

 1. system_version
 2. device_model
 3. database_directory
 4. files_directory

### Sending files from local device
The TDLib [`inputFileLocal`](https://core.telegram.org/tdlib/docs/classtd_1_1td__api_1_1input_file_local.html) API is used whenever we need to include a file in our local device and is always used as part of other APIs. For example, the [`inputMessagePhoto`](https://core.telegram.org/tdlib/docs/classtd_1_1td__api_1_1input_message_photo.html) API is used to send a photo message and the photo can be selected from a local device using `inputFileLocal`. However, **Telegram Relay** is unable to send the file located on our local device to Telegram Server. We will have to explicitly upload the file to **Telegram Relay** server first and call the appropriate API referencing the file we have just uploaded.

#### Upload files
**Telegram Relay** provides a REST API for files to be uploaded to a temporary location on the server that is defined as follows:

<table>
	<tr>
		<th>Endpoint</th>
		<td><a name="uploadfiles-endpoint">https://[host_name]/api/Telegram/UploadFiles</a></td>
	</tr>
	<tr>
		<th>Verb</th>
		<td>POST</td>
	</tr>
	<tr>
		<th>Headers</th>
		<td>
			<a name="x-connection-id"><b>X-Connection-Id</b></a><br />
			The <a href="https://learn.microsoft.com/en-us/javascript/api/%40microsoft/signalr/hubconnection#@microsoft-signalr-hubconnection-connectionid"><code>connectionId</code></a> of SignalR Client's <a href="https://learn.microsoft.com/en-us/javascript/api/@microsoft/signalr/hubconnection"><code>HubConnection</code></a> (must be connected to the SignalR Hub in order to retrieve this).
			<br /><br />
			<b>Content-Type</b><br />
			<code>multipart/form-data;boundary=[some_value]</code><br />
			The spec at <a href="https://tools.ietf.org/html/rfc2046#section-5.1">https://tools.ietf.org/html/rfc2046#section-5.1</a> states that 70 characters is a reasonable limit for <code>[some_value]</code>.
		</td>
	</tr>
	<tr>
		<th>Body</th>
		<td>The files to be uploaded according to the format defined in <a href="https://tools.ietf.org/html/rfc2046#section-5.1">RFC2046</a>.<br />
		Files with the following extensions are forbidden: <code>.bat</code>, <code>.com</code>, <code>.exe</code>, <code>.vb</code>, <code>.vbe</code>, <code>.vbs</code>, <code>.vsmacros</code>.
		</td>
	</tr>
	<tr>
		<th>Returned Result</th>
		<td>
			HTTP Status Code <code>400 Bad Request</code> when the <a href="#x-connection-id"><code>X-Connection-Id</code></a> provided is invalid (not connected to the SignalR Hub).<br />
			HTTP Status Code <code>201 Created</code> and JSON string with the following data structure is returned on success:
			<table>
				<tr>
					<th>Field Name</th>
					<th>Data Type</th>
					<th>Description</th>
				</tr>
				<tr>
					<td>uploadedFileCount</td>
					<td>int32</td>
					<td>The number of files that are successfully uploaded.</td>
				</tr>
				<tr>
					<td>totalSizeUploaded</td>
					<td>int64</td>
					<td>The total size in bytes of the uploaded files.</td>
				</tr>
				<tr>
					<td>uploadedFiles</td>
					<td>Array&lt;<a href="#uploadedfile"><code>UploadedFile</code></a>&gt;</td>
					<td>The list of files that are successfully uploaded.</td>
				</tr>
				<tr>
					<td>notUploadedFiles</td>
					<td>Array&lt;string&gt;</td>
					<td>The list of file names that are failed to be uploaded.</td>
				</tr>
			</table>
			<br />
			<a name="uploadedfile">The data structure of <code>UploadedFile</code> mentioned above is as follows:</a>
			<table>
				<tr>
					<th>Field Name</th>
					<th>Data Type</th>
					<th>Description</th>
				</tr>
				<tr>
					<td><a name="uploadedfile-fileid">fileId</a></td>
					<td>string</td>
					<td>The ID of the uploaded file. Use this in TDLib APIs to refer to this uploaded file.</td>
				</tr>
				<tr>
					<td>originalFileName</td>
					<td>string</td>
					<td>
						The original file name of the uploaded file.<br/>
						<strong>Telegram Relay</strong> renames the uploaded file on the server.
					</td>
				</tr>
				<tr>
					<td>uploadedFilePath</td>
					<td>string</td>
					<td>The path that is used to access the uploaded file which has been renamed. </td>
				</tr>
				<tr>
					<td>uploadedFileSize</td>
					<td>int64</td>
					<td>The file size in bytes of the uploaded file.</td>
				</tr>
			</table>
		</td>
	</tr>
</table>

##### Example
The following is an example of uploading two files using the API above with [Postman](https://www.postman.com/):  
**Note:** The API endpoint used in the example below is not the latest. Please refer to the specification [above](#uploadfiles-endpoint).

![image](https://github.com/bytedash-dev/telegram-relay/assets/143246263/ed55ca80-efa4-40c1-979b-d782a1f10aed)

![image](https://github.com/bytedash-dev/telegram-relay/assets/143246263/70b67b54-feb2-42b8-8343-9e9c6493e1da)

#### Send files as messages

After uploading the files to **Telegram Relay** server, we can then proceed to call the relevant TDLib API to act upon those files.
We can refer to the uploaded file using the [`fileId`](#uploadedfile-fileid) of the [`UploadedFile`](#uploadedfile).

The example below shows how to send the two uploaded files above as photos in two separate Telegram messages using the [`sendMessage`](https://core.telegram.org/tdlib/docs/classtd_1_1td__api_1_1send_message.html) API.
```javascript
// first photo
const payload1 = {
    "@type": "sendMessage",
    "@extra": null,
    "chat_id": 123456789,
    "input_message_content": {
        "@type": "inputMessagePhoto",
        "@extra": null,
        "width": 500,
        "height": 300,
        "caption": {
            "@type": "formattedText",
            "@extra": null,
            "text": "You can send a caption along with the Photo or just pass 'null' if you do not want a caption."
        },
        "photo": {
            "@type": "inputFileLocal",
            "@extra": null,
            "path": "iwdruscu.jpg" // this is the fileId of the UploadedFile
        }
    }
};

const data1 = JSON.stringify(payload1);
await connection.send("Send", data1);


// second photo
const payload2 = {
    "@type": "sendMessage",
    "@extra": null,
    "chat_id": 123456789,
    "input_message_content": {
        "@type": "inputMessagePhoto",
        "@extra": null,
        "width": 400,
        "height": 1000,
        "caption": null,
        "photo": {
            "@type": "inputFileLocal",
            "@extra": null,
            "path": "w22k2pez.jpg" // this is the fileId of the UploadedFile
        }
    }
};

const data2 = JSON.stringify(payload2);
await connection.send("Send", data2);
```

### Delete account
**Telegram Relay** provides a REST API to delete an existing account from the server. This doesn't delete the Telegram account itself, but only removes its authentication data and temporary files from **Telegram Relay** server. The account **MUST** not be in use. Upon successful call, subsequent connection and initialisation to the same Account Name will require the Account to be authenticated again.

<table>
	<tr>
		<th>Endpoint</th>
		<td>
			https://[host_name]/api/Telegram/RemoveAccount/<strong>{accountName}</strong><br /><br />
			<strong>accountName</strong> is the Account Name used during <a href="#initialise-the-connection">Initialisation</a>.
		</td>
	</tr>
	<tr>
		<th>Verb</th>
		<td>DELETE</td>
	</tr>
	<tr>
		<th>Returned Result</th>
		<td>
			HTTP Status Code <code>400 Bad Request</code> if the Account is in use.<br />
			HTTP Status Code <code>204 No Content</code> on success.
		</td>
	</tr>
</table>

## Sample Code
Below is a short program on sending and receiving a text messages The flow of the program is marked with a comment and number to indicate the step-by-step taken by the program, eg: `// Flow Step #01`.
```javascript
const telegramPhoneNumber = "+60123456789"; // Enter your Telegram phone number with + prefix, eg: +60123456789

// It's recommended to use a different Telegram account on another device to send you a message first so the chat will appear at the top.
const nameOfTelegramAccount = "John Doe"; // The name of the Telegram account that sent you a message just now. It's usually in the format of "{FirstName} {LastName}"

const sampleTextMessage = "Hello World"; // Test message to send.

// Flow Step #01
const connection = new window.signalR.HubConnectionBuilder()
    .withUrl("https://telegram.bytedash.com.my/signalr-telegram")
    .configureLogging(signalR.LogLevel.Information)
    .build();
    

let chatId; // this is to store the chat_id that we are interested in.

connection.on("ClientReceive", async data => {
    var result = JSON.parse(data);
    switch (result["@type"]) {
        case "updateAuthorizationState":
            {
                const authorizationType = result["authorization_state"]["@type"];
                switch (authorizationType) {
                    case "authorizationStateWaitTdlibParameters": // Flow Step #03
                        {
                            const payload = {
                                "@type": "setTdlibParameters",
                                "api_id": 94575,
                                "api_hash": "a3406de8d171bb422bb6ddf3bbd800e2",
                                "device_model": "Web",
                                "system_language_code": "en",
                                "application_version": "1.0.0"
                            };
                            
                            const data = JSON.stringify(payload);
                            await connection.send("Send", data);
                        }
                        break;
    
                    case "authorizationStateWaitPhoneNumber": // Flow Step #04 - If you have successfully authenticated previously, this step will be skipped
                        {
                            const payload = {
                                "@type": "setAuthenticationPhoneNumber",
                                "phone_number": telegramPhoneNumber
                            };
                            
                            const data = JSON.stringify(payload);
                            await connection.send("Send", data);
                        }
                        break;
    
                    case "authorizationStateWaitCode": // Flow Step #05 - If you have successfully authenticated previously, this step will be skipped
                        {
                            let authorizationCode = "12345"; // Enter the authorization code you received on your Telegram account.

                            const payload = {
                                "@type": "checkAuthenticationCode",
                                "code": authorizationCode
                            };

                            const data = JSON.stringify(payload);
                            await connection.send("Send", data);
                        }
                        break;

                    case "authorizationStateReady": // Flow Step #06 - This indicates that the application has passed authentication and ready for use.
                        {
                            // Let's retrieve a list of existing chats.
                            
                            console.log("Application is ready. Retrieving chat list. This might take a while.");
                            
                            const payload = {
                                "@type": "loadChats",
                                "limit": 10
                            };
                            
                            const data = JSON.stringify(payload);
                            await connection.send("Send", data);
                        }
                        break;
                  }
              }
            break;
    
        case "updateNewChat": // Flow Step #07
            {
                // Here is to receive the response of "loadChats" sent above.
                // If you have 10 chats, this part of the code will be called 10 times, so we need to only filter the chat that we're interested in.
                // Depending on how big your chat list is, it might take quite some time before reaching here. In the mean time, it might appear to be not doing anything.
                
                console.log("Receiving chat.");
                
                const chat = result.chat;
                
                if (chat.title == nameOfTelegramAccount) { // only do something if it's the chat that we want.
                    chatId = chat.id; // save the Chat ID for later use.
                    
                    // Now try sending a text message with the content "Hello World" to this chat
                    
                    console.log("Sending test text message.");
                    
                    const payload = {
                        "@type": "sendMessage",
                        "@extra": null,
                        "chat_id": chatId,
                        "input_message_content": {
                            "@type": "inputMessageText",
                            "@extra": null,
                            "text": {
                                "@type": "formattedText",
                                "@extra": null,
                                "text": sampleTextMessage
                            }
                        }
                    };
                    
                    const data = JSON.stringify(payload);
                    await connection.send("Send", data);
                    
                    // The other Telegram account would receive the text message.
                    
                    // Flow Step #08
                    // Now, send a text message using the other Telegram account to this account.
                }
            }
            break;
            
        case "updateNewMessage": // Flow Step #09
            {
                // Here is where you will receive messages.
                
                console.log("Receiving text message.");
                
                const message = result.message;
                
                if (message["chat_id"] == chatId) { // only interested in the messages of the chatId we saved.
                    // Here will be called twice. The first one for the message that you sent out (outgoing message).
                    // The second time is when the other Telegram account sends you a message (incoming message).
                    console.log(message);
                }
            }
            break;
            
        case "error":
            {
                // In the event of error.
                console.error(result);
            }
            break;
    }
});

// Flow Step #02
await connection.start(); // connect to the Hub

const accountName = telegramPhoneNumber;
await connection.invoke("Init", accountName);


// Flow Step #10 - run the uncommented code below to stop and dispose the connection.
//await connection.stop(); // close the connection to the Hub
```

## Important Notes

### TDLib APIs are case-sensitive
As such the JSON-serialised string sent must be case-sensitive as well.

### JSON-serialised string MUST not contain whitespaces.
Although whitespaces are allowed in JSON string such as
```javascript
{ "@type": "exampleType", "name": "hello world" }
```

For performance reasons, the JSON string sent to **Telegram Relay** Hub **MUST** not contain whitespaces.
```javascript
{"@type":"exampleType","name":"hello world"}
```
All JSON serialiser library should be able to cater for this by default.

### JSON-serialised string for TDLib APIs MUST start with `@type`
Although fields order/sequence doesn't matter in JSON such as
```javascript
{"name":"hello world","@type":"exampleType","description":"This is a description"}
```

For performance reasons, the field `@type` **MUST** appear before any other fields. The order of other fields don't matter.
```javascript
{"@type":"exampleType","name":"hello world","description":"This is a description"}
```

This applies only to the main API and not the nested ones. Consider this example.
```javascript
const payload = {
    "@type": "sendMessage",
    "@extra": null,
    "chat_id": 123456789,
    "input_message_content": {
        "@extra": null,
        "width": 500,
        "height": 300,
        "@type": "inputMessagePhoto",
        "caption": {
            "@extra": null,
            "text": "You can send a caption along with the Photo or just pass 'null' if you do not want a caption."
            "@type": "formattedText",
        },
        "photo": {
            "@extra": null,
            "path": "iwdruscu.jpg" // this is the fileId of the UploadedFile
            "@type": "inputFileLocal",
        }
    }
};
```
Only the `@type` of `sendMessage` must be at the top, other `@type`s of nested objects are not required to be bound by this requirement.

The JSON serialiser library must be configured to account for this requirement.

### Downloaded/Uploaded files are temporary
The files downloaded/uploaded using TDLib and REST APIs are valid and accessible only during the same connection to the **Telegram Relay** SignalR Hub. Once disconnected, those files are purged. The files are no longer accessible to subsequent sessions of the same Telegram Account and has to be re-downloaded again. In order to prevent that, the application that we build has to keep a cache to survive re-connection.
