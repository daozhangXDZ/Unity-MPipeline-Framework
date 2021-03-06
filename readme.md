# MPipeline Framework
## Abstract:
* MPipeline framework is a design pattern for Unity graphics programmers to develop their own rendering pipeline by using SRP. The goal of developing such a design pattern is, to enhance team-work's efficiency, reducing the coupling between different parts of rendering, and helping project resources management. MPipeline is an object-oriented based framework, store and manage data by using Unity Editor's serialize system.
## Features:
* Designed for Scriptable Rendering Pipeline
* Job System supported
* Toolbox and utility functions supported
* Based on event-system design, reduce high-level coupling.
## Tutorial for Beginners:

### 1. Helloworld!

Before reading this tutorial, I hope you have already studied the basic API in SRP and CommandBuffer, the first "Helloworld" event will be drawing a pure color onto screen, which should be easy to do. Firstly lets create a new script called "PureScreen.cs", and then type such code:

![hello-world](demo/helloworld.png)

We will start to look at these lines, as we declared, all of the rendering events will be ScriptableObject, so we definitely want to create such a instance in editor during developing, so we will have this:    [CreateAssetMenu(menuName = "Demo/Pure Screen Color")], you could easily create one in the Project window. Then we will write two lines of code in FrameUpdate method, this method will be called by the main pipeline later, to set screen as the render target and clear it into a half blue half green color. You do not have to execute the command buffer manually unless it is necessary, the main pipeline code will do it for you.
After having a ScriptableObject instance in your project's folder, we can start doing the second step: create the main pipeline's resources. You can press right mouse button in Project window and click "Create Assets/MPipeline/Create Resources", this will give you a  ScriptableObject of main pipeline, open it and it should be looks like this:

![pipelineresources](demo/res.png)

Then drag the PureScreen object we have already created at the last step into the "Demo Pipeline Events" array, just like the picture showed, and now our first pipeline is finished! you can drag this object into "Edit/Project Settings/Graphics" on the menu now. But how do we use the pipeline? The answer is quite simple, you need only add a component "PipelineCamera.cs" onto the target camera, then this camera will be available for your custom pipeline, there is a enum setting called "rendering path" in PipelineCamera.cs, this enum is just like Camera.RenderingPath in the legacy rendering pipeline, that means you can have multiple different rendering path for different camera, like unlit, forward, deferred, etc. We will discuss this later.

### 2. Property!

Having properties is the most important step in a rendering pipeline, like material, no matter PostProcessing, or Skybox, they all need a material, and now we will show you how to create a property or properties inside the rendering events. In the helloworld step, we can see there are three methods called "Init", "CheckProperty" and "Dispose", that will be called while the main pipeline is initializing and disposing, just like Awake and OnDestroy in MonoBehaviour classes. CheckProperty method is design to prevent Unity editor accidentially remove or dispose some native properties, this method will only be called in Editor mode. So we will create the properties inside Init method, and dispose it(if necessary) in the Dispose method, For example, lets use a material to redo the helloworld again:

![property](demo/prop.png)

As you can see, we are loading the material from "unlitShader", which is a regular serialized value, you can also store and load some properties in PipelineResources, the advantage to store resources in PipelineResources is the developing team could manage a unique piece of resources, and the disadvantage is such behavior may produce more coupling, for now, we just create the material from local variable just for Demo. After compiling code, we can add the new shader into the resources and load the pipeline(you always want to disload the pipeline when you are working on that, otherwise some errors or crashes may happened):

![property-resources](demo/propRes.png)

### 3. Use SRP's API!

As we declared before, MPipeline is an SRP based framework, so we passed the necessary datas from PipelineCommandData, which is a struct:

![commanddata](demo/commanddata.png)

The comment is pretty clear enough, so we can just write a simple pure-color pipeline drawing onto the screen:

![srpsource](demo/srpsource.png)

It is a pretty simple and clear demo code here, just draw a single pass which has tag "LightMode" = "Unlit", the result will be looks like this:

![srptest](demo/srptest.png)

## Features:

### Events dependent:

Although we are trying to use such design pattern to avoid coupling, there are some coupling is useful and necessary, such as depth texture, for example an SSAO post processing will need Geometry Buffer like Depth Map, Normal Map, Specular Map, etc. But these maps has been generated from a previous event, so if the GBuffers are absence, there will be an unhappy result, so we want to have a dependent mark to another events:

![requireevent](demo/requireevent.png)

By using RequireEventAttribute, when the main framework found the target type of events in the rendering path list, a connection will be created between those two events. In this example, the event "VolumetricLightEvent" is always disabled if the event "LightingEvent" is disabled, that means the property "Enabled" will always be False.
To get another events is also sometimes necessary, so we have the API: 
    MPipeline.RenderPipeline.GetEvent<T>(PipelineResources.CameraRenderingPath path) where T : PipelineEvent
    MPipeline.RenderPipeline.GetEvent(PipelineResources.CameraRenderingPath path, Type targetType)

![getevent](demo/getevent.png)

### Having Multiple Rendering Paths:

As mentioned above, usually we may have different types of rendering path in one project, to change the source code in PipelineResources.cs, we can easily have multiple rendering path, firstly we need to add several available paths in the enum in PipelineResources.cs:

![pathsenum](demo/pathsenum.png)

For now you will be able to choose a path in PipelineCamera.cs, then you have to add an array of PipelineEvent into the global dictionary:

![addnewevents](demo/addnewevents.png)

### Job system support:

We want to use job system to do some calculations, which could gain performance benefits from multi-threading, to use job system, you can do the job schedule by overriding the PreRenderFrame method in PipelineEvent.cs, and a JobHandle.ScheduleBatchedJobs() will be called after all of the PreRenderFrame methods being finished and before any FrameUpdate methods being start. According to Unity's official tutorial, we should call ScheduleBatchedJobs as less as we can, so the scheduled jobs should be batched together.