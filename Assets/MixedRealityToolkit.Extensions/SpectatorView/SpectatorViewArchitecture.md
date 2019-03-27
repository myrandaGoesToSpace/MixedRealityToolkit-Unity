# Spectator View Architecture
While refactoring, Spectator View logic will live in the MixedRealityToolkit.Extensions directory. Extensions are components that may be maintained by Microsoft or non-Microsoft MRTK community members. As an extension, usage of Spectator View should not require dependencies on the rest of the MRTK code base, but it should continue to remain compatible with MRTK projects. Long term, Spectator View may end up in its own repository. But for now, documentation and feature work can be found in both the [Microsoft/MixedRealityToolkit-Unity](https://github.com/Microsoft/MixedRealityToolkit-Unity/tree/feature/spectatorView/Assets/MixedRealityToolkit.Extensions/SpectatorView) and [Microsoft/MixedRealityToolkit](https://github.com/Microsoft/MixedRealityToolkit/tree/feature/spectatorView/SpectatorViewPlugin) repositories in the feature/spectatorView branches.

# Key Components
Many of the key components defined in Spectator View are applicable to other shared experiences. A high level overview of component [interface definitions](../Sharing/) and their intended functionality are outlined below.

## IMatchMakingService, IPlayerService and INetworkingService
* [**IMatchMakingService**](../Sharing/IMatchMakingService.cs) - A match making service is responsible for connecting a single device to a shared app experience.

* [**IPlayerService**](../Sharing/IPlayerService.cs)  - A player service is responsible for providing information on what users exist, enter and leave a shared app experience.

* [**INetworkingService**](../Sharing/INetworkingService.cs) - A networking service is responsible for sending data over a network between users.

## ISpatialCoordinateService
* [**ISpatialCoordinateService**](../Sharing/ISpatialCoordinateService.cs) - A spatial coordinate service enables physically locating a user's device in an app experience. For spectator view, this is a shared app experience. 

## IMarkerDetector and IMarkerVisual
* [**IMarkerDetector**](../MarkerDetection/IMarkerDetector.cs)  - A marker detector is a component capable of locating a marker (ArUco marker, etc) in a local application's 3D 
space.

* [**IMarkerVisual**](../MarkerDetection/IMarkerVisual.cs) - A marker visual is a component capable of showing a marker (ArUco marker, etc). For the default Spectator View experience, mobile devices display ArUco markers on their screen via the IMarkerVisual component.

## IRecordingService and IRecordingServiceVisual
* [**IRecordingService**](../ScreenRecording/IRecordingService.cs) - A recording service is a component capable of recording screen content to a video file. Different platforms (HoloLens, iOS, Android, etc) have different recording service implementations.

* [**IRecordingServiceVisual**](../ScreenRecording/IRecordingServiceVisual.cs) - A recording service visual is aware of IRecordingService functionality and is responsible for the UI/user interactions that kick off any recording logic. 

# High Level Overview
The majority of the Spectator View setup logic can be found in [SpectatorView.cs](https://github.com/Microsoft/MixedRealityToolkit-Unity/blob/feature/spectatorView/Assets/MixedRealityToolkit.Extensions/SpectatorView/Scripts/SpectatorView.cs). Said script is responsible for the following setup process:
1. On Awake(), SpectatorView assesses that it has all of its needed service components. It also registers any service component observers.
2. On Start(), SpectatorView subscribes to any various service component events.
>Note: Subscriptions to service events as well as maintaining service observers is explicitly conducted in SpectatorView to prevent services from taking dependencies on one another. As a richer MRTK shared experience code base is defined and acceptable component dependencies are better understood, this may change.
3. On Update(), SpectatorView interacts with the IMatchMakingService to understand whether or not it has established any network connections with other devices. If no connection is found, it attempts to connect via the IMatchMakingService.
4. On Update(), after SpectatorView establishes a network connection, it begins interacting with the ISpatialCoordinateService to try and obtain the transform from its local app origin to the shared app experience origin. If a valid transform is found, its applied to the scene root GameObject. Applying this transform results in the local app content being positioned relative to the shared app experience origin, which effectively places holograms in the same physical location across devices.

## Setting up the connection
### UDPBroadcastNetworkingService
* [**UDPBroadcastNetworkingService**](Scripts/Sharing/UDPBroadcastNetworkingService.cs) implements [**IMatchMakingService**](../Sharing/IMatchMakingService.cs), [**IPlayerService**](../Sharing/IPlayerService.cs), and [**INetworkingService**](../Sharing/INetworkingService.cs). It provides a UDP communication pipeline for devices on the same local network. A device declares itself as a server or client. Said device will then monitor broadcasts from their server/client counterpart(s). Port values for server and client broadcasts can be modified during application startup through provided or custom UI, which allows for multiple demos on the same local network.

## Setting up shared spatial coordinates
### MarkerSpatialCoordinateService
* [**MarkerSpatialCoordinateService**](Scripts/Sharing/MarkerSpatialCoordinateService.cs) implements [**ISpatialCoordinateService**](../Sharing/ISpatialCoordinateService.cs). It also contains references to [**IMarkerDetector**](../MarkerDetection/IMarkerDetector.cs) and [**IMarkerVisual**](../MarkerDetection/IMarkerVisual.cs) components. The MarkerSpatialCoordinateService allows multiple devices to transform their local application content to be based on a shared application origin. This is achieved via the following steps:

1. A spectator declares that it exists to a main user
2. After the main user discovers the new spectator, it assigns the spectator a marker id
3. After the spectator learns it has been assigned an id, it uses its [IMarkerVisual](../MarkerDetection/IMarkerVisual.cs) component to display said marker
4. The main user then locates the marker in its local application space using its [IMarkerDetector](../MarkerDetection/IMarkerDetector.cs) component
5. The main user then tells the spectator where the marker exists relative to the shared application origin
>Note: The shared application origin for the MarkerSpatialCoordinateService is the main user's local application origin
6. After getting the marker location relative to the shared application origin, the spectator uses this information in addition to the transform from the marker to its camera and the transform of its camera to its own local application origin to build a transform from its local application origin to the shared application origin
7. The transform between application origins can then be obtained from the MarkerSpatialCoordinateService via methods defined in ISpatialCoordinateService to apply to content in the scene
>Note: [SpectatorView.cs](Scripts/SpectatorView.cs) applies the transform for the local application origin to the shared application origin to a root game object. But, this transform could also be applied to the spectator camera compared to any game content.

## Customizing UI
Various components in spectator view utilize UI for a richer user experience. These UI interactions have been abstracted with the following interfaces in hopes of enabling further customization.

### Screen Recording UI
* [**IRecordingServiceVisual**](../ScreenRecording/IRecordingServiceVisual.cs) - Interface definition that enables user interactions with a RecordingService. This allows starting, stopping and viewing recordings.

### UDPBroadcastNetworkingService UI
* [**IUDPBroadcastNetworkingServiceVisual**](Scripts/Sharing/UDPBroadcastNetworkingService.cs) - Interface definition that allows a user to specify server and client ports for the UDPBroadcastNetworkingService. This enables having multiple spectator view experiences run on the same network.

### MarkerSpatialCoordinateService UI
* [**IMarkerSpatialCoordinateServiceOverlayVisual**](Scripts/Sharing/MarkerSpatialCoordinateService.cs) - Interface definition that allows the MarkerSpatialCoordinateService to display state information to the user. This is helpful for guiding users through the marker detection steps.
* [**IMarkerSpatialCoordinateServiceResetVisual**](Scripts/Sharing/MarkerSpatialCoordinateService.cs) - Interface definition that allows a user to reset the MarkerSpatialCoordinateService's coordinate system. This is helpful if an inaccurate marker location is obtained or if a mobile device loses tracking.