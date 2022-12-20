1. Create "web" dir and "index.html" file inside
1. Create Azure Map service
1. Insert content:
```
<!DOCTYPE html>
 <html>
 <head>
     <title>Live Flight Data Map</title>
     <meta charset="utf-8">
     <meta name="viewport" content="width=device-width, initial-scale=1, shrink-to-fit=no">

     <!-- Add references to the Azure Maps Map control JavaScript and CSS files. -->
     <link rel="stylesheet" href="https://atlas.microsoft.com/sdk/javascript/mapcontrol/2/atlas.min.css" type="text/css">
     <script src="https://atlas.microsoft.com/sdk/javascript/mapcontrol/2/atlas.min.js"></script>

     <!-- Add a reference to the Azure Maps Services Module JavaScript file. -->
     <script src="https://atlas.microsoft.com/sdk/javascript/mapcontrol/2/atlas-service.min.js"></script>

     <script>
     function GetMap(){
         //Add Map Control JavaScript code here.
     }
     </script>

     <style>
         html,
         body {
             width: 100%;
             height: 100%;
             padding: 0;
             margin: 0;
         }

         #myMap {
             width: 100%;
             height: 100%;
         }
     </style>
 </head>
 <body onload="GetMap()">
     <div id="myMap"></div>
 </body>
 </html>
 ```
 4. Inside GetMap() function put:
 ```
 //Instantiate a map object
var map = new atlas.Map("myMap", {
    //Add your Azure Maps subscription key to the map SDK. Get an Azure Maps key at https://azure.com/maps
    authOptions: {
        authType: 'subscriptionKey',
        subscriptionKey: '<Your Azure Maps Key>'
    }
});
```
5. Insert your subscription key
6. Add basic params:
```
style: "night",
center: [19.550399, 52.207330],
zoom: 5
```
7. Open the file in a browser and check if you can see a map.
8. Add Promise
```
<!-- Promis based http client. https://github.com/axios/axios -->
<script src="https://unpkg.com/axios/dist/axios.min.js"></script>
```
9. Add function:
```
function GetFlightData() {
    return axios.get('https://opensky-network.org/api/states/all?lamin=48.164342&lomin=10.783784&lamax=57.075963&lomax=32.914270')
    .then(function (response) {
        return response;
    }).catch(console.error)
    }
```
10. Inside GetMap() function put:
```
//Wait until the map resources are ready.
        map.events.add('ready', function () {
            map.imageSprite.add('plane-icon', 'https://piotrstorage123.blob.core.windows.net/imgs/airplane.png');

            //Create a data source and add it to the map
            var datasource = new atlas.source.DataSource();
            map.sources.add(datasource);

            //Create a symbol layer using the data source and add it to the map
            map.layers.add(
                new atlas.layer.SymbolLayer(datasource, null, {
                    iconOptions: {
                        ignorePlacement: true,
                        allowOverlap: true,
                        image: 'plane-icon',
                        size: 0.08,
                        rotation: ['get', 'rotation']
                    },
                    textOptions: {
                        textField: ['concat', ['to-string', ['get', 'name']], '- ', ['get', 'altitude']],
                        color: '#FFFFFF',
                        offset: [2, 0]
                    }
                }));

            GetFlightData().then(function (response) {
                for (var flight of response.data.states) {
                    var pin = new atlas.Shape(new atlas.data.Point([flight[5], flight[6]]));
                    pin.addProperty('name', flight[1]);
                    pin.addProperty('altitude', flight[7]);
                    pin.addProperty('rotation', flight[10]);
                    datasource.add(pin);
                }
            });
        });
```
11. Now check if you can see current planes draw on a map.
12. Create CosmosDB account and collection
13. Create SignalR service with Serverless service mode
14. Create Azure Funtion App.
15. Create new Azure Fucntion in VS Code.
16. Add CosmosDB package:
```
dotnet add package Microsoft.Azure.WebJobs.Extensions.CosmosDB --version 4.0.0
```
17. Write the function code (fill in databaseName and containerName):
```
public async Task Run(
            [TimerTrigger("0 */1 * * * *")]TimerInfo myTimer,
            [CosmosDB(
            databaseName: "DB_NAME",
            containerName: "CONTAINER_NAME",
            Connection = "AzureCosmosDBConnection")]IAsyncCollector<Flight> documents,
         ILogger log)
        {
            log.LogInformation($"C# Timer trigger function executed at: {DateTime.Now}");

            var openSkyUrl = "https://opensky-network.org/api/states/all?lamin=48.164342&lomin=10.783784&lamax=57.075963&lomax=32.914270";

            using (HttpResponseMessage res = await client.GetAsync(openSkyUrl))
            using (HttpContent content = res.Content)
            {
                var result = JsonConvert.DeserializeObject<Rootobject>(await content.ReadAsStringAsync());
                foreach (var item in result.states) {
                    await documents.AddAsync(Flight.CreateFromData(item));
                }

                log.LogInformation($"Total flights processed{result.states.Length}");
            }
        }
```
18. Get CosmosDB connection string and put a variable "AzureCosmosDBConnection" with Connection String in these places:
- in local.setting.json file
- in Configuration of Azure Function App
19. Test function locally
20. Deploy it to Azure Funtion App
21. In the same project create new Azure Function with CosmosDB trigger.
22. Paste function code  (fill in databaseName and containerName):
```
public static async Task Run([CosmosDBTrigger(
            databaseName: "DB_NAME",
            containerName: "CAONTAINER_NAME",
            Connection = "AzureCosmosDBConnection",
            LeaseContainerName = "leases",
            CreateLeaseContainerIfNotExists = true)]IReadOnlyList<Flight> input,
            ILogger log)
        {
            if (input != null && input.Count > 0)
            {
                log.LogInformation("Documents modified " + input.Count);
                log.LogInformation("First document Id " + input[0].icao24);
            }
        }
```
23. Test function locally
24. Install SignalR nuget package:
```
dotnet add package Microsoft.Azure.WebJobs.Extensions.SignalRService --version 1.8.0
```
25. Get SignalR connection string and put a variable "AzureSignalRConnectionString" with Connection String in these places:
- in local.setting.json file
- in Configuration of Azure Function App
26. Modify the function as follows:
```
if (input != null && input.Count > 0)
{
    log.LogInformation("Documents modified " + input.Count);
    foreach (var flight in input)
    {
        await signalRMessages.AddAsync(new SignalRMessage
        {
            Target = "newFlightData",
            Arguments = new[] { flight }
        });
    }
}
```
27. Open Live Tracing in SignalR and check that your messages are there.
28. Create new Azure Function with HTTP trigger, give it a name: "negotiate"
29. Write function code:
```
public static class negotiate
{
    [FunctionName("negotiate")]
    public static async Task<IActionResult> Run(
        [HttpTrigger(AuthorizationLevel.Function, "get", "post", Route = null)] HttpRequest req,
        [SignalRConnectionInfo(HubName = "flightdata")] SignalRConnectionInfo connectionInfo,
        ILogger log)
    {
        return new OkObjectResult(connectionInfo);
    }
}
```
30. Add CORS rule to local.settingg.json:
```
"Host": {
    "LocalHttpPort": 7071,
    "CORS": "*"
}
```
31. Open index.html and add SignalR lib:
```
<script src="https://unpkg.com/@aspnet/signalr@1.0.2/dist/browser/signalr.js"></script>
```
32. Add new function that will call negotaiate endpoint:
```
function GetConnectionInfo() {
    return axios.get('http://localhost:7071/api/negotiate')
        .then(function (response) {
            return response.data
        }).catch(console.error)
}
```
33. Add function that initiates connection to signalR
```
function StartConnection(connection) {
    console.log('connecting...')
    connection.start()
        .then(function () { console.log('connected!') })
        .catch(function (err) {
            console.error(err)
            setTimeout(function () { StartConnection(connection) }, 2000)})
}
```
34. Refactor code. Add this variables and function: 
```
let datasource;
let planes = [];

function ProcessFlightData(flight) {
    console.log(flight);

    var newFlightPin = new atlas.Shape(new atlas.data.Point([flight.longitute, flight.latitude]), flight.id);
    newFlightPin.addProperty('name', flight.callsign);
    newFlightPin.addProperty('altitude', flight.altitude);
    newFlightPin.addProperty('rotation', flight.trueTrack);

    planes[flight.id] = newFlightPin;
    datasource.setShapes(Object.values(planes));
}
```
35. Remove GetFlightData() function and call this function from GetMap().
36. Remove "var" keyword from initialization of "datasource" variable. We want to make it local.
37. Initiate the connection, put this code in map "ready" callback:
```
GetConnectionInfo().then(function (info) {
    let accessToken = info.accessToken
    const options = {
        accessTokenFactory: function () {
            if (accessToken) {
                const _accessToken = accessToken
                accessToken = null
                return _accessToken
            } else {
                return GetConnectionInfo().then(function (info) {
                    return info.accessToken
                })
            }
        }
    }

    const connection = new signalR.HubConnectionBuilder()
        .withUrl(info.url, options)
        .build()

    StartConnection(connection)

    connection.on('newFlightData', ProcessFlightData)

    connection.onclose(function () {
        console.log('disconnected')
        setTimeout(function () { StartConnection(connection) }, 5000)
    })           
}).catch(console.error)
```
38. Test the app!
