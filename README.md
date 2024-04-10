## Using dotnet-monitor on Azure Container Apps

[Dotnet-monitor](https://github.com/dotnet/dotnet-monitor) is a diagnostics tool that can be deployed as a sidecar container to collect diagnostics information about a main application. It has [two mechanisms](https://github.com/dotnet/dotnet-monitor?tab=readme-ov-file#overview) for diagnostics collection, an HTTP API for on demand collection and Triggers for rule-based configuration for always-on collection of artifacts.

The HTTP API mechanism is suggested for running on Kubernetes in the documentation. However, this currently doesn't work for Azure Container Apps neither it's recommended to expose another port in production scenarios so Triggers are the way to go.

You may want to take a look at this [repository](https://github.com/Azure-Samples/dotNET-FrontEnd-to-BackEnd-with-dotnet-monitor-on-Azure-Container-Apps) of a .NET app deployed with dotnet-monitor on Azure Container Apps. In particular, at this [bicep file](https://github.com/Azure-Samples/dotNET-FrontEnd-to-BackEnd-with-dotnet-monitor-on-Azure-Container-Apps/blob/main/Azure/container_app.bicep). There's not a detailed explanation on how to set up dotnet-monitor but this file is a good starting point and it'll be a reference for this guide.

The following are the steps to use dotnet-monitor 6 on Azure Container Apps with Azure Blob Storage after deploying an application.

### Set the storage mount

Create a storage mount (volume mount) for the container application, an ephemeral storage should be enough (see [documentation](https://learn.microsoft.com/en-us/azure/container-apps/storage-mounts?pivots=azure-portal)) and [configure it](https://learn.microsoft.com/en-us/azure/container-apps/storage-mounts?pivots=azure-portal#configuration) to the main application’s container such that the mount path is set to “/diag” . The storage or volume mount is needed so enable communication between both containers.

### Set environment variables in the main container

Set the environment variable `DOTNET_DiagnosticPorts` to `/diagport/port.sock` for dotnet-monitor 6. For other versions of dotnet-monitor, see the following [samples](https://github.com/dotnet/dotnet-monitor/blob/main/documentation/kubernetes.md#example-deployment).

### Deploy the dotnet-monitor container as a sidecar container

In Azure Portal, go to the Azure Container App and on the left panel, there’s an option for “Revisions and replicas”. Select “Create a new revision”, we can then go to “add a sidecar container” (not init container) and fill the fields as shown.

| Field | Value |
|----------|----------|
|   Name   |   dotnet-monitor   |
|   Image source   |  Docker Hub or other registries    |
|   Image type   |   Public   |
|   Registry login server   |  mcr.microsoft.com    |
|   Image and tag   |   dotnet/monitor:6   |
|   Command override   |   dotnet-monitor   |

Set CPU cores and Memory (Gi) as convenient (there are some Azure restrictions to these values so if the inserted values are invalid, a text will pop-up with the restrictions).

Also, switch to the volume mounts section, and [configure](https://learn.microsoft.com/en-us/azure/container-apps/storage-mounts?pivots=azure-portal#configuration) this container as well  with the same storage mount and same mount path as the main container.

The following step is to add at the bottom the environment variables necessary to configure dotnet-monitor, the azure blob storage and the triggers. As this step is rather long, it will be covered in its own section below.

### Environment variables in dotnet-monitor

There are some essential environment variables that dotnet-monitor need to work (as shown [here](https://github.com/Azure-Samples/dotNET-FrontEnd-to-BackEnd-with-dotnet-monitor-on-Azure-Container-Apps/blob/main/Azure/container_app.bicep#L100-L113)). The variables just depend on the version of dotnet-monitor used (see [documentation](https://github.com/dotnet/dotnet-monitor/blob/main/documentation/kubernetes.md#example-deployment)).

In particular, this example uses [Azure Blob Storage](https://learn.microsoft.com/en-us/azure/storage/blobs/storage-blobs-overview) to store the artifacts collected. See this [quickstart](https://learn.microsoft.com/en-us/azure/storage/blobs/storage-quickstart-blobs-portal) on how to create and use one. Once created, the account key (which is necessary to configure egress), can be found by going to the Storage Account / Security + networking / Access keys.

<img src=".\AccessKey.png" alt="request"/>

There are also some environment variables that configure the [egress](https://github.com/dotnet/dotnet-monitor/blob/main/documentation/egress.md) for dotnet-monitor (as shown [here](https://github.com/Azure-Samples/dotNET-FrontEnd-to-BackEnd-with-dotnet-monitor-on-Azure-Container-Apps/blob/main/Azure/container_app.bicep#L115-L130)). In this particular case, the app outputs the dumps to an Azure Blob Storage, and the configuration can be structured as a [JSON](https://github.com/dotnet/dotnet-monitor/blob/main/documentation/configuration/egress-configuration.md#example-azureblobstorage-provider).



```JSON
{
    "Egress": {
        "AzureBlobStorage": {
            "monitorBlob": {
                "accountUri": "https://exampleaccount.blob.core.windows.net",
                "containerName": "dotnet-monitor",
                "blobPrefix": "artifacts",
                "accountKeyName": "MonitorBlobAccountKey"
            }
        },
        "Properties": {
            "MonitorBlobAccountKey": "accountKey"
        }
    }
}
```

However, a JSON can’t be injected to dotnet-monitor so essentially this file must be converted to Kubernetes Environment Variables format. These are the environment variables that need to be enabled for dotnet-monitor to configure the egress.

```yaml
- name: DotnetMonitor_Egress__AzureBlobStorage__monitorBlob__accountUri
  value: "https://exampleaccount.blob.core.windows.net"
- name: DotnetMonitor_Egress__AzureBlobStorage__monitorBlob__containerName
  value: "dotnet-monitor"
- name: DotnetMonitor_Egress__AzureBlobStorage__monitorBlob__blobPrefix
  value: "artifacts"
- name: DotnetMonitor_Egress__AzureBlobStorage__monitorBlob__accountKeyName
  value: "MonitorBlobAccountKey"
- name: DotnetMonitor_Egress__Properties__MonitorBlobAccountKey
  value: "accountKey"
```

Finally, the [collection rules](https://github.com/dotnet/dotnet-monitor/blob/main/documentation/collectionrules/collectionrules.md#collection-rules) are the core of dotnet-monitor as they will define the automatic collection of diagnostic artifacts.

They consist of four key aspects:

[Filters](https://github.com/dotnet/dotnet-monitor/blob/main/documentation/collectionrules/collectionrules.md#filters): Describes for which processes the rule is applied. Can filter on aspects such as process name, ID, and command line. If not configured, the rule is applied to all the processes.

[Trigger](https://github.com/dotnet/dotnet-monitor/blob/main/documentation/collectionrules/collectionrules.md#triggers): A condition to monitor in the target process.

[Actions](https://github.com/dotnet/dotnet-monitor/blob/main/documentation/collectionrules/collectionrules.md#actions): A list of actions to execute when the trigger condition is satisfied.

[Limits](https://github.com/dotnet/dotnet-monitor/blob/main/documentation/configuration/collection-rule-configuration.md#limits): Limits applied to the rule or action execution.

Specific information on what filters, triggers, actions and limits can be set can be found in the links above. Once they’re determined, the JSON will look something like this (see more [samples](https://github.com/dotnet/dotnet-monitor/blob/main/documentation/collectionrules/collectionruleexamples.md)).

```JSON
{
    "CollectionRules": {
        "HighRequestsRule": {
            "Trigger": {
                "Type": "AspNetRequestCount",
                "Settings": {
                    "RequestCount": 100,
                    "SlidingWindowDuration": "00:01:00",
                }
            },
            "Actions": [{
                "Type": "CollectTrace",
                "Settings": {
                    "Profile": "Cpu",
                    "Duration": "00:01:00",
                    "Egress": "monitorBlob"
                }
            }],
            "Limits": {
                "ActionCount": 1,
                "ActionCountSlidingWindowDuration": "00:01:00"
            }
        }
    }
}
```

However, this information should be injected as environment variables too. Be careful of the Actions key as its value it's an array of possible actions so each element is indexed with `__<index>__`.

```yaml
- name: CollectionRules__HighCpuRule__Trigger__Type
  value: AspNetRequestCount

- name: CollectionRules__HighCpuRule__Trigger__Settings__RequestCount
  value: 100

- name: CollectionRules__HighCpuRule__Trigger__Settings__SlidingWindowDuration
  value: 00:01:00

- name: CollectionRules__HighCpuRule__Actions__0__Type
  value: CollectTrace

- name: CollectionRules__HighCpuRule__Actions__0__Settings__Profile
  value: Cpu

- name: CollectionRules__HighCpuRule__Actions__0__Settings__Duration
  value: 00:01:00

- name: CollectionRules__HighCpuRule__Actions__0__Settings__Egress
  value: monitorBlob

- name: CollectionRules__HighCpuRule__Limits__ActionCount
  value: 1

- name: CollectionRules__HighCpuRule__Limits__ActionCountSlidingWindowDuration
  value: 00:01:00
```

A sample of a full set of environment variables that configure dotnet-monitor, the egress and the collection rules are shown below.

```yaml
- name: DotnetMonitor_DiagnosticPort__ConnectionMode
  value: Listen

- name: DotnetMonitor_Storage__DumpTempFolder
  value: /diag

- name: DotnetMonitor_Urls
  value: http://*:52323

- name: DotnetMonitor_DiagnosticPort__EndpointName
  value: /diag/dotnet-monitor.sock

# The URI of the Azure blob storage account.
- name: Egress__AzureBlobStorage__monitorBlob__AccountUri
  value: https://storagetraces.blob.core.windows.net

# The name of the container to which the blob will be egressed. If egressing to the root container, use the "$root" sentinel value.
- name: Egress__AzureBlobStorage__monitorBlob__ContainerName
  value: dotnet-monitor

# The account key used to access the Azure blob storage account; must be specified if accountKeyName is not specified.
- name: Egress__AzureBlobStorage__monitorBlob__AccountKey
  value: accountKey

- name: CollectionRules__HighCpuRule__Trigger__Type
  value: AspNetRequestCount

- name: CollectionRules__HighCpuRule__Trigger__Settings__RequestCount
  value: 100

- name: CollectionRules__HighCpuRule__Trigger__Settings__SlidingWindowDuration
  value: 00:01:00

- name: CollectionRules__HighCpuRule__Actions__0__Type
  value: CollectTrace

- name: CollectionRules__HighCpuRule__Actions__0__Settings__Profile
  value: Cpu

- name: CollectionRules__HighCpuRule__Actions__0__Settings__Duration
  value: 00:01:00

- name: CollectionRules__HighCpuRule__Actions__0__Settings__Egress
  value: monitorBlob

- name: CollectionRules__HighCpuRule__Limits__ActionCount
  value: 1

- name: CollectionRules__HighCpuRule__Limits__ActionCountSlidingWindowDuration
  value: 00:01:00
```

### Additional setting

Finally, we need to set the args for the container. The `--no-auth` flag will be used as well for simplicity but [authentication](https://github.com/dotnet/dotnet-monitor/blob/main/documentation/authentication.md) should be configured in production scenarios. To add the args config, we need to run the following commands:

```powershell

$ az login

$ az containerapp show -n <APP_NAME> -g <RESOURCE_GROUP_NAME> -o yaml > app.yaml
```

This command allows to get the internal yaml file used by ACA. It should be edited to add the args as shown here (environment variables omitted) and then upload the yaml file.

```yaml
...
template:
    containers:
    - env:
      - name: ASPNETCORE_URLS
        value: http://+:8080
      - name: DOTNET_DiagnosticPorts
        value: /diag/dotnet-monitor.sock
      image: eduardovpcr.azurecr.io/aspnet-test:latest
      name: aspnet-test
      probes: []
      resources:
        cpu: 0.25
        ephemeralStorage: 1Gi
        memory: 0.5Gi
      volumeMounts:
      - mountPath: /diag
        volumeName: diagvol
    - args: # add this key
      - collect 
      - --no-auth
      command:
      - dotnet-monitor
      image: mcr.microsoft.com/dotnet/monitor:6
      name: dotnet-monitor
      probes: []
      resources:
        cpu: 0.25
        ephemeralStorage: 1Gi
        memory: 0.5Gi
      volumeMounts:
      - mountPath: /diag
        volumeName: diagvol
```

Finally, this last command allows to update the internal yaml file with the edited one.

```powershell
$ az containerapp update --name <APP_NAME> --resource-group <RESOURCE_GROUP_NAME> --yaml app.yaml
```

After completing all these steps, dotnet-monitor will execute the defined actions according to the specified triggers.