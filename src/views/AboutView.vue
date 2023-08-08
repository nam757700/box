<template>
  <div class="content">
    <div class="page">
      <div class="container">
        <canvas id="box-canvas"></canvas>
        <div class="ui-controls">
          <button
            class="unbutton ui-controls__button"
            id="zoom-in"
            aria-label="Zoom in"
          >
            +
          </button>
          <button
            class="unbutton ui-controls__button"
            id="zoom-out"
            aria-label="Zoom out"
          >
            -
          </button>
          <div>Scroll to animate</div>
        </div>
      </div>
    </div>
  </div>
</template>
<script>
import * as THREE from "three";
import { OrbitControls } from "three/addons/controls/OrbitControls.js";
import { mergeBufferGeometries } from "three/addons/utils/BufferGeometryUtils.js";
import { GUI } from "three/addons/libs/lil-gui.module.min.js";
import { useWindowSize } from "@vueuse/core";
import { watch, onMounted, ref, computed } from "vue";
import { gsap } from "gsap";
import ScrollTrigger from "gsap/ScrollTrigger";
export default {
  name: "About",
  setup() {
    // const container = ref(null);
    // const boxCanvas = ref(null);

    let box = {
      params: {
        width: 50,
        widthLimits: [15, 70],
        length: 50,
        lengthLimits: [15, 120],
        depth: 100,
        depthLimits: [15, 70],
        thickness: 0.6,
        thicknessLimits: [0.1, 1],
        fluteFreq: 5,
        fluteFreqLimits: [3, 7],
        flapGap: 1,
        copyrightSize: [27, 10],
      },
      els: {
        group: new THREE.Group(),
        backHalf: {
          width: {
            top: new THREE.Mesh(),
            side: new THREE.Mesh(),
            bottom: new THREE.Mesh(),
          },
          length: {
            top: new THREE.Mesh(),
            side: new THREE.Mesh(),
            bottom: new THREE.Mesh(),
          },
        },
        frontHalf: {
          width: {
            top: new THREE.Mesh(),
            side: new THREE.Mesh(),
            bottom: new THREE.Mesh(),
          },
          length: {
            top: new THREE.Mesh(),
            side: new THREE.Mesh(),
            bottom: new THREE.Mesh(),
          },
        },
      },

      animated: {
        openingAngle: 0.02 * Math.PI,
        flapAngles: {
          backHalf: {
            width: {
              top: 0,
              bottom: 0,
            },
            length: {
              top: 0,
              bottom: 0,
            },
          },
          frontHalf: {
            width: {
              top: 0,
              bottom: 0,
            },
            length: {
              top: 0,
              bottom: 0,
            },
          },
        },
      },
    };
    let renderer,
      scene,
      camera,
      orbit,
      lightHolder,
      rayCaster,
      mouse,
      copyright,
      copyright2,
      copyright3,
      copyright4;
    gsap.registerPlugin(ScrollTrigger);

    // const { width, height } = useWindowSize();
    function updateSceneSize() {
      const container = document.querySelector(".container");
      camera.aspect = container.clientWidth / container.clientHeight;
      camera.updateProjectionMatrix();
      renderer.setSize(container.clientWidth, container.clientHeight);
    }

    function setGeometryHierarchy() {
      box.els.group.add(
        box.els.frontHalf.width.side,
        box.els.frontHalf.length.side,
        box.els.backHalf.width.side,
        box.els.backHalf.length.side
      );
      box.els.frontHalf.width.side.add(
        box.els.frontHalf.width.top,
        box.els.frontHalf.width.bottom
      );
      box.els.frontHalf.length.side.add(
        box.els.frontHalf.length.top,
        box.els.frontHalf.length.bottom
      );
      box.els.backHalf.width.side.add(
        box.els.backHalf.width.top,
        box.els.backHalf.width.bottom
      );
      box.els.backHalf.length.side.add(
        box.els.backHalf.length.top,
        box.els.backHalf.length.bottom
      );
    }

    function createBoxElements() {
      for (let halfIdx = 0; halfIdx < 2; halfIdx++) {
        for (let sideIdx = 0; sideIdx < 2; sideIdx++) {
          const half = halfIdx ? "frontHalf" : "backHalf";
          const side = sideIdx ? "width" : "length";

          const sideWidth =
            side === "width" ? box.params.width : box.params.length;
          const flapWidth = sideWidth - 2 * box.params.flapGap;
          const flapHeight =
            // 0.5 * box.params.width - 0.75 * box.params.flapGap;
            0.5 * box.params.width - 0.75 * box.params.flapGap;

          const sidePlaneGeometry = new THREE.PlaneGeometry(
            sideWidth,
            box.params.depth,
            Math.floor(5 * sideWidth),
            Math.floor(0.2 * box.params.depth)
          );

          const flapPlaneGeometry = new THREE.PlaneGeometry(
            flapWidth,
            flapHeight,
            Math.floor(5 * flapWidth),
            Math.floor(0.2 * flapHeight)
          );

          const sideGeometry = createSideGeometry(
            sidePlaneGeometry,
            [sideWidth, box.params.depth],
            [true, true, true, true],
            false
          );
          const topGeometry = createSideGeometry(
            flapPlaneGeometry,
            [flapWidth, flapHeight],
            [false, false, true, false],
            true
          );
          const bottomGeometry = createSideGeometry(
            flapPlaneGeometry,
            [flapWidth, flapHeight],
            [true, false, false, false],
            true
          );

          topGeometry.translate(0, 0.5 * flapHeight, 0);
          bottomGeometry.translate(0, -0.5 * flapHeight, 0);

          box.els[half][side].top.geometry = topGeometry;
          box.els[half][side].side.geometry = sideGeometry;
          box.els[half][side].bottom.geometry = bottomGeometry;
          copyright3.geometry = sideGeometry;
          copyright4.geometry = sideGeometry;

          box.els[half][side].top.position.y = 0.5 * box.params.depth;
          box.els[half][side].bottom.position.y = -0.5 * box.params.depth;
        }
      }

      updatePanelsTransform();
    }

    function createSideGeometry(baseGeometry, size, folds, hasMiddleLayer) {
      const geometriesToMerge = [];
      geometriesToMerge.push(
        getLayerGeometry(
          (v) =>
            -0.5 * box.params.thickness +
            0.01 * Math.sin(box.params.fluteFreq * v)
        )
      );

      geometriesToMerge.push(
        getLayerGeometry(
          (v) =>
            0.5 * box.params.thickness +
            0.01 * Math.sin(box.params.fluteFreq * v)
        )
      );
      if (hasMiddleLayer) {
        geometriesToMerge.push(
          getLayerGeometry(
            (v) =>
              0.5 * box.params.thickness * Math.sin(box.params.fluteFreq * v)
          )
        );
      }

      function getLayerGeometry(offset) {
        const layerGeometry = baseGeometry.clone();
        const positionAttr = layerGeometry.attributes.position;
        for (let i = 0; i < positionAttr.count; i++) {
          const x = positionAttr.getX(i);
          const y = positionAttr.getY(i);
          let z = positionAttr.getZ(i) + offset(x);
          z = applyFolds(x, y, z);
          positionAttr.setXYZ(i, x, y, z);
        }
        return layerGeometry;
      }

      function applyFolds(x, y, z) {
        let modifier = (c, s) => 1 - Math.pow(c / (0.5 * s), 10);
        if ((x > 0 && folds[1]) || (x < 0 && folds[3])) {
          z *= modifier(x, size[0]);
        }
        if ((y > 0 && folds[0]) || (y < 0 && folds[2])) {
          z *= modifier(y, size[1]);
        }
        return z;
      }

      const mergedGeometry = new mergeBufferGeometries(
        geometriesToMerge,
        false
      );
      mergedGeometry.computeVertexNormals();

      return mergedGeometry;
    }

    // End of box geometries
    // --------------------------------------------------

    // --------------------------------------------------
    // Clickable copyright

    function createCopyright() {
      const canvas = document.createElement("canvas");
      const canvas1 = document.createElement("canvas");
      canvas.width = box.params.copyrightSize[0] * 10;
      canvas.height = box.params.copyrightSize[1] * 10;
      const planeGeometry = new THREE.PlaneGeometry(
        box.params.copyrightSize[0],
        box.params.copyrightSize[1]
      );

      const ctx = canvas.getContext("2d");
      const ctx1 = canvas1.getContext("2d");
      // ctx.clearRect(0, 0, canvas.width, canvas.width);
      ctx.fillStyle = "#000000";
      ctx.font = "30px sans-serif";
      ctx.textAlign = "end";
      ctx.fillText("Rikkei", 135, 50);
      ctx.lineWidth = 2;

      ctx1.fillStyle = "#000000";
      ctx1.font = "30px sans-serif";
      ctx1.textAlign = "end";
      ctx1.fillText("Rikkei2", canvas.width - 50, 90);
      ctx1.lineWidth = 2;

      const texture = new THREE.CanvasTexture(canvas);
      const texture2 = new THREE.CanvasTexture(canvas1);
      copyright = new THREE.Mesh(
        planeGeometry,
        new THREE.MeshBasicMaterial({
          // map: texture,
          map: new THREE.TextureLoader().load("/images/earth.jpg"),
          transparent: true,
          opacity: 1,
        })
      );
      copyright2 = new THREE.Mesh(
        planeGeometry,
        new THREE.MeshBasicMaterial({
          map: texture,
          transparent: true,
          opacity: 1,
        })
      );
      copyright3 = new THREE.Mesh(
        planeGeometry,
        new THREE.MeshBasicMaterial({
          map: texture2,
          transparent: true,
          opacity: 1,
        })
      );
      copyright4 = new THREE.Mesh(
        planeGeometry,
        new THREE.MeshBasicMaterial({
          map: texture,
          transparent: true,
          opacity: 1,
        })
      );
      const texture1 = new THREE.TextureLoader().load("/images/earth.jpg");
      texture1.wrapS = THREE.RepeatWrapping;
      texture1.wrapT = THREE.RepeatWrapping;
      texture1.repeat.set(4, 4);

      scene.add(copyright);
      scene.add(copyright2);
      scene.add(copyright3);
      scene.add(copyright4);
      // trackLinks();
    }

    // function trackLinks() {
    //   document.addEventListener(
    //     "mousemove",
    //     (e) => {
    //       updateMousePosition(e.clientX, e.clientY);
    //       checkCopyrightIntersect();
    //     },
    //     false
    //   );
    //   document.addEventListener("click", (e) => {
    //     updateMousePosition(
    //       e.targetTouches ? e.targetTouches[0].pageX : e.clientX,
    //       e.targetTouches ? e.targetTouches[0].pageY : e.clientY
    //     );
    //     const linkCode = checkCopyrightIntersect();
    //     if (linkCode === 1) {
    //       window.open("https://ksenia-k.com/", "_blank").focus();
    //     } else if (linkCode === 2) {
    //       window.open("https://codepen.io/ksenia-k/", "_blank").focus();
    //     }
    //   });

    //   function updateMousePosition(x, y) {
    //     mouse.x = (x / window.innerWidth) * 2 - 1;
    //     mouse.y = (-y / window.innerHeight) * 2 + 1;
    //   }
    // }

    function checkCopyrightIntersect() {
      let linkHovered = 0;
      rayCaster.setFromCamera(mouse, camera);
      const intersects = rayCaster.intersectObject(copyright);
      if (intersects.length) {
        document.body.style.cursor = "pointer";
        linkHovered = intersects[0].uv.y > 0.5 ? 1 : 2;
      } else {
        document.body.style.cursor = "auto";
      }
      return linkHovered;
    }

    // End of Clickable copyright
    // --------------------------------------------------

    // --------------------------------------------------
    // Animation

    function createFoldingAnimation() {
      gsap
        .timeline({
          scrollTrigger: {
            trigger: ".page",
            start: "0% 0%",
            end: "100% 100%",
            scrub: true,
          },
          onUpdate: () => {
            updatePanelsTransform();
            checkCopyrightIntersect();
          },
        })
        .to(box.animated, {
          duration: 1,
          openingAngle: 0.5 * Math.PI,
          ease: "power1.inOut",
        })
        .to(
          [
            box.animated.flapAngles.backHalf.width,
            box.animated.flapAngles.frontHalf.width,
          ],
          {
            duration: 0.6,
            bottom: 0.6 * Math.PI,
            ease: "back.in(3)",
          },
          0.9
        )
        .to(
          box.animated.flapAngles.backHalf.length,
          {
            duration: 0.7,
            bottom: 0.5 * Math.PI,
            ease: "back.in(2)",
          },
          1.1
        )
        .to(
          box.animated.flapAngles.frontHalf.length,
          {
            duration: 0.8,
            bottom: 0.49 * Math.PI,
            ease: "back.in(3)",
          },
          1.4
        )
        .to(
          [
            box.animated.flapAngles.backHalf.width,
            box.animated.flapAngles.frontHalf.width,
          ],
          {
            duration: 0.6,
            top: 0.6 * Math.PI,
            ease: "back.in(3)",
          },
          1.4
        )
        .to(
          box.animated.flapAngles.backHalf.length,
          {
            duration: 0.7,
            top: 0.5 * Math.PI,
            ease: "back.in(3)",
          },
          1.7
        )
        .to(
          box.animated.flapAngles.frontHalf.length,
          {
            duration: 0.9,
            top: 0.49 * Math.PI,
            ease: "back.in(4)",
          },
          1.8
        );
    }

    function updatePanelsTransform() {
      // place width-sides aside of length-sides (not animated)
      box.els.frontHalf.width.side.position.x = 0.5 * box.params.length;
      box.els.backHalf.width.side.position.x = -0.5 * box.params.length;

      copyright3.position.x = 0.5 * box.params.length;
      copyright4.position.x = -0.5 * box.params.length;

      // rotate width-sides from 0 to 90 deg
      box.els.frontHalf.width.side.rotation.y = box.animated.openingAngle;
      box.els.backHalf.width.side.rotation.y = box.animated.openingAngle;

      copyright3.position.y = box.animated.openingAngle;
      copyright4.position.y = box.animated.openingAngle;

      // move length-sides to keep the box centered
      const cos = Math.cos(box.animated.openingAngle); // animates from 1 to 0
      copyright3.position.x = -0.5 * cos * box.params.width;
      copyright4.position.x = 0.5 * cos * box.params.width;

      box.els.frontHalf.length.side.position.x = -0.5 * cos * box.params.width;
      box.els.backHalf.length.side.position.x = 0.5 * cos * box.params.width;

      // move length-sides to define box inner space
      const sin = Math.sin(box.animated.openingAngle); // animates from 0 to 1
      box.els.frontHalf.length.side.position.z = 0.5 * sin * box.params.width;
      box.els.backHalf.length.side.position.z = -0.5 * sin * box.params.width;

      copyright3.position.z = 0.5 * sin * box.params.width;
      copyright4.position.z = -0.5 * sin * box.params.width;

      box.els.frontHalf.width.top.rotation.x =
        -box.animated.flapAngles.frontHalf.width.top;
      box.els.frontHalf.length.top.rotation.x =
        -box.animated.flapAngles.frontHalf.length.top;
      box.els.frontHalf.width.bottom.rotation.x =
        box.animated.flapAngles.frontHalf.width.bottom;
      box.els.frontHalf.length.bottom.rotation.x =
        box.animated.flapAngles.frontHalf.length.bottom;

      box.els.backHalf.width.top.rotation.x =
        box.animated.flapAngles.backHalf.width.top;
      box.els.backHalf.length.top.rotation.x =
        box.animated.flapAngles.backHalf.length.top;
      box.els.backHalf.width.bottom.rotation.x =
        -box.animated.flapAngles.backHalf.width.bottom;
      box.els.backHalf.length.bottom.rotation.x =
        -box.animated.flapAngles.backHalf.length.bottom;

      copyright.position.copy(box.els.frontHalf.length.side.position);
      copyright.position.x +=
        0.5 * box.params.length - 0.5 * box.params.copyrightSize[0];
      copyright.position.y -=
        0.5 * (box.params.depth - box.params.copyrightSize[1]);
      copyright.position.z += box.params.thickness;
      console.log("boc", box.els.backHalf.length.side.position);

      copyright2.position.copy({
        x: 0.5 * box.params.length - 0.5 * box.params.copyrightSize[0],
        y: 0.5 * (box.params.depth - box.params.copyrightSize[1]),
        z: -0.5 * sin * box.params.width,
      });
      copyright2.rotation.x = 1 * Math.PI;
      copyright2.rotation.z = 1 * Math.PI;

      copyright2.position.x -=
        0.5 * box.params.length - 0.5 * box.params.copyrightSize[0];
      copyright2.position.y -=
        0.5 * (box.params.depth - box.params.copyrightSize[1]);
      copyright2.position.z -= box.params.thickness;

      // ------------------------

      // copyright3.position.copy({
      //   x: 0.5 * sin * box.params.length + box.params.copyrightSize[0],
      //   y: 0.5 * box.params.depth - 0.5 * box.params.copyrightSize[1],
      //   z: -0.5 * sin * box.params.width,
      // });
      // copyright3.rotation.x = 1 * Math.PI;
      // copyright3.rotation.z = 1 * Math.PI;
      // copyright3.rotation.y = 0.5 * Math.PI;

      // copyright3.position.x +=
      //   1 * cos * box.params.width - box.params.copyrightSize[0] + 1;
      // copyright3.position.y -=
      //   0.5 * (box.params.depth - box.params.copyrightSize[1]);
      // copyright3.position.z -= box.params.thickness - 12;

      // copyright3.position.copy({
      //   x: -0.5 * sin * box.params.width,
      //   y: 0.5 * (box.params.depth - box.params.copyrightSize[1]),
      //   z: 0.5 * box.params.length - 0.5 * box.params.copyrightSize[0],
      // });
      // copyright3.rotation.x = -0.5 * Math.PI;
      // copyright3.rotation.y = -0.5 * Math.PI;
      // copyright3.rotation.z = -0.5 * Math.PI;

      // copyright3.position.x -= box.params.thickness;
      // copyright3.position.y -=
      //   0.5 * (box.params.depth - box.params.copyrightSize[1]);
      // copyright3.position.z -=
      //   0.5 * box.params.length - 0.5 * box.params.copyrightSize[0];
    }

    // End of animation
    // --------------------------------------------------

    // --------------------------------------------------
    // Manual zoom (buttons only since the scroll is used
    // by folding animation)

    function createZooming() {
      const zoomInBtn = document.querySelector("#zoom-in");
      const zoomOutBtn = document.querySelector("#zoom-out");

      let zoomLevel = 1;
      const limits = [0.4, 2];

      zoomInBtn.addEventListener("click", () => {
        zoomLevel *= 1.3;
        applyZoomLimits();
      });
      zoomOutBtn.addEventListener("click", () => {
        zoomLevel *= 0.75;
        applyZoomLimits();
      });

      function applyZoomLimits() {
        if (zoomLevel > limits[1]) {
          zoomLevel = limits[1];
          zoomInBtn.classList.add("disabled");
        } else if (zoomLevel < limits[0]) {
          zoomLevel = limits[0];
          zoomOutBtn.classList.add("disabled");
        } else {
          zoomInBtn.classList.remove("disabled");
          zoomOutBtn.classList.remove("disabled");
        }
        gsap.to(camera, {
          duration: 0.2,
          zoom: zoomLevel,
          onUpdate: () => {
            camera.updateProjectionMatrix();
          },
        });
      }
    }

    // End of Manual zoom
    // --------------------------------------------------

    // --------------------------------------------------
    // Range sliders for box parameters

    function createControls() {
      const gui = new GUI();
      gui
        .add(
          box.params,
          "width",
          box.params.widthLimits[0],
          box.params.widthLimits[1]
        )
        .step(1)
        .onChange(() => {
          createBoxElements();
          updatePanelsTransform();
        });
      gui
        .add(
          box.params,
          "length",
          box.params.lengthLimits[0],
          box.params.lengthLimits[1]
        )
        .step(1)
        .onChange(() => {
          createBoxElements();
          updatePanelsTransform();
        });
      gui
        .add(
          box.params,
          "depth",
          box.params.depthLimits[0],
          box.params.depthLimits[1]
        )
        .step(1)
        .onChange(() => {
          createBoxElements();
          updatePanelsTransform();
        });
      // gui
      //   .add(
      //     box.params,
      //     "fluteFreq",
      //     box.params.fluteFreqLimits[0],
      //     box.params.fluteFreqLimits[1]
      //   )
      //   .step(1)
      //   .onChange(() => {
      //     createBoxElements();
      //   })
      //   .name("flute");
      // gui
      //   .add(
      //     box.params,
      //     "thickness",
      //     box.params.thicknessLimits[0],
      //     box.params.thicknessLimits[1]
      //   )
      //   .step(0.05)
      //   .onChange(() => {
      //     createBoxElements();
      //   });
    }
    function render() {
      // orbit.update();
      lightHolder.quaternion.copy(camera.quaternion);
      renderer.render(scene, camera);
      requestAnimationFrame(render);
    }

    function initScene() {
      const container = document.querySelector(".container");
      const boxCanvas = document.querySelector("#box-canvas");
      renderer = new THREE.WebGLRenderer({
        alpha: true,
        antialias: true,
        canvas: boxCanvas,
      });
      renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));

      scene = new THREE.Scene();
      camera = new THREE.PerspectiveCamera(
        80,
        container.clientWidth / container.clientHeight,
        10,
        1000
      );
      camera.position.set(40, 90, 110);

      rayCaster = new THREE.Raycaster();
      mouse = new THREE.Vector2(0, 0);

      updateSceneSize();

      scene.add(box.els.group);
      setGeometryHierarchy();

      const ambientLight = new THREE.AmbientLight(0xffffff, 1);
      scene.add(ambientLight);
      lightHolder = new THREE.Group();
      const topLight = new THREE.PointLight(0xffffff, 1);
      topLight.position.set(-30, 300, 0);
      lightHolder.add(topLight);
      const sideLight = new THREE.PointLight(0xffffff, 0.7);
      sideLight.position.set(50, 0, 150);
      lightHolder.add(sideLight);
      scene.add(lightHolder);

      scene.add(box.els.group);
      setGeometryHierarchy();
      const texture = new THREE.TextureLoader().load("/images/earth1.avif");
      texture.wrapS = THREE.RepeatWrapping;
      texture.wrapT = THREE.RepeatWrapping;
      texture.repeat.set(4, 4);

      const material = new THREE.MeshStandardMaterial({
        map: new THREE.TextureLoader().load("/images/earth1.avif"),
        color: new THREE.Color(0xffffff),
        // emissive: new THREE.Color(0x9c8d7b),
        side: THREE.DoubleSide,
        border: new THREE.Color("red"),
      });
      box.els.group.traverse((c) => {
        if (c.isMesh) c.material = material;
      });

      orbit = new OrbitControls(camera, boxCanvas);
      orbit.enableZoom = false;
      orbit.enablePan = false;
      orbit.enableDamping = true;
      orbit.autoRotate = true;
      orbit.autoRotateSpeed = 0.25;

      createCopyright();
      createBoxElements();
      createFoldingAnimation();
      createZooming();

      render();
    }
    onMounted(() => {
      // container.value = document.querySelector(".container");
      // boxCanvas.value = document.querySelector("#box-canvas");
      // console.log("boxCanvas", boxCanvas);
      initScene();
      createControls();
      window.addEventListener("resize", updateSceneSize);
    });
  },
};
</script>
