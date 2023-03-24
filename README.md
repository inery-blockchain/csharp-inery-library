### Description

C# library that enables interaction with the blockchain via API. The library is used to connect to a specific blockchain network through a unique ChainId and HTTP node URL address (HttpEndpoint). The library also has methods for reading information from the blockchain, such as account information (GetAccount), block information (GetBlock), and table information (GetTableRows). It also supports reading data from the table in JSON format and provides support for mapping fields from the API. The library also supports transaction signing via a customizable SignProvider, which can be tailored to different implementations depending on the available keys and specific usage requirements.

#### Configuration

In order to interact with inery blockchain you need to create a new instance of the **Inery** object class with a **IneryConfigurator**.

Example:

```clibrary
Inery inery = new Inery(new IneryConfigurator()
{    
    HttpEndpoint = "https://nodes.inery42.io", //Mainnet
    ChainId = "aca376f206b8fc25a6ed44dbdc66547c36c6c33e3a119ffbeaef943642f0e906",
    ExpireSeconds = 60,
    SignProvider = new DefaultSignProvider("myprivatekey")
});
```
* HttpEndpoint refers to the URL location of a node that provides access to a blockchain's API via the HTTP or HTTPS protocols.
* ChainId - Unique identifier for a specific blockchain network that you want to connect to. It is used to ensure that you are interacting with the correct network, as different networks can have different rules and protocols. 
* ExpireInSeconds - Parameter used in transactions to specify the number of seconds before the transaction expires. The time is based on the clock of the nodineryd server that is processing the transaction. An unexpired transaction that may have had an error is a liability until the expiration is reached, so this time should be kept brief to minimize potential risks. 
* SignProvider - A component that provides a signature implementation to handle available keys and sign transactions. It is used to sign transactions within the library. The SignProvider can be customized to use different signature implementations depending on the available keys and the specific requirements of the use case.

#### Api read methods

- **GetAccount** call
```clibrary
var result = await inery.GetAccount("myaccountname");
```
Returns:
```clibrary
class GetAccountResponse
{
    string account_name;
    UInt32 head_block_num;
    DateTime head_block_time;
    bool privileged;
    DateTime last_code_update;
    DateTime created;
    Int64 ram_quota;
    Int64 net_weight;
    Int64 cpu_weight; 
    Resource net_limit; 
    Resource cpu_limit;
    UInt64 ram_usage;
    List<Permission> permissions;
    RefundRequest refund_request;
    SelfDelegatedBandwidth self_delegated_bandwidth;
    TotalResources total_resources;
    VoterInfo voter_info;
}
```

- **GetInfo** call
```clibrary
var result = await inery.GetInfo();
```
Returns:
```clibrary
class GetInfoResponse 
{ 
    string server_version;
    string chain_id;
    UInt32 head_block_num;
    UInt32 last_irreversible_block_num;
    string last_irreversible_block_id;
    string head_block_id;
    DateTime head_block_time;
    string head_block_producer;
    string virtual_block_cpu_limit;
    string virtual_block_net_limit;
    string block_cpu_limit;
    string block_net_limit;
}
```

- **GetBlock** call
```clibrary
var result = await inery.GetBlock("blockIdOrNumber");
```
Returns:
```clibrary
class GetBlockResponse
{
    DateTime timestamp;
    string producer;
    UInt32 confirmed;
    string previous;
    string transaction_mroot;
    string action_mroot;
    UInt32 schedule_version;
    Schedule new_producers;
    List<Extension> block_extensions;
    List<Extension> header_extensions;
    string producer_signature;
    List<TransactionReceipt> transactions;
    string id;
    UInt32 block_num;
    UInt32 ref_block_prefix;
}
```

- **GetTableRows** call
    * Json - specify that the output format should be JSON
    * Code - accountName of the contract to search for table rows
    * Scope - scope text segmenting the table set
    * Table - table name
    * IndexPosition - 1 - primary (first), 2 - secondary index (in order defined by   * multi_index), 3 - third index, etc
    * KeyType - Type of the index chosen, ex: i64
    * LowerBound - lower bound for the selected index value
    * UpperBound - upper bound for the selected index value
    * Limit - maximum number of rows to return
    * EncodeType - specify encoding type for binary data, ex: dec, hex
    * Reverse - flag to reverse result order
    * ShowPayer - flag to indicate if the RAM payer should be displayed or not.

```clibrary
var result = await inery.GetTableRows(new GetTableRowsRequest() {
    json = true,
    code = "inery.token",
    scope = "INE",
    table = "stat"
});
```

Returns:

```clibrary
class GetTableRowsResponse
{
    List<object> rows
    bool?        more
}
```

Using generic type

```clibrary
/*JsonProperty helps map the fields from the api*/
public class Stat
{
    public string issuer { get; set; }
    public string max_supply { get; set; }
    public string supply { get; set; }
}

var result = await Inery.GetTableRows<Stat>(new GetTableRowsRequest()
{
    json = true,
    code = "inery.token",
    scope = "INE",
    table = "stat"
});
```

Returns:

```clibrary
class GetTableRowsResponse<Stat>
{
    List<Stat> rows
    bool?      more
}
```

- **GetTableByScope** call
    * Code - Account name of the smart contract to search for tables.
    * Table - Name of the table to filter.
    * LowerBound - Lower bound of the scope (optional).
    * UpperBound - Upper bound of the scope (optional).
    * Limit - Maximum number of rows to return.
    * Reverse - Flag to reverse the result order.

```clibrary
var result = await inery.GetTableByScope(new GetTableByScopeRequest() {
   code = "inery.token",
   table = "accounts"
});
```

Returns:

```clibrary
class GetTableByScopeResponse
{
    List<TableByScopeResultRow> rows
    string more
}

class TableByScopeResultRow
{
    string code;
    string scope;
    string table;
    string payer;
    UInt32? count;
}
```

- **GetActions** call
    * accountName - accountName to get actions history
    * pos - a absolute sequence positon -1 is the end/last action
    * offset - the number of actions relative to pos, negative numbers return [pos-offset,pos), positive numbers return [pos,pos+offset)

```clibrary
var result = await inery.GetActions("myaccountname", 0, 10);
```

Returns:

```clibrary
class GetActionsResponse
{
    List<GlobalAction> actions;
    UInt32 last_irreversible_block;
    bool time_limit_exceeded_error;
}
```

#### Create Transaction

**NOTE: On WEBGL Unity exports, it is not possible to utilize anonymous objects or properties as action data.
Use data as dictionary or strongly typed objects with fields.**

```clibrary
var result = await inery.CreateTransaction(new Transaction()
{
    actions = new List<Api.v1.Action>()
    {
        new Api.v1.Action()
        {
            account = "inery.token",
            authorization = new List<PermissionLevel>()
            {
                new PermissionLevel() {actor = "tester112345", permission = "active" }
            },
            name = "transfer",
            data = new { from = "tester112345", to = "tester212345", quantity = "0.0001 INE", memo = "hello crypto world!" }
        }
    }
});
```

Data can also be a Dictionary with key as string. The dictionary value can be any object with nested Dictionaries

```clibrary
var result = await inery.CreateTransaction(new Transaction()
{
    actions = new List<Api.v1.Action>()
    {
        new Api.v1.Action()
        {
            account = "inery.token",
            authorization = new List<PermissionLevel>()
            {
                new PermissionLevel() {actor = "tester112345", permission = "active" }
            },
            name = "transfer",
            data = new Dictionary<string, string>()
            {
                { "from", "tester112345" },
                { "to", "tester212345" },
                { "quantity", "0.0001 INE" },
                { "memo", "hello crypto world!" }
            }
        }
    }
});
```

Returns the transactionId
#### Custom SignProvider

Is also possible to implement your own **ISignProvider** to customize how the signatures and key handling is done.

Example:

```clibrary
/// <summary>
/// Signature provider implementation that uses a private server to hold keys
/// </summary>
class SuperSecretSignProvider : ISignProvider
{
   /// <summary>
   /// Get available public keys from signature provider server
   /// </summary>
   /// <returns>List of public keys</returns>
   public async Task<IEnumerable<string>> GetAvailableKeys()
   {
        var result = await HttpHelper.GetJsonAsync<SecretResponse>("https://supersecretserver.com/get_available_keys");
        return result.Keys;
   }
   
   /// <summary>
   /// Sign bytes using the signature provider server
   /// </summary>
   /// <param name="chainId">INERY Chain id</param>
   /// <param name="requiredKeys">required public keys for signing this bytes</param>
   /// <param name="signBytes">signature bytes</param>
   /// <param name="abiNames">abi contract names to get abi information from</param>
   /// <returns>List of signatures per required keys</returns>
   public async Task<IEnumerable<string>> Sign(string chainId, List<string> requiredKeys, byte[] signBytes)
   {
        var result = await HttpHelper.PostJsonAsync<SecretSignResponse>("https://supersecretserver.com/sign", new SecretRequest {
            chainId = chainId,
            RequiredKeys = requiredKeys,
            Data = signBytes
        });
        return result.Signatures;
   }
}

// create new Inery client instance using your custom signature provider
Inery inery = new Inery(new IneryConfigurator()
{
    SignProvider = new SuperSecretSignProvider(),
    HttpEndpoint = "https://nodes.inery42.io", //Mainnet
    ChainId = "aca376f206b8fc25a6ed44dbdc66547c36c6c33e3a119ffbeaef943642f0e906"
});
```
#### CombinedSignersProvider

Is also possible to combine multiple signature providers to complete all the signatures for a transaction

Example:

```clibrary
Inery inery = new Inery(new IneryConfigurator()
{    
    HttpEndpoint = "https://nodes.inery42.io", //Mainnet
    ChainId = "aca376f206b8fc25a6ed44dbdc66547c36c6c33e3a119ffbeaef943642f0e906",
    ExpireSeconds = 60,
    SignProvider = new CombinedSignersProvider(new List<ISignProvider>() {
       new SuperSecretSignProvider(),
       new DefaultSignProvider("myprivatekey")
    }),
});
```

