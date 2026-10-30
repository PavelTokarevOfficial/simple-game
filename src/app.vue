
<script setup lang="ts">
import { onMounted, onBeforeUnmount, ref } from "vue";

import { Scene } from "@babylonjs/core/scene";
import { Engine } from "@babylonjs/core/Engines/engine";
import { Vector3 } from "@babylonjs/core/Maths/math.vector";
import { ArcRotateCamera } from "@babylonjs/core/Cameras/arcRotateCamera";
import { TransformNode } from "@babylonjs/core/Meshes/transformNode";
import { SceneLoader } from "@babylonjs/core/Loading/sceneLoader";
import { SceneLoaderFlags } from "@babylonjs/core/Loading/sceneLoaderFlags";
import { HavokPlugin } from "@babylonjs/core/Physics/v2/Plugins/havokPlugin";

import HavokPhysics from "@babylonjs/havok";

import "@babylonjs/core/Loading/loadingScreen";
import "@babylonjs/core/Loading/Plugins/babylonFileLoader";
import "@babylonjs/loaders/glTF";

import "@babylonjs/core/Cameras/camera";
import "@babylonjs/core/Cameras/universalCamera";
import "@babylonjs/core/Cameras/arcRotateCamera";

import "@babylonjs/core/Meshes/groundMesh";

import "@babylonjs/core/Lights/directionalLight";
import "@babylonjs/core/Lights/Shadows/shadowGeneratorSceneComponent";
import "@babylonjs/core/Lights/Shadows/cascadedShadowGenerator";

import "@babylonjs/core/Materials/PBR/pbrMaterial";
import "@babylonjs/core/Materials/standardMaterial";
import "@babylonjs/core/Materials/imageProcessingConfiguration";

import "@babylonjs/core/XR/features/WebXRDepthSensing";

import "@babylonjs/core/Rendering/depthRendererSceneComponent";
import "@babylonjs/core/Rendering/prePassRendererSceneComponent";

import "@babylonjs/core/Materials/Textures/cubeTexture";
import "@babylonjs/core/Materials/Textures/Loaders/envTextureLoader";
// Bundle the sky shaders directly; Babylon's dynamic shader URL resolves to the
// Nuxt HTML fallback in this app instead of the shader modules.
import "@babylonjs/materials/sky/sky.vertex";
import "@babylonjs/materials/sky/sky.fragment";
import "@babylonjs/core/Shaders/shadowMap.vertex";
import "@babylonjs/core/Shaders/shadowMap.fragment";
import "@babylonjs/core/Shaders/lightProxy.vertex";
import "@babylonjs/core/Shaders/lightProxy.fragment";

import "@babylonjs/core/Physics";

import "@babylonjs/materials/sky";

import { loadScene } from "babylonjs-editor-tools";
import { scriptsMap } from "@/scripts";

const config = useRuntimeConfig();

const canvasRef = ref<HTMLCanvasElement | null>(null);

let engine: Engine | null = null;
let scene: Scene | null = null;
let resizeListener: (() => void) | null = null;
let inputListener: ((event: KeyboardEvent) => void) | null = null;
let keyUpListener: ((event: KeyboardEvent) => void) | null = null;
let blurListener: (() => void) | null = null;

/**
 * Загружаем игрового персонажа.
 */
async function loadPlayer(scene: Scene) {
    const baseURL = config.app.baseURL;

    const result = await SceneLoader.ImportMeshAsync(
        "",
        `${baseURL}assets/characters/`,
        "player.glb",
        scene
    );

    // GLB импортирует риг отдельными узлами. Контейнер двигает их все вместе.
    const player = new TransformNode("player-controller", scene);
    const importedRoot = result.meshes.find((mesh) => mesh.name === "__root__");
    const roots = importedRoot
        ? [importedRoot]
        : result.meshes.filter((mesh) => !mesh.parent);
    roots.forEach((root) => {
        root.parent = player;
        root.position.setAll(0);
        root.rotationQuaternion = null;
        root.rotation.setAll(0);
        root.scaling.setAll(1);
    });
    player.position.set(160, 0, 0);
    player.scaling.setAll(100);

    return {
        player,
        animationGroups: result.animationGroups,
    };
}

/**
 * Загружаем сцену и игровые ресурсы.
 */
async function handleLoad(engine: Engine, scene: Scene) {
    const havok = await HavokPhysics();

    scene.enablePhysics(
        new Vector3(0, -981, 0),
        new HavokPlugin(true, havok)
    );

    SceneLoaderFlags.ForceFullSceneLoadingForIncremental = true;

    // Загружаем сцену из Babylon Editor
    await loadScene(
        `${config.app.baseURL}scene/`,
        "example.babylon",
        scene,
        scriptsMap,
        {
            quality: "high",
        }
    );

    // Загружаем персонажа
    const { player, animationGroups } = await loadPlayer(scene);

    // Камера следует за героем; вращение мышью меняет направление движения.
    const camera = new ArcRotateCamera(
        "third-person-camera",
        -Math.PI / 2,
        Math.PI / 2.8,
        8,
        player.position.add(new Vector3(0, 120, 0)),
        scene
    );
    camera.lowerRadiusLimit = 400;
    camera.upperRadiusLimit = 1200;
    camera.lowerBetaLimit = 0.35;
    camera.upperBetaLimit = Math.PI / 2.05;
    camera.wheelPrecision = 60;
    // Стрелки используются для движения, поэтому отключаем управление камерой с клавиатуры.
    camera.keysUp = [];
    camera.keysDown = [];
    camera.keysLeft = [];
    camera.keysRight = [];
    camera.attachControl(canvasRef.value, true);
    // Камера остаётся за персонажем; колесо мыши по-прежнему меняет дистанцию.
    camera.inputs.attached.pointers.buttons = [];
    scene.activeCamera = camera;

    const keys = new Set<string>();
    const idle = animationGroups.find((group) => group.name === "Idle_Loop");
    const jog = animationGroups.find((group) => group.name === "Jog_Fwd_Loop");
    const moveSpeed = 400;
    let activeAnimation = "";
    let verticalVelocity = 0;
    let grounded = true;
    let wasGrounded = true;
    const jumpSpeed = 560;
    const gravity = 1600;

    const playAnimation = (group: (typeof animationGroups)[number] | undefined) => {
        if (!group || activeAnimation === group.name) return;
        animationGroups.forEach((candidate) => {
            if (candidate !== group) candidate.stop();
        });
        group.start(true);
        activeAnimation = group.name;
    };
    playAnimation(idle ?? animationGroups[0]);

    inputListener = (event) => {
        const key = event.key.toLowerCase();
        if (["w", "a", "s", "d", "arrowup", "arrowdown", "arrowleft", "arrowright", " "].includes(key)) {
            event.preventDefault();
            keys.add(key);
        }
    };
    keyUpListener = (event) => keys.delete(event.key.toLowerCase());
    blurListener = () => keys.clear();
    window.addEventListener("keydown", inputListener);
    window.addEventListener("keyup", keyUpListener);
    window.addEventListener("blur", blurListener);

    scene.onBeforeRenderObservable.add(() => {
        const delta = Math.min(engine.getDeltaTime() / 1000, 0.05);
        if (keys.has(" ") && grounded) {
            grounded = false;
            verticalVelocity = jumpSpeed;
            playAnimation(animationGroups.find((group) => group.name === "Jump_Start") ?? animationGroups.find((group) => group.name === "Jump_Loop"));
        }

        const forward = (keys.has("w") || keys.has("arrowup") ? 1 : 0) - (keys.has("s") || keys.has("arrowdown") ? 1 : 0);
        const strafe = (keys.has("d") || keys.has("arrowright") ? 1 : 0) - (keys.has("a") || keys.has("arrowleft") ? 1 : 0);
        const moving = forward !== 0 || strafe !== 0;
        if (moving) {
            const facing = new Vector3(
                Math.sin(player.rotation.y),
                0,
                Math.cos(player.rotation.y)
            );
            const right = new Vector3(facing.z, 0, -facing.x);
            const movement = facing.scale(forward).add(right.scale(strafe));
            movement.normalize();
            player.position.addInPlace(movement.scale(moveSpeed * delta));
            const targetRotation = Math.atan2(movement.x, movement.z);
            const turnAmount = 1 - Math.exp(-10 * delta);
            const rotationDelta = Math.atan2(
                Math.sin(targetRotation - player.rotation.y),
                Math.cos(targetRotation - player.rotation.y)
            );
            player.rotation.y += rotationDelta * turnAmount;
        }

        wasGrounded = grounded;
        if (!grounded) {
            verticalVelocity -= gravity * delta;
            player.position.y = Math.max(0, player.position.y + verticalVelocity * delta);
            if (player.position.y === 0) {
                grounded = true;
                verticalVelocity = 0;
            }
        }

        const cameraTurnAmount = 1 - Math.exp(-6 * delta);
        const targetCameraAlpha = -player.rotation.y - Math.PI / 2;
        const cameraAlphaDelta = Math.atan2(
            Math.sin(targetCameraAlpha - camera.alpha),
            Math.cos(targetCameraAlpha - camera.alpha)
        );
        camera.alpha += cameraAlphaDelta * cameraTurnAmount;
        camera.target.copyFrom(player.position.add(new Vector3(0, 120, 0)));
        if (!grounded) {
            const jump = animationGroups.find((group) => group.name === "Jump_Loop");
            if (jump && activeAnimation !== jump.name) playAnimation(jump);
        } else if (!wasGrounded) {
            const land = animationGroups.find((group) => group.name === "Jump_Land");
            if (land) {
                playAnimation(land);
                land.onAnimationGroupEndObservable.addOnce(() => {
                    if (grounded) {
                        activeAnimation = "";
                        playAnimation(idle ?? animationGroups[0]);
                    }
                });
            }
        } else {
            playAnimation(moving ? jog ?? idle ?? animationGroups[0] : idle ?? animationGroups[0]);
        }
    });

    // Запускаем рендеринг
    engine.runRenderLoop(() => {
        scene.render();
    });
}

onMounted(() => {
    if (!canvasRef.value) {
        return;
    }

    const isMobile = /iPhone|iPad|Android/i.test(navigator.userAgent);

    engine = new Engine(canvasRef.value, true, {
        stencil: true,
        antialias: !isMobile,
        audioEngine: true,
        adaptToDeviceRatio: false,
        disableWebGL2Support: false,
        useHighPrecisionFloats: true,
        powerPreference: "high-performance",
        failIfMajorPerformanceCaveat: false,
    });

    // Снижаем разрешение рендеринга на мобильных устройствах
    engine.setHardwareScalingLevel(isMobile ? 1.5 : 1);

    scene = new Scene(engine);

    handleLoad(engine, scene).catch((error) => {
        console.error("Scene loading error:", error);
    });

    resizeListener = () => {
        engine?.resize();
    };

    window.addEventListener("resize", resizeListener);
});

onBeforeUnmount(() => {
    scene?.dispose();
    engine?.dispose();

    if (resizeListener) {
        window.removeEventListener("resize", resizeListener);
    }
});
</script>

<template>
    <main class="app-root">
        <canvas ref="canvasRef" class="app-canvas"></canvas>
    </main>
</template>


<style>
html,
body,
#__nuxt {
	margin: 0;
	padding: 0;
	width: 100vw;
	height: 100vh;
	overflow: hidden;
}

.app-root {
	display: flex;
	flex-direction: column;
	align-items: center;
	justify-content: space-between;
	width: 100vw;
	height: 100vh;
}

.app-canvas {
	width: 100%;
	height: 100%;
	outline: none;
	user-select: none;
}
</style>
