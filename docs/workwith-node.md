 # Work with full node 
 
 This document is for developer who is very interested in transactions in every block, or order book, or account changes 
 or block fee charge and would like to built own down stream services of full node. Please refer to [run a full node](fullnode.md)
 if you still have't deploy a full node.
 
 
 You can enable the `publishKafka` or  `publishLocal` option in  `node-binary/fullnode/{network}/node/app.toml`. 
 The full node will publish messages that you are interested to local file or kafka, and you can consume them in your own
 apps. 

 The supported messages: 
 
 - executionResult: The trades that been filled, orders that changed and proposals that been submitted in this blocks.
 - books: the order book after this block ends.
 - account: the changed accounts after this blocks.
 - blockfee:  the block fee charged in this block.
 - transfers: the transfer transaction in this block.
 
 Accordingly, you need enable `publishOrderUpdates`, `publishOrderBook`, `publishAccountBalance`, `publishBlockFee`, `publishTransfer` 
 to choose your interested messages.  

 
## Publish to local file 
Enable the `publishLocal` option in `node-binary/fullnode/{network}/node/app.toml` to publish message to local file.
The output file is `{fullnode home}`/marketdata/marketdata.json. 
Each line of marketdata.json represents a message. All message is is encoded based on json.

- executionResult: 
```
{
	Height:    int64,
	Timestamp: int64, // milli seconds since Epoch
	NumOfMsgs: int,   // number of individual messages we published, consumer can verify messages they received against this field to make sure they does not miss messages
	Trades: {
	    NumOfMsgs: int,
        Trades:    []{
            Id:     string,
        	Symbol: string,
        	Price:  int64,
        	Qty:    int64,
        	Sid:    string,
        	Bid:    string,
        	Sfee:   string,
        	Bfee:   string,
        	SAddr:  string, // string representation of AccAddress
        	BAddr:  string // string representation of AccAddress
        },
	},
	Orders: {
	    NumOfMsgs: int,
        Orders:    []{
            	Symbol:               string,
            	Status:               uint8,
            	OrderId:              string,
            	TradeId:              string,
            	Owner:                string,
            	Side:                 int8,
            	OrderType:            int8,
            	Price:                int64,
            	Qty:                  int64,
            	LastExecutedPrice:    int64,
            	LastExecutedQty:      int64,
            	CumQty:               int64,
            	Fee:                  string,
            	OrderCreationTime:    int64,
            	TransactionTime:      int64,
            	TimeInForce:          int8,
            	CurrentExecutionType: uint8,
            	TxHash:               string
        }
	},
	Proposals: {
	    NumOfMsgs: int,
        Proposals: []{
            Id:     int64,
            Status: uint8
        }
	}
}

```
- books:
```
{
    Height:    int64,
    Timestamp: int64,
    NumOfMsgs: int,
    Books:     []{
       Symbol: string,
       Buys:   []{
            Price:   int64,
            LastQty: int64
       },
       Sells:  []{
            Price:   int64,
            LastQty: int64
       } 
    }
}
```

- account:
```
{
	Owner:    string,
	Fee:      string,
	Balances: []{
	    Asset:  string,
        Free:   int64,
        Frozen: int64,
        Locked: int64
	}

}
```

- blockfee
```
{
    	Height:     int64,
    	Fee:        string,
    	Validators: []string
}
```

- transfers
```
{
	Height:    int64,
	Num:       int,
	Timestamp: int64,
	Transfers: []{
	    From: string,
        To:   []{
            Addr:  string,
            Coins: []{
                Denom:  string, 
                Amount: int64 
            }
        }
	}
}
```



 
## Publish to Kafka 
The message is encoded based on `Avro` serialization system.
The schemas are shown as below: 

- executionResult:
```
{
    "type": "record",
    "name": "ExecutionResults",
    "namespace": "org.binance.dex.model.avro",
    "fields": [
        { "name": "height", "type": "long" },
        { "name": "timestamp", "type": "long" },
        { "name": "numOfMsgs", "type": "int" },
        { "name": "trades", "type": ["null", {
            "type": "record",
            "name": "Trades",
            "namespace": "org.binance.dex.model.avro",
            "fields": [
                { "name": "numOfMsgs", "type": "int" },
                { "name": "trades", "type": {
                    "type": "array",
                    "items":
                        {
                            "type": "record",
                            "name": "Trade",
                            "namespace": "org.binance.dex.model.avro",
                            "fields": [
                                { "name": "symbol", "type": "string" },
                                { "name": "id", "type": "string" },
                                { "name": "price", "type": "long" },
                                { "name": "qty", "type": "long"	},
                                { "name": "sid", "type": "string" },
                                { "name": "bid", "type": "string" },
                                { "name": "sfee", "type": "string" },
                                { "name": "bfee", "type": "string" },
                                { "name": "saddr", "type": "string" },
                                { "name": "baddr", "type": "string" }
                            ]
                        }
                    }
                }
            ]
        }], "default": null },
        { "name": "orders", "type": ["null", {
            "type": "record",
            "name": "Orders",
            "namespace": "org.binance.dex.model.avro",
            "fields": [
                { "name": "numOfMsgs", "type": "int" },
                { "name": "orders", "type": {
                    "type": "array",
                    "items":
                    {
                        "type": "record",
                        "name": "Order",
                        "namespace": "org.binance.dex.model.avro",
                        "fields": [
                            { "name": "symbol", "type": "string" },
                            { "name": "status", "type": "string" },
                            { "name": "orderId", "type": "string" },
                            { "name": "tradeId", "type": "string" },
                            { "name": "owner", "type": "string" },
                            { "name": "side", "type": "int" },
                            { "name": "orderType", "type": "int" },
                            { "name": "price", "type": "long" },
                            { "name": "qty", "type": "long" },
                            { "name": "lastExecutedPrice", "type": "long" },
                            { "name": "lastExecutedQty", "type": "long" },
                            { "name": "cumQty", "type": "long" },
                            { "name": "fee", "type": "string" }, 
                            { "name": "orderCreationTime", "type": "long" },
                            { "name": "transactionTime", "type": "long" },
                            { "name": "timeInForce", "type": "int" },
                            { "name": "currentExecutionType", "type": "string" },
                            { "name": "txHash", "type": "string" }
                        ]
                    }
                   }
                }
            ]
        }], "default": null },
        { "name": "proposals", "type": ["null", {
            "type": "record",
            "name": "Proposals",
            "namespace": "org.binance.dex.model.avro",
            "fields": [
                { "name": "numOfMsgs", "type": "int" },
                { "name": "proposals", "type": {
                    "type": "array",
                    "items":
                    {
                        "type": "record",
                        "name": "Proposal",
                        "namespace": "org.binance.dex.model.avro",
                        "fields": [
                            { "name": "id", "type": "long" },
                            { "name": "status", "type": "string" }
                        ]
                    }
                   }
                }
            ]
        }], "default": null }
    ]
}
```

- booksSchema:
```
{
    "type": "record",
    "name": "Books",
    "namespace": "com.company",
    "fields": [
        { "name": "height", "type": "long" },
        { "name": "timestamp", "type": "long" },
        { "name": "numOfMsgs", "type": "int" },
        { "name": "books", "type": {
            "type": "array",
            "items":
                {
                    "type": "record",
                    "name": "OrderBookDelta",
                    "namespace": "com.company",
                    "fields": [
                        { "name": "symbol", "type": "string" },
                        { "name": "buys", "type": {
                            "type": "array",
                            "items": {
                                "type": "record",
                                "name": "PriceLevel",
                                "namespace": "com.company",
                                "fields": [
                                    { "name": "price", "type": "long" },
                                    { "name": "lastQty", "type": "long" }
                                ]
                            }
                        } },
                        { "name": "sells", "type": {
                            "type": "array",
                            "items": "com.company.PriceLevel"
                        } }
                    ]
                }
            }, "default": []
        }
    ]
}
```

- accountSchema:
```
{
    "type": "record",
    "name": "Accounts",
    "namespace": "com.company",
    "fields": [
        { "name": "height", "type": "long" },
        { "name": "numOfMsgs", "type": "int" },
        { "name": "accounts", "type": {
            "type": "array",
            "items":
                {
                    "type": "record",
                    "name": "Account",
                    "namespace": "com.company",
                    "fields": [
                        { "name": "owner", "type": "string" },
                        { "name": "fee", "type": "string" },
                        { "name": "balances", "type": {
                                "type": "array",
                                "items": {
                                    "type": "record",
                                    "name": "AssetBalance",
                                    "namespace": "com.company",
                                    "fields": [
                                        { "name": "asset", "type": "string" },
                                        { "name": "free", "type": "long" },
                                        { "name": "frozen", "type": "long" },
                                        { "name": "locked", "type": "long" }
                                    ]
                                }
                            }
                        }
                    ]
                }
           }, "default": []
        }
    ]
}

```
   

- blockfeeSchema: 
```
{
    "type": "record",
    "name": "BlockFee",
    "namespace": "com.company",
    "fields": [
        { "name": "height", "type": "long"},
        { "name": "fee", "type": "string"},
        { "name": "validators", "type": { "type": "array", "items": "string" }}
    ]
}
```

- transfersSchema: 
```
{
    "type": "record",
    "name": "Transfers",
    "namespace": "com.company",
    "fields": [
        { "name": "height", "type": "long"},
        { "name": "num", "type": "int" },
        { "name": "timestamp", "type": "long" },
        { "name": "transfers",
          "type": {	
            "type": "array",
            "items": {
                "type": "record",
                "name": "Transfer",
                "namespace": "com.company",
                "fields": [
                    { "name": "from", "type": "string"},
                    { "name": "to", 
                        "type": {
                            "type": "array",
                            "items": {
                                "type": "record",
                                "name": "Receiver",
                                "namespace": "com.company",
                                "fields": [
                                    { "name": "addr", "type": "string" },
                                    { "name": "coins",
                                        "type": {
                                            "type": "array",
                                            "items": {
                                                "type": "record",
                                                "name": "Coin",
                                                "namespace": "com.company",
                                                "fields": [
                                                    { "name": "denom", "type": "string" },
                                                    { "name": "amount", "type": "long" }
                                                ]
                                            }
                                        }	
                                    }
                                ]
                            }
                        }
                    }
                ]
            }
          }	
        }
    ]
}
```