# OneApi Pharo Smalltalk Client

## Prerequisites

This library is written and tested on [Pharo](http://www.pharo-project.org/home) 1.4.1 (summer release) but you are encouraged to try it on newer versions.
These are file-outs of packages which can be filed-in into the image using File Browser component. Before loading the source code make sure to load the dependencies!

- [Zinc](https://github.com/svenvc/zinc) HTTP client - the library depends on Zinc HTTP client components (examples use also HTTP server) which should already be present in this Pharo release. You might want to update to the newest version as it is constantly improving.
- [NeoJSON] (https://github.com/svenvc/docs/blob/master/neo/neo-json-paper.md) - JSON is handled using this small library which **is not** present in this release. You can use configuration browser to load it.

## Usage examples

The library includes set of unit tests that you can look into for detailed usage examples.


### Initialization

Before you begin sending messages you have to create a client instance with a predefined configuration. Set your Parseco account settings in `ParsecoConfiguration class >> setDefaults`

    smsClient := ParsecoClient new configuration: ParsecoConfiguration default.

### Login

Login to obtain authentication token form the server:

    smsClient login.

### Send a single SMS:

    request := ParsecoSmsRequest new message: 'Hello from Pharo!'; senderAddress: '987654321'; address: '12391000'.
    smsClient send: request.

### Sending muliple SMSes and receiveing delivery reports:

To send multiple SMSes just pass a collection of addresses.
This method makes periodical HTTP requests to the OneApi server, which may not be optimal.

    request := ParsecoSmsRequest new message: 'Hello from Pharo!'; senderAddress: '987654321'; address: { '12391000'. '12391001'. '12391002' }.
    smsClient send: request onDelivery: [ :deliveryInfo |
        deliveryInfo statusesDo: [ :address :status | self logCr: address, ': ', status ] ].

### Receiving delivery reports on your HTTP server

Having the delivery report pushed to your HTTP server is much better (resource-wise) than pulling.

    | request server onDeliveryReport ip port path |
        port := 8080.
        ip := 'your-public-ip-address'.
        path := '/'.

        "You shouldn't start a new HTTP server instance for every message you send, of course."
        server := ZnServer startOn: port.

        onDeliveryReport := [ :httpRequest |
            | deliveryInfo |
            deliveryInfo := ParsecoDeliveryNotification on: httpRequest entity readStream.
            self logCr: deliveryInfo address, ': ', deliveryInfo status.
            [ 1 second asDelay wait. server stop ] fork.
            ZnResponse ok: (ZnStringEntity text: 'OK')
        ].
        server delegate: (ZnValueDelegate with: onDeliveryReport).

        request := ParsecoSmsRequest new
            message: 'Hello from Pharo!';
            notifyURL: 'http://', ip, ':', port asString, path;
            senderAddress: '987654321';
            address: { '12391000' }.
        smsClient send: request.

### Performing an HLR:

    hlrResponse := smsClient hlrFor: '12391000'.
    self logCr: hlrResponse currentRoaming.

### It's always nice to clean up behind you:

    smsClient logout.
    smsClient close. "Closes the underlying TCP socket, which is keep open for HTTP1.1 keep-alive feature."


License
-------

This library is licensed under the [Apache License, Version 2.0](http://www.apache.org/licenses/LICENSE-2.0)