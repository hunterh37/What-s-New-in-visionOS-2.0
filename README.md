## **What's New in visionOS 2.0 - WWDC 2024**

WWDC 24 videos
https://developer.apple.com/videos/wwdc2024/

visionOS Documentation home
https://developer.apple.com/documentation/visionos

-----------------------------------------------


**ARKit**
* ObjectTrackingProvider
* ObjectAnchor
* ReferenceObject
* SpatialTrackingSession


**RealityKit**
- AnimationLibraryComponent (https://developer.apple.com/documentation/RealityKit/AnimationLibraryComponent)
- BlendShapeWeightsComponent (BlendShapeWeightsComponent)
- ForceEffects
- PhysicsJoints
- AnimationBlending
- Dynamic Lights and shadows
- SurroundingsEffect 


**Enterprise API’s**
- Main Camera Access
- Passthrough In-screen Capture
- Spatial barcode and QR code scanning
- Object tracking parameter adjustment
- Increased performance


**Volumes**
- Baseplate
    - .volumeBaseplateVisibility(.visible) // Default!
- Programmatic resize
    - .windowResizability(.contentSize) // Default!
- Viewpoint change
    - RealityView.onVolumeViewpointChange


**Immersive Spaces**
- Immersion Amount change
    - ImmersiveSpace.onImmersionChange


---------------------------------------------


## **WWDC 2024 Videos (visionOS)** 

- Discover RealityKit APIs for iOS, macOS and visionOS:
    - ForceEffects, PhysicsJoints, Dynamic Lights and shadows
https://developer.apple.com/videos/play/wwdc2024/10103/

- Dive deep into volumes and immersive spaces
    - Customize volumes and immersive spaces 
https://developer.apple.com/videos/play/wwdc2024/10153/

- Design Great visionOS Apps
https://developer.apple.com/videos/play/wwdc2024/10086/


---------------------------------------------

## **Apple visionOS Sample Projects  (WWDC 2024)**
- BOT-anist https://developer.apple.com/documentation/visionos/bot-anist
- Exploring Object Tracking with ARKit https://developer.apple.com/documentation/visionos/exploring_object_tracking_with_arkit

- Creating a Spaceship Game https://developer.apple.com/documentation/realitykit/creating-a-spaceship-game
- Creating a Spatial Drawing App with RealityKit https://developer.apple.com/documentation/realitykit/creating-a-spatial-drawing-app-with-realitykit
- Customizing SharePlay Templates https://developer.apple.com/documentation/GroupActivities/customizing-spatial-persona-templates




---------------------------------------------



## **Sample Code**
 


       

Loading multiple entities concurrently:  BOT-anist , RobotProvider.swift:

```
import Foundation
import RealityKit
import BOTanistAssets

/// A class that loads and provides the robot entities and materials.
@MainActor
final public class RobotProvider: Sendable {
    
    static let shared = RobotProvider()
    
    private var loadCompleteListeners = [(RobotProvider) -> Void]()
    
    var materials = [RobotPart: [RobotMaterial: [ShaderGraphMaterial]]]()
    var robotParts = [RobotPart: [Entity]]()

    var robotsLoaded = false
    
    init() {
        Task { [self] in
            // Load models and meshes concurrently
            await withTaskGroup(of: RobotPartLoadResult.self) { taskGroup in
                await loadRobotParts(taskGroup: &taskGroup)
                
                for await result in taskGroup {
                    Task { @MainActor in
                        var parts = robotParts[result.type] ?? [Entity]()
                        parts.append(result.entity)
                        parts.sort { left, right in
                            return left.name.compare(right.name) == ComparisonResult.orderedAscending
                        }
                        robotParts[result.type] = parts
                    }
                }
            }

            await withTaskGroup(of: RobotMaterialResult.self) { taskGroup in
                await loadRobotMaterials(taskGroup: &taskGroup)
                
                for await result in taskGroup {
                    for partKey in result.materials.keys {
                        if self.materials[partKey] == nil {
                            self.materials[partKey] = [RobotMaterial: [ShaderGraphMaterial]]()
                        }
                        if let shaders = result.materials[partKey] {
                            if self.materials[partKey]?[result.material] == nil {
                                self.materials[partKey]?[result.material] = [ShaderGraphMaterial]()
                            }
                            self.materials[partKey]?[result.material]?.append(contentsOf: shaders)
                        }
                    }
                    
                }

                robotsLoaded = true
                for listener in loadCompleteListeners {
                    listener(self)
                }
            }
        }
    }

    public func listenForLoadComplete(completion: @escaping (RobotProvider) -> Void) {
        loadCompleteListeners.append(completion)
        if robotsLoaded {
            completion(self)
        }
    }
    
    /// Returns the mesh entity corresponding to the provided robot part and index.
    func getMesh(forPart part: RobotPart, index: Int) -> Entity {
        guard let parts = robotParts[part] else { fatalError("Failed to find expected robot part.") }
        let entity = parts[index]
        entity.components.set(InputTargetComponent())
        return entity
    }
    
    /// Returns the mesh entity corresponding to the provided robot part and name.
    func getMesh(forPart part: RobotPart, name: String) -> Entity {
        guard let parts = robotParts[part] else { fatalError("Failed to find expected robot part.") }
        guard let entity = parts.first(where: { $0.name == name }) else { fatalError() }
        entity.components.set(InputTargetComponent())
        return entity
    }
    
    /// Returns the shader graph material corresponding to the provided robot part, material type, and index.
    func getMaterial(forPart part: RobotPart, material: RobotMaterial, index: Int) -> ShaderGraphMaterial {
        guard let materials = materials[part]?[material] else { fatalError("Failed ot find expected shader graph material.") }
        return materials[index]
    }
}
```


Loading animations at runtime into Sendable
See: BOT-anist, PlantAnimationProvider.swift

```
import Foundation
import RealityKit
import BOTanistAssets
import SwiftUI
import Spatial

/// An object that loads and provides all the necessary animations for each plant model.
@MainActor
class PlantAnimationProvider: Sendable {
    static var shared = PlantAnimationProvider()
    
    /// A dictionary of the grow animations for each type of plant.
    public var growAnimations = [PlantComponent.PlantTypeKey: AnimationResource]()
    /// A dictionary of the celebration animations for each type of plant.
    public var celebrateAnimations = [PlantComponent.PlantTypeKey: AnimationResource]()
    /// A dictionary of the current animation controllers for each type of plant.
    public var currentGrowAnimations = [PlantComponent.PlantTypeKey: AnimationPlaybackController]()
    
    init() {
        Task { @MainActor in
            await withTaskGroup(of: PlantAnimationResult.self) { taskGroup in
                for plantType in PlantComponent.PlantTypeKey.allCases {
                    taskGroup.addTask {
                        let growAnim = await self.generateGrowAnimationResource(for: plantType)
                        let celeAnim = await self.generateCelebrateAnimationResource(for: plantType)
                        return PlantAnimationResult(growAnim: growAnim, celebrateAnim: celeAnim, plantType: plantType)
                    }
                }
                for await result in taskGroup {
                    growAnimations[result.plantType] = result.growAnim
                    celebrateAnimations[result.plantType] = result.celebrateAnim
                }
            }
        }
    }
    
    /// Loads the grow animation for the given plant type.
    private func generateGrowAnimationResource(for plantType: PlantComponent.PlantTypeKey) async -> AnimationResource {
        let sceneName = "Assets/plants/animations/\(plantType.rawValue)_grow_anim"
        var ret: AnimationResource? = nil
        do {
            let rootEntity = try await Entity(named: sceneName, in: BOTanistAssetsBundle)
            rootEntity.forEachDescendant(withComponent: BlendShapeWeightsComponent.self) { entity, component in
                ret = entity.availableAnimations[0]
            }
            guard let ret else { fatalError("Animation resource unexpectedly nil.") }
            return ret
        } catch {
            fatalError("Error: \(error.localizedDescription)")
        }
    }
    
    /// Loads the celebration animation for the given plant type.
    private func generateCelebrateAnimationResource(for plantType: PlantComponent.PlantTypeKey) async -> AnimationResource {
        let sceneName = "Assets/plants/animations/\(plantType.rawValue)_celebrate_anim"
        var ret: AnimationResource? = nil
        do {
            let rootEntity = try await Entity(named: sceneName, in: BOTanistAssetsBundle)
             rootEntity.forEachDescendant(withComponent: BlendShapeWeightsComponent.self) { entity, component in
                ret = entity.availableAnimations[0]
             }
            guard let ret else { fatalError("Animation resource unexpectedly nil.") }
            return ret
        } catch {
            fatalError("Error: \(error.localizedDescription)")
        }
    }
}

@MainActor
public struct PlantAnimationResult: Sendable {
    var growAnim: AnimationResource
    var celebrateAnim: AnimationResource
    var plantType: PlantComponent.PlantTypeKey
}
```


Set SurroundingsEffect tint color on RealityView
```
// Apply effect to tint passthrough
struct ImmersiveExplorationView: View {
    var body: some View {
        RealityView { content in
            // ...
        }
        .preferredSurroundingsEffect(surroundingsEffect)
    }

    // The resolved surroundings effect based on tint color
    var surroundingsEffect: SurroundingsEffect? {
        if let color = appModel.tintColor {
            return SurroundingsEffect.colorMultiply(color)
        } else {
            return nil
        }
    }
}
```



Create entities for hand tracking + Example usage of wrapping entities inside container
See: Spaceship: HandShipControlProvider.swift

```
  static func makeHandTrackingEntities() -> Entity {
        let container = Entity()
        container.name = "HandTrackingEntitiesContainer"

        let leftHand = AnchorEntity(.hand(.left, location: .palm))
        leftHand.components.set(HandTrackingComponent(location: .leftPalm))

        let rightHand = AnchorEntity(.hand(.right, location: .palm))
        rightHand.components.set(HandTrackingComponent(location: .rightPalm))

        let indexTip = AnchorEntity(.hand(.left, location: .indexFingerTip))
        indexTip.components.set(HandTrackingComponent(location: .leftIndexFingerTip))

        let thumbTip = AnchorEntity(.hand(.left, location: .thumbTip))
        thumbTip.components.set(HandTrackingComponent(location: .leftThumbTip))

        container.addChild(leftHand)
        container.addChild(rightHand)
        container.addChild(indexTip)
        container.addChild(thumbTip)

        return container
    }
```
