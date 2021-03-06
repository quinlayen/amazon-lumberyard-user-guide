# Creating a Custom Asset Builder<a name="asset-builder-custom"></a>

You can use the Asset Builder SDK to develop an asset builder that processes your custom asset type source files into game\-ready files\. This topic shows you how to create your own asset builder and uses examples from the example asset builder that is included with Lumberyard\.

At a high level, to create a builder for a custom asset type, you perform the following steps:

[1\. Create Builder Classes](#asset-builder-custom-create-builder-class) – Create one or more builder classes that build the asset\. The classes must implement the appropriate callbacks and handle shutdown messages from Asset Processor\.

[2\. Create a Lifecycle Component](#asset-builder-custom-create-a-lifecycle-component) – Create a lifecycle `AZ::Component` that registers all of your builder classes\. The lifecycle component provides Asset Processor with information to ensure that the correct asset builder is invoked for a file\.

[3\. Tag Components for Builder Mode](#asset-builder-custom-tag-components-for-builder-mode) – Tag your lifecycle component and any supporting `AZ::Component` instances with the `AssetBuilder` tag to ensure that they are activated in builder mode\.

[4\. \(Optional\) Implement Message Logging](#asset-builder-custom-optional-implement-message-logging) – Use the `BuilderLog()` function and standard `AZ_Trace` macros to log builder\-related messages or errors\.

This topic describes how to create builder classes, register your builder, tag your components, and implement message logging for your builder\.

## Builder Resources<a name="asset-builder-custom-builder-resources"></a>

This topic draws on the following resources, which are included with Lumberyard:
+ Lumberyard Asset Builder SDK – The Asset Builder SDK enables you to build custom asset processing tools\. The source code is located in the following directory:

  `lumberyard_version\dev\Code\Tools\AssetProcessor\AssetBuilderSDK\AssetBuilderSDK`
+ CustomAssetExample gem – Provides sample custom asset builder code\. The builder\-related source files for the gem are located in the following directory:

  `lumberyard_version\dev\Gems\CustomAssetExample\Code\Source\CustomAssetExample\Builder`

## Prerequisites<a name="asset-builder-custom-prerequisites"></a>

This topic assumes that you have a working knowledge of Lumberyard [Gems](gems-system-structure.md), [AZ::Modules](az-module-parts.md), and [AZ::Components](component-entity-system-create-component.md)\. The next section includes a brief overview of asset builders inside gems\.

## Asset Builders Inside Gems<a name="asset-builder-custom-asset-builders-inside-gems"></a>

Gems contain two kinds of modules:
+ A runtime module: `gem_name.dll`
+ A tools module: `gem_name.Editor.dll`

These modules contain [system components](az-module-system-components.md) and tool components\. When Lumberyard starts, an [AZ::ComponentApplication](https://docs.aws.amazon.com/lumberyard/latest/apireference/class_a_z_1_1_component_application.html) activates all required system components for the gems that are enabled for the project\.

## 1\. Create Builder Classes<a name="asset-builder-custom-create-builder-class"></a>

To create an asset builder, you must implement one or more builder classes\. You can create a builder class with the following steps:

[A\. Implement the CreateJobsFunction Callback Function](#asset-builder-custom-create-builder-class-createjobs-callback)

[B\. \(Optional\) Declare Source File Dependencies](#asset-builder-custom-create-builder-class-optional-declare-source-file-dependencies)

[C\. \(Optional\) Handle Platform\-Specific Cases](#asset-builder-custom-create-builder-class-platform-specific)

[D\. Implement the Callback for ProcessJobFunction](#asset-builder-custom-create-builder-class-processjob-callback)

[E\. \(Optional\) Create a JobCancelListener](#asset-builder-custom-create-builder-class-optional-create-jobcancellistener)

[F\. Shut Down Properly](#asset-builder-custom-create-builder-class-shut-down-properly)

Each builder class requires two callback functions: one for `CreateJobFunction`, and one for `ProcessJobFunction`\. Asset Processor uses these callbacks to communicate with your builder class\.

Example builder class code is located in the `lumberyard_version\dev\Gems\CustomAssetExample\Code\Source\Builder\CustomAssetExampleBuilderWorker.cpp` file\.

### A\. Implement the CreateJobsFunction Callback Function<a name="asset-builder-custom-create-builder-class-createjobs-callback"></a>

In most cases, you should build a `JobDescriptor` for each processing job for each enabled platform\. Then, add the `JobDescriptor` to the `CreateJobsResponse` list in the callback for `CreateJobsFunction`\.

Keep in mind the following:
+ To ensure that critical files are included, Lumberyard Editor blocks on startup until all callbacks for `CreateJobFunction` have completed\. Due to this startup constraint, we recommend that your code perform minimal work during the `CreateJobsFunction` callback\. For heavy processing work, use the callback for the `ProcessJobFunction`\.
+ For extra configurability, you can place arbitrary key–value pairs into the `JobDescriptor.m_jobParameters` field\. The key–value pairs are retained and sent to the callback for `ProcessJobsFunction`\. You can store information gathered during the `CreateJobsFunction` callback in these key–value pairs and then pass the information as parameters to the callback for `ProcessJobsFunction`\.
+ To delete stale products, Asset Processor compares the `JobDescriptor` that you create with the `JobDescriptors` that were created in the last iteration\. Asset Processor compares `JobDescriptors` that have the same input source files, `PlatformInfo` value, and job key\.
+ You do not have to check whether a `JobDescriptor` that you create needs to be processed later\. Instead, create all possible jobs at every iteration for a particular input asset on each enabled platform\.

For an example of a `CreateJobsFunction` callback, see the `CustomAssetExample::ExampleBuilderWorker::CreateJobs()` function in the CustomAssetExample gem\.

### B\. \(Optional\) Declare Source File Dependencies<a name="asset-builder-custom-create-builder-class-optional-declare-source-file-dependencies"></a>

You can use the Asset Builder SDK to declare dependencies for a source file on other source files\. These other files can be any file within the project directory or directories\. They do not have to be source files consumed by a builder\.

#### Declaring Dependencies<a name="asset-builder-custom-create-builder-class-declaring-dependencies"></a>

To declare dependencies, add them in your builder class's `CreateJobsFunction` callback to `m_sourceFileDependencyList` in the `CreateJobsResponse` structure\.

Keep in mind the following:
+ Declaring dependencies for a source file implies that the data in its output changes if the files that the source file depends on change\. If any of the source dependency files are modified, Asset Processor calls the `CreateJobsFunction` and `ProcessJobsFunction` callbacks on the dependent files to reprocess them\.
+ Asset Processor automatically handles source file dependencies recursively\. If the source files downstream emit their own dependencies when they are queried, you do not have to recursively traverse the full tree of dependencies\. Just emit your local dependencies for each node in the tree, and Asset Processor takes care of the rest\.
+ Because metafiles like `*.assetinfo` files are special case files that cause your asset to rebuild automatically, you do not have add them as dependencies\.

#### Source File UUIDs versus Paths<a name="asset-builder-custom-create-builder-class-source-file-uuids-versus-paths"></a>

The `SourceFileDependency` structure contains `m_sourceFileDependencyPath` and `m_sourceFileDependencyUUID`\. Your builder class must supply a value for only one of these fields\.

Keep in mind the following:
+ If the UUID of the file to add as a dependency is known inside the `CreateJobs` method, the builder can simply fill in the `m_sourceFileDependencyUUID` field\. Otherwise, the builder must provide the appropriate source dependency file path for the `m_sourceFileDependencyPathfield`\.
+ The field `m_sourceFileDependencyPath` can take both absolute and relative file paths\. If a relative path is specified, the appropriate overriding asset is used, if present\.
+ If the builder uses a relative file path for the `m_sourceFileDependencyPath` field, then the specified path has to be relative to one of the watched directories\. However, if both the source and the source dependency file exist in the same directory, you can provide just the file name without a path\.

For an example of adding dependencies inside of a `CreateJobsFunction` callback, see the `CustomAssetExampleBuilderWorker::CreateJobs()` function in the CustomAssetExample gem\.

### C\. \(Optional\) Handle Platform\-Specific Cases<a name="asset-builder-custom-create-builder-class-platform-specific"></a>

`CreateJobsRequest` provides helper functions for operations related to the enabled platforms\. These helper functions can be useful for building the output `JobDescriptor` for a specific enabled platform\.

For more information about declaring, enabling, or disabling platforms, see [Configuring the Asset Pipeline](asset-pipeline-configuring.md)\.

The following functions are available in the Asset Builder SDK\. For source code, see `lumberyard_version\dev\Code\Tools\AssetProcessor\AssetBuilderSDK\AssetBuilderSDK\AssetBuilderSDK.*`
+ `HasPlatform(const char* platformIdentifier)` – For the specified platform identifier, returns whether that platform is enabled for this `CreateJobsRequest`\. The platform identifier is data driven and user specified\. It is usually a string representation of the platform name \(for example, "pc" or "osx"\)\.
+ `HasPlatformWithTag(const char* platformTag)` – For the specified platform tag, returns whether Lumberyard has any enabled platforms in the `CreateJobRequest` that contain that tag\. Tags are data driven and user specified\. They usually identify features that are not specific to a single platform \(for example, "mobile" or "console"\)\.

### D\. Implement the Callback for ProcessJobFunction<a name="asset-builder-custom-create-builder-class-processjob-callback"></a>

Asset Processor calls the callback for the `ProcessJobFunction` when it has a job for the builder class to begin processing\. The callback for your `ProcessJobFunction` should perform the following tasks:

1. Process the source file and perform all work inside the temporary directory\.

1. Create at least one product file in the temporary directory\.

1. Register the product file\(s\) with `ProcessJobResponse`\.

1. Return a success value in `ProcessJobResponse`\.

Keep in mind the following:
+ Because the callback occurs on a worker thread, do not spawn threads to do the work in your builder class\.
+ Do not interact with other threads during the callback\. The `JobDescriptor` that the callback receives is the `JobDescriptor` that your builder class created in the `CreateJobsFunction` callback\.
+ During the callback, create, modify, or write files only in the temporary directory\. You can use this temporary directory any way that you want\. For example, you can use this directory to write intermediate files for your final products and/or the final products themselves\.
+ After your job succeeds, Asset Processor copies your registered products to the asset cache\. You should never write to the cache directly\.

For an example of a `ProcessJobFunction` callback, see the `CustomAssetExample::ExampleBuilderWorker::ProcessJob()` function in the CustomAssetExample gem in the `lumberyard_version\dev\Gems\CustomAssetExample\Code\Source\CustomAssetExample\Builder\CustomAssetExampleBuilderWorker.cpp` file\.

### E\. \(Optional\) Create a JobCancelListener<a name="asset-builder-custom-create-builder-class-optional-create-jobcancellistener"></a>

Builder classes can use the `JobCancelListener` function to listen for job cancellation requests in their `processJob` method\. Your code should listen for cancellation requests and then cancel work if possible when a request is received\. The address of this listener is the job ID of the job that is in `processJobRequest`\. If additional processing like signaling a semaphore or other threading work is required, you can derive from the `JobCancelListener` and then reimplement `Cancel()`\.

For a basic example of `JobCancelListener`, see the `CustomAssetExample::ExampleBuilderWorker::ProcessJob()` function in the CustomAssetExample gem in the `lumberyard_version\dev\Gems\CustomAssetExample\Code\Source\CustomAssetExample\Builder\CustomAssetExampleBuilderWorker.cpp` file\.

### F\. Shut Down Properly<a name="asset-builder-custom-create-builder-class-shut-down-properly"></a>

When your asset builder application needs to shut down, Asset Processor uses the address of the registered builder's `AZ::Uuid` to broadcast the `Shutdown()` message on the `AssetBuilderSDK::AssetBuilderBus`\. At this point, your builder must stop all tasks and return control to Asset Processor\.

**Important**  
Failure to terminate promptly when Asset Processor shuts down and then restarts can cause your system to stop responding\. The shutdown message comes from a thread that is separate from the `ProcessJob()` thread\.

## 2\. Create a Lifecycle Component<a name="asset-builder-custom-create-a-lifecycle-component"></a>

In this step, you create a lifecycle component that registers your builder class with Asset Processor in the `Activate()` function\.

To register each of the builder classes, your lifecycle component must call the `AssetBuilderSDK::AssetBuilderBus` Ebus\. This registration allows Asset Processor to send requests to the two registered callback functions in your builder class\.

For a code example of a lifecycle component, see the `lumberyard_version\dev\Gems\CustomAssetExample\Code\Source\Builder\CustomAssetExampleBuilderComponent.cpp` file\.

## 3\. Tag Components for Builder Mode<a name="asset-builder-custom-tag-components-for-builder-mode"></a>

In this step, you tag your lifecycle component and all the system components that must be active for your builder worker's `ProcessJobs` and `CreateJobs` functions to work\. The components that you tag can be any components that aid in processing your asset type\. For example, you might tag a component that contains an implementation of an EBus handler that loads your asset type\.

To tag components as mandatory in builder mode, you add an `AZ::Edit::Attributes::SystemComponentTags` attribute to the `AZ::SerializeContext` of each component's `Reflect()` function\. The Asset Builder creates and activates components that you tag as `AssetBuilderSDK::ComponentTags::AssetBuilder`\. 

**Example**  
The following example is from the lifecycle component of the CustomAssetExample gem's `CustomAssetExample::ExampleBuilderComponent::Reflect()` function\.  

```
// Perform static reflection or type registration here of any types that you need the serializer to know about
    void ExampleBuilderComponent::Reflect(AZ::ReflectContext* context)
    {
        if (AZ::SerializeContext* serialize = azrtti_cast<AZ::SerializeContext*>(context))
        {
            serialize->Class<ExampleBuilderComponent, AZ::Component>()
                ->Version(0)
                ->Attribute(AZ::Edit::Attributes::SystemComponentTags, AZStd::vector<AZ::Crc32>({ AssetBuilderSDK::ComponentTags::AssetBuilder }))
            ;
        }
    }
```

The `AZ::Edit::Attributes::SystemComponentTags` attribute takes a single `AZ::Crc32` value or a vector of `AZ::Crc32` values\. The example uses a vector\. Using a vector makes it easier to add tags later in development\.

For the source code, see the `lumberyard_version\dev\Gems\CustomAssetExample\Code\Source\CustomAssetExample\Builder\CustomAssetExampleBuilderComponent.cpp` file\.

### More About Tagging<a name="asset-builder-custom-tag-components-more-about-tagging"></a>

Many system components contain logic and startup systems that can be detrimental to asset builds\. For example, systems that simulate physics, render GUIs, or attempt to acquire device or network contexts can negatively impact asset builder performance\. For this reason, Lumberyard's Asset Processor loads the same set of gems that Lumberyard Editor loads from the `lumberyard_version\dev\project_name\Config\Editor.xml` file, but activates only the components that are tagged as mandatory in builder mode\. This selective activation of tagged components also makes it possible for an arbitrary number of builder and non\-builder components to reside together inside a gem\.

## 4\. \(Optional\) Implement Message Logging<a name="asset-builder-custom-optional-implement-message-logging"></a>

To log builder registration in your lifecycle component, use the following syntax:

```
BuilderLog(AZ::Uuid builderId, const char* message, ...)
```

Note that `BuilderLog` records messages at the application level\. To log job\-specific messages in the builder classes, use standard `AZ_Trace` macros like the following in the function callbacks in your builder class:

```
AZ_TracePrintf(window, msg) 
AZ_Warning(...) 
AZ_Error(...) 
AZ_Assert(...)
```

This ensures that your messages are in the job's log file and display in Asset Processor's log view for that specific job\.

### Logging Resources<a name="asset-builder-custom-logging-resources"></a>

For `BuilderLog` source code, see `lumberyard_version\dev\Code\Tools\AssetProcessor\AssetBuilderSDK\AssetBuilderSDK\AssetBuilderSDK.*`\.

For trace\-related source code, see `lumberyard_version\dev\Code\Framework\AzCore\AzCore\Debug\Trace.*`\.