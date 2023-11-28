Introduction
"Indexing" refers to the process of synchronizing block data from AElf blockchain nodes to a locally centralized ElasticSearch environment for storage. This system then provides various data interfaces. Whether you are a dApp developer looking to build exciting applications on the AElf blockchain or just curious about how the AElf node's scanning system operates, this document is suitable for you.
Overall Workflow
The overall workflow of indexer, starting from the AElf nodes, pushing block data to the DApp, getting the desired on-chain data. 
[Image]
1. AElf Node Push
The AElf Indexer enhances functionality for AElf nodes, enabling automatic asynchronous or synchronous transmission of historical or latest block information to the RabbitMQ message queue. The AElf Indexer's storage module then receives and processes the relevant block data.
2. Indexer Storage
Upon receiving block data from RabbitMQ, the Indexer storage module identifies and processes the data, identifying any forked blocks. During this process, some auxiliary data is stored in MongoDB, but ultimately, all block data (excluding forks) is stored in Elasticsearch. The data is organized into different indices based on the structures of Block, Transaction, and Logevent.
3. Indexer Subscription and Push
Subscription: Consumers of the AElf Indexer can initiate subscriptions for block-related information through the subscription API. Currently, subscriptions primarily support block height, block transactions, and block transaction event dimensions, especially subscribing based on transaction events, which is applicable in various scenarios. After making a subscription API request with a client ID, a subscription version is returned, which is noted and later written into the client interface plugin developed subsequently.
Push: Upon receiving a subscription request, the AElf Indexer subscription and push module fetches data from Elasticsearch based on the subscription requirements and streams the data to the Kafka message queue.
4. Indexer Client
The Indexer client receives subscribed block data from Kafka and passes it to the interface plugin for processing. Interface plugins are developed separately and handle various transactions and events within blocks, storing the processed result set in a specific Elasticsearch index. Based on requirements, GraphQL interfaces are defined, utilizing Elasticsearch as a data source to develop business logic and expose the data externally.
5. DApp Integration
Within the DApp, desired data can be requested by directly calling the GraphQL interface exposed by the AElf Indexer client interface plugin, based on the client ID.
Build Indexer
Step 1. Subscribe block information
Obtaining Authentication Authorization
The demand side (DApp) needs to contact the indexer system administrator first to get the client ID and key assigned by the indexer, which looks like this
  {
    "ClientId": "BingoGame_DApp",
    "ClientSecret": "1q2w3e*"
  }
Each DApp that requires indexer should apply for a corresponding client ID and key, which will be valid for a long time.
Upon obtaining the client ID and key pre-allocated by the AElf Indexer, you can initiate an authentication authorization request to the Indexer, obtaining an authentication token (Token) upon successful verification.
Post request address：http://URL:{Port}/connect/token
The URL and port correspond to the server address where the AElf Indexer AuthServer service is located. Please contact AElf to get it.
Request Mode:：x-www-from-urlencoded
Request Body
grant_type:client_credentials
scope:AElfIndexer
client_id:BingoGame_DApp
client_secret:1q2w3e*
Response：
{
    "access_token": "eyJhbGciOiJSUzI1NiIsImtpZCI6IkY1RDFFRjAzRDlEMEU2MTI1N0ZFMTc0ODVBRkI2RjUzNDc0QzJEQjkiLCJ4NXQiOiI5ZEh2QTluUTVoSlhfaGRJV3Z0dlUwZE1MYmsiLCJ0eXAiOiJhdCtqd3QifQ.eyJvaV9wcnN0IjoiQUVsZkluZGV4ZXJfREFwcCIsImNsaWVudF9pZCI6IkFFbGZJbmRleGVyX0RBcHAiLCJvaV90a25faWQiOiI5MTljZmYzOC0xNWNhLTJkYWUtMzljYi0zYTA4YzdhZjMxYzkiLCJhdWQiOiJBRWxmSW5kZXhlciIsInNjb3BlIjoiQUVsZkluZGV4ZXIiLCJleHAiOjE2NzM3OTEwOTYsImlzcyI6Imh0dHA6Ly9sb2NhbGhvc3Q6ODA4My8iLCJpYXQiOjE2NzM3ODc0OTZ9.aABo_opBCiC3wePnIJpc6y3E4-nj50_WP93cYoYwxRGOxnXIq6LXz_r3-V_rmbzbxL3TbQvWQVuCcslF_rUJTMo6e6WC1ji5Ec9DtPpGbOOOvYALNhgOiP9p9TbzVubxHg7WdT6OEDLFihh4hsxtVBTK5_z8YXTa7fktLqve5Bd2eOpjb1TnQC7yZMwUvhnvQrjxuK9uRNxe9ODDt2EIcRhIQW5dQ-SDXpVoNfypY0GxQpuyHjwoJbtScJaX4HfHbh0Fis8EINOwpJr3-GKtcS6F4-t4FyOWMVW19y1_JAoCKTUlNy__htpdMOMQ-5nmFYYzlNr27LSOC_cylXz4lw",
    "token_type": "Bearer",
    "expires_in": 3593
}
The access_token is the authentication token. It is required when making specific subscription requests to the Subscription API.
Send Subscription 
By sending a request to the Subscription API, you inform the Indexer system that your DApp needs to subscribe to specific blocks, transactions, or events. Subsequently, when the interface plugin subscribes to the Indexer client, the Indexer system filters out the specified blocks/transactions/events and pushes them to the corresponding interface plugin.
Post request address：http://URL:{Port}/api/app/subscription
The URL and port correspond to the server address where the AElf Indexer HttpAPI service is located. Please contact AElf to get it.
Request Mode：raw
Request Header
Authorization
Bearer {access_token}
Request Body
[
    {
        "chainId": "tDVV",
        "startBlockNumber": 48532699,
        "onlyConfirmedBlock": false,
        "filterType" : "Transaction",
        "subscribeEvents": 
        [
            {
                "contractAddress":"2q7NLAr6eqF4CTsnNeXnBZ9k4XcmiUeM61CLWYaym6WsUmbg1k",
                "eventNames":
                [
                    "Played",
                    "Bingoed"
                ]
            }
        ]
    }
]
Parameters Explaination
| ChainId | The AElf chain ID to subscribe. |
| --- | --- |
| StartBlockNumber | The initial push height for subscription. |
| OnlyConfirmedBlock | Whether only confirmed blocks are subscribed or not. |
| FilterType | The type of block data to be subscribed. Currently, the indexer system categorizes a complete block data into three levels of data structures: Block, Transaction, and Logevent. For details, refer to the Scanning Data Structure Example. |
| SubscribeEvents | The subscribed events. |

After successfully calling the API, the version of subscription will be returned, e.g.
932e5a54b6044e049cf939607b248d89
Note down this version number, as it will be used in the development of the client interface plugin in Step 2.
Get Existing Subscription 
If you need to view all the initiated subscription information, you can query it through the following API.
Get request address：http://URL:{port}/api/app/subscription
Request Mode：none
Request Header
Authorization

Bearer {access_token}
Request Body：none
This will return the existing subscription, e.g.
{
    "currentVersion": {
        "version": "932e5a54b6044e049cf939607b248d89",
        "subscriptionInfos": [
            {
                "chainId": "tDVV",
                "startBlockNumber": 48532699,
                "onlyConfirmedBlock": false,
                "filterType": 1,
                "subscribeEvents": [
                    {
                        "contractAddress": "2q7NLAr6eqF4CTsnNeXnBZ9k4XcmiUeM61CLWYaym6WsUmbg1k",
                        "eventNames": [
                            "Played",
                            "Bingoed"
                        ]
                    }
                ]
            }
        ]
    },
    "newVersion": null
}
Stop Running Subscription
Post request address http://URL:{port}/api/app/block-scan/stop?version={subscription_version}
Request Mode：none
Request Header
Authorization
Bearer {access_token}
Request Body：none
Replay Running Subscription by New Subscription
Post request address: http://URL:8081/api/app/block-scan/upgrade
This API is used to replace current subscription version by new version. After a new subscription is created, it will be at "newVersion". When it's ready to use, this API is required to be called to upgrade it to currentVersion.
[Image]
Request Mode：none
Request Header
Authorization
Bearer {access_token}
Request Body：none
Update Running Subscription
Put request address：http://URL:{Port}/api/app/subscription/{Version}
Request Mode：raw
Request Header
Authorization
Bearer {access_token}
Request Body
[
    {
        "chainId": "AELF",
        "startBlockNumber": 54541,
        "onlyConfirmedBlock": false,
        "filterType" : "LogEvent",
        "subscribeEvents": 
        [
            {
                //update content
            }
        ]
    }
]
Step 2. Client Interface Plugin Development
Having understood the working principle of the AElf Indexer, you will find that to enable a DApp to request data from the AElf Indexer, the main task is to develop a client interface plugin.
[Image]
The following will use a sample as an example to explain in detail how to develop a client interface plugin.
A completed indexer project repo: https://github.com/Portkey-Wallet/bingo-game-indexer
Development Environment
.Net 7.0
Building the Project Skeleton
1. Build a .Net 7.0 empty project 
2. Import AElfIndexer.Client package. The latest version of this package is "1.0.0-28"
Code of the project file Sample.csproj:
<Project Sdk="Microsoft.NET.Sdk">
    <PropertyGroup>
        <TargetFramework>net7.0</TargetFramework>
        <ImplicitUsings>enable</ImplicitUsings>
    </PropertyGroup>
    <ItemGroup>
      <PackageReference Include="AElfIndexer.Client" Version="1.0.0-28" />
    </ItemGroup>
</Project>
3. Creating the GraphQL Query Interface Class
This interface will serve as the user's interface for querying data. It should include the logic based on which GraphQL returns data to the user when querying. This will be talked about in GraphQL interface development section.
public class Query
{
}
4. Create the GraphQL structure class SampleSchema, inheriting from AElfIndexerClientSchema<Query>.
public class SampleSchema : AElfIndexerClientSchema<Query>
{
    public SampleSchema(IServiceProvider serviceProvider) : base(serviceProvider)
    {
    }
}
For completed example, please refer: https://github.com/Portkey-Wallet/bingo-game-indexer/blob/master/src/BingoGame.Indexer.CA/GraphQL/BingoGameIndexerCASchema.cs
5. Create the module class SampleModule, dependent on AElfIndexerClientModule, and inherit from AElfIndexerClientPluginBaseModule<SampleModule, SampleSchema, Query>.
[DependsOn(typeof(AElfIndexerClientModule))]
public class SampleModule:AElfIndexerClientPluginBaseModule<SampleModule, SampleSchema, Query>
{
    protected override void ConfigureServices(IServiceCollection serviceCollection)
    {
        var configuration = serviceCollection.GetConfiguration();
        Configure<ContractInfoOptions>(configuration.GetSection("ContractInfo"));
    }

    protected override string ClientId => ""; //fill your own client here
    protected override string Version => ""; //fill the version returned by Send Subscription in Step 1

}
For completed example, please refer: https://github.com/Portkey-Wallet/bingo-game-indexer/blob/master/src/BingoGame.Indexer.CA/BingoGameIndexerCAModule.cs
Data Storage 
After the interface plugin receives the corresponding block information data from the AElf Indexer Client, it needs to process the block data for each height according to the custom code logic. The processed results should be updated and stored in the index library. In general, behind each interface, there is a corresponding index library that stores its result set.
Currently, the AElf Indexer system supports using ElasticSearch as the medium for persistent storage of index libraries. However, the entity class for the index library structure of the result set needs to be defined manually, inheriting from AElfIndexerClientEntity and implementing the IIndexBuild interface.
This entry refers to the data structure utilized when storing information into ElasticSearch after processing the data obtained through AElf Indexer.
public class SampleIndexEntry : AElfIndexerClientEntity<string>, IIndexBuild
{
    [Keyword]
    public string FromAddress { get; set; }
    
    public long Timestamp { get; set; }
    
    public long Balance { get; set; }
   
    //Define it according to your own usage requirements.
}
For completed example, please refer: https://github.com/Portkey-Wallet/bingo-game-indexer/blob/master/src/BingoGame.Indexer.CA/Entities/BingoGameIndexEntry.cs
Processor Development 
Depending on the subscribed block information type (Block/Transaction/LogEvent), the processing methods for each may vary slightly.
LogEvent
Processing block transaction data of LogEvent structure type primarily involves handling LogEventInfo. To do this, you need to inherit from the AElfLogEventProcessorBase class, override and implement its GetContractAddress and HandleEventAsync methods.
public class SampleLogEventProcessor : AElfLogEventProcessorBase<SampleEvent,LogEventInfo>
{
    private readonly IAElfIndexerClientEntityRepository<SampleIndex, LogEventInfo> _repository;
    private readonly ContractInfoOptions _contractInfoOptions;
    private readonly IObjectMapper _objectMapper;

    public NFTProtocolCreatedProcessor(ILogger<SampleLogEventProcessor> logger, IObjectMapper objectMapper,
        IAElfIndexerClientEntityRepository<SampleIndex, LogEventInfo> repository,
        IOptionsSnapshot<ContractInfoOptions> contractInfoOptions) : base(logger)
    {
        _objectMapper = objectMapper;
        _repository = repository;
        _contractInfoOptions = contractInfoOptions.Value;
    }

    public override string GetContractAddress(string chainId)
    {
        return _contractInfoOptions.ContractInfos.First(c=>c.ChainId == chainId).SampleContractAddress;
    }

    protected override async Task HandleEventAsync(SampleEvent eventValue, LogEventContext context)
    {
        //implement your handling logic here
    }
}
For completed example, please refer: https://github.com/Portkey-Wallet/bingo-game-indexer/blob/master/src/BingoGame.Indexer.CA/Processors/PlayedProcessor.cs
Block
Processing block structure type block data mainly involves handling BlockInfo. To do this, you need to inherit from the BlockDataHandler class and override and implement its ProcessDataAsync method.
public class SampleBlockProcessor : BlockDataHandler
{
    private readonly IAElfIndexerClientEntityRepository<SampleIndex, BlockInfo> _repository;

    public SampleBlockProcessor(IClusterClient clusterClient, IObjectMapper objectMapper,
        IAElfIndexerClientInfoProvider aelfIndexerClientInfoProvider,
        IAElfIndexerClientEntityRepository<SampleIndex, BlockInfo> repository,
        ILogger<SampleBlockProcessor> logger) : base(clusterClient, objectMapper, aelfIndexerClientInfoProvider,logger)
    {
        _repository = repository;
    }

    protected override async Task ProcessDataAsync(List<BlockInfo> data)
    {
        foreach (var block in data)
        {
            var index = ObjectMapper.Map<BlockInfo, SampleIndex>(block);
            Logger.LogDebug(index.ToJsonString());
            await _repository.AddOrUpdateAsync(index);
        }
        
    }

    protected override Task ProcessBlocksAsync(List<BlockInfo> data)
    {
        //implement your handling logic here
    }
}
Transaction
Processing transaction structure type block transaction data mainly involves handling TransactionInfo. To do this, you need to inherit from the AElfLogEventProcessorBase class, and override and implement its GetContractAddress and HandleEventAsync methods.
public abstract class SampleTransactionProcessor :AElfLogEventProcessorBase<SampleEvent,TransactionInfo>
{
    protected readonly IAElfIndexerClientEntityRepository<SampleTransactionIndex, TransactionInfo> SampleTransactionIndexRepository;
    protected readonly IAElfIndexerClientEntityRepository<SampleIndex, LogEventInfo> SampleIndexRepository;
    protected readonly ContractInfoOptions ContractInfoOptions;
    protected readonly IObjectMapper ObjectMapper;

    protected SampleTransactionProcessor(ILogger<SampleTransactionProcessor> logger,
        IAElfIndexerClientEntityRepository<SampleIndex, LogEventInfo> sampleIndexRepository,
        IAElfIndexerClientEntityRepository<SampleTransactionIndex, TransactionInfo> sampleTransactionIndexRepository,
        IOptionsSnapshot<ContractInfoOptions> contractInfoOptions,
        IObjectMapper objectMapper) : base(logger)
    {
        SampleTransactionIndexRepository = sampleTransactionIndexRepository;
        SampleIndexRepository = sampleIndexRepository;
        ContractInfoOptions = contractInfoOptions.Value;
        ObjectMapper = objectMapper;
    }

    public override string GetContractAddress(string chainId)
    {
        return ContractInfoOptions.ContractInfos.First(c=>c.ChainId == chainId).SampleContractAddress;
    }

    protected override async Task HandleEventAsync(SampleEvent eventValue, LogEventContext context)
    {
        //implement your handling logic here
    }
}
In addition to that, handling block transaction data of the Transaction structure type requires inheriting from the TransactionDataHandler class and implementing its ProcessTransactionsAsync method. However, this method only needs to be implemented once and doesn't need to be implemented for each event.
public class SampleTransactionHandler : TransactionDataHandler
{
    public SampleTransactionHandler(IClusterClient clusterClient, IObjectMapper objectMapper,
        IAElfIndexerClientInfoProvider aelfIndexerClientInfoProvider, IDAppDataProvider dAppDataProvider,
        IBlockStateSetProvider<TransactionInfo> blockStateSetProvider,
        IDAppDataIndexManagerProvider dAppDataIndexManagerProvider,IEnumerable<IAElfLogEventProcessor<TransactionInfo>> processors,
        ILogger<CAHolderTransactionHandler> logger) : base(clusterClient, objectMapper, aelfIndexerClientInfoProvider,
        dAppDataProvider,blockStateSetProvider,dAppDataIndexManagerProvider,processors, logger)
    {
    }

    protected override Task ProcessTransactionsAsync(List<TransactionInfo> transactions)
    {
        return Task.CompletedTask;
    }
}
Processor Registration 
Once processor development is finished, registration is also required for processors.
[DependsOn(typeof(AElfIndexerClientModule))]
public class SampleModule:AElfIndexerClientPluginBaseModule<SampleModule, SampleSchema, Query>
{
    protected override void ConfigureServices(IServiceCollection serviceCollection)
    {
        var configuration = serviceCollection.GetConfiguration();
        Configure<ContractInfoOptions>(configuration.GetSection("ContractInfo"));
        
        serviceCollection.AddSingleton<IBlockChainDataHandler, SampleBlockProcessor>();
        serviceCollection.AddTransient<IBlockChainDataHandler, SampleTransactionHandler>();
        serviceCollection.AddSingleton<IAElfLogEventProcessor<LogEventInfo>, SampleLogEventProcessor>();
        serviceCollection.AddSingleton<IAElfLogEventProcessor<TransactionInfo>, SampleTransactionProcessor>();
        //register developed Processor    
    }

    protected override string ClientId => "";
    protected override string Version => "";

}
For completed example, please refer: https://github.com/Portkey-Wallet/bingo-game-indexer/blob/master/src/BingoGame.Indexer.CA/BingoGameIndexerCAModule.cs
GraphQL Interface Development
After development of processors, the processed data should be able to stored in ElasticSearch.The next step is to develop GraphQL interface, so that users can get the data. The GraphQL interface should construct query conditions, retrieve data from the AElf indexer, and organize the results.
public class Query
{
    //implement your logic
}
For completed example, please refer: https://github.com/Portkey-Wallet/bingo-game-indexer/blob/master/src/BingoGame.Indexer.CA/GraphQL/Query.cs
Deployment
Compile the developed indexer project, and obtain the compiled DLL file. Hand over the compiled Sample.dll file to the administrator of the AElf Indexer system. The administrator will place the Sample.dll file into the plugIns folder within the DApp module of the AElf Indexer system. 
ubuntu@protkey-did-test-indexer-a-01:/opt/aelf-indexer/dapp-bingo/plugins$ ls
BingoGame.Indexer.CA.dll
Subsequently, the AElf Indexer system will automatically initiate the process of pushing blocks to the interface plugin for processing, adhering to the pre-subscribed requirements, and simultaneously expose the corresponding GraphQL interfaces to external entities. The GraphQL interface address will be http://URL:{port}/AElfIndexer_DApp/SampleSchema/graphql
This playground can check whether the indexer works properly, e.g. The playground for bingogame indexer:
[Image]
Conclusion
By following these steps, DApps can seamlessly integrate with the AElf Indexer, enabling efficient retrieval and processing of on-chain data. This comprehensive guide gives introduction and ensures a smooth development process.
