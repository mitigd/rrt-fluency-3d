<script lang="ts">
  import { onMount, onDestroy } from "svelte";
  import * as THREE from "three";
  import Chart from "chart.js/auto";

  // --- Types & Interfaces ---
  type Attribute = "color" | "shape" | "size" | "position";
  type ShapeType = "cube" | "sphere" | "tetrahedron";
  type ColorType = "red" | "blue" | "green";
  type SizeType = "small" | "medium" | "large";
  type PositionType = "left" | "center" | "right";

  type PerspectiveType = "symbolic" | "spatial";
  type SpatialRepType = "rotation" | "folding" | "cutout" | "instant";
  type ObserverPos = "me" | "opposite";
  type SpatialMatchMode = "view" | "object"; // NEW: Strategy Toggle

  type UserAction = "match" | "non_match";
  type PhysicalSide = "left" | "right";

  interface Stimulus {
    color: ColorType;
    backColor: ColorType;
    netColors: ColorType[];
    shape: ShapeType;
    size: SizeType;
    position: PositionType;
    id: number;
    stroopText: string;
    stroopInk: ColorType;
    observerPos: ObserverPos;
  }

  interface GameHistory {
    date: string;
    fullDate: string;
    score: number;
    accuracy: number;
    maxNBack: number;
    perspectiveMode: boolean;
    perspectiveType: PerspectiveType;
    activeRules: Attribute[];
  }

  // --- Game Constants ---
  const ALL_ATTRIBUTES: Attribute[] = ["color", "shape", "size", "position"];
  const COLORS: ColorType[] = ["red", "blue", "green"];
  const SHAPES: ShapeType[] = ["cube", "sphere", "tetrahedron"];
  const SIZES: SizeType[] = ["small", "medium", "large"];
  const POSITIONS: PositionType[] = ["left", "center", "right"];
  const STROOP_WORDS = ["RED", "BLUE", "GREEN"];

  const COLOR_MAP: Record<ColorType, number> = {
    red: 0xff0000,
    blue: 0x007bff,
    green: 0x28a745,
  };

  const CSS_COLOR_MAP: Record<ColorType, string> = {
    red: "#dc3545",
    blue: "#007bff",
    green: "#28a745",
  };

  const SIZE_MAP: Record<SizeType, number> = {
    small: 0.8,
    medium: 1.5,
    large: 2.2,
  };

  const POS_MAP: Record<PositionType, number> = {
    left: -2.5,
    center: 0,
    right: 2.5,
  };

  const INTERVAL_MS = 3500;
  const BLOCK_SIZE = 20;

  // --- State Variables ---
  let isPlaying = false;
  let score = 0;

  // Configuration State
  let activeRules: Attribute[] = ["color"];
  let perspectiveMode = false;
  let perspectiveType: PerspectiveType = "symbolic";
  let spatialType: SpatialRepType = "rotation";
  let spatialMatchMode: SpatialMatchMode = "view"; // Default to Empathy/View
  let showStroop = true;
  let distractorsEnabled = false;
  let controlSwapEnabled = false;

  // Timer State
  let useTimer = true;
  let gameDurationSeconds = 60;
  let timeLeftSeconds = gameDurationSeconds;
  let gameTimerInterval: number | undefined;

  let configMinutes = 1;
  let configSeconds = 0;

  // Progression State
  let nBack = 1;
  let autoProgress = true;
  let blockTurnCount = 0;
  let blockMistakes = 0;
  let totalMistakes = 0;
  let totalTurns = 0;
  let lastStroopText = "";
  let hasResponded = false;
  let isSwapped = false;

  // Perspective Logic
  let perspectiveMapping: Record<string, Record<string, string>> = {};

  // Game State
  let currentRule: Attribute = "color";
  let stimulusHistory: Stimulus[] = [];
  let currentStimulus: Stimulus | null = null;

  let timer: number | undefined;
  let animationFrameId: number;
  let spinStartTime = 0;
  let isSpinning = false;

  // UI State
  let feedbackMsg = "";
  let feedbackColor = "";
  let levelUpMsg = "";
  let showHistory = false;
  let showSettings = false;

  // Chart & Three.js refs
  let canvasChart: HTMLCanvasElement;
  let chartInstance: Chart | null = null;
  let container: HTMLDivElement;
  let scene: THREE.Scene;
  let camera: THREE.PerspectiveCamera;
  let renderer: THREE.WebGLRenderer;
  let currentMesh: THREE.Object3D | null = null;
  let holeTexture: THREE.CanvasTexture | null = null;

  // --- Lifecycle ---

  onMount(() => {
    loadSettings();
    initThree();
    window.addEventListener("resize", onResize);
    window.addEventListener("keydown", handleKeydown);
  });

  onDestroy(() => {
    stopGame();
    if (renderer) {
      renderer.dispose();
      renderer.forceContextLoss();
    }
    if (chartInstance) chartInstance.destroy();
    window.removeEventListener("resize", onResize);
    window.removeEventListener("keydown", handleKeydown);
    if (animationFrameId) cancelAnimationFrame(animationFrameId);
    if (currentMesh) cleanMesh(currentMesh);
    if (holeTexture) holeTexture.dispose();
  });

  function cleanMesh(obj: THREE.Object3D) {
    if (obj.children.length > 0) {
      obj.children.forEach((child) => cleanMesh(child));
    }
    if (obj instanceof THREE.Mesh) {
      if (obj.geometry) obj.geometry.dispose();
      if (Array.isArray(obj.material)) {
        obj.material.forEach((m: any) => m.dispose());
      } else if (obj.material) {
        obj.material.dispose();
      }
    }
  }

  // --- Three.js Setup ---

  function initThree() {
    scene = new THREE.Scene();
    camera = new THREE.PerspectiveCamera(
      75,
      window.innerWidth / window.innerHeight,
      0.1,
      1000,
    );
    camera.position.z = 6;

    renderer = new THREE.WebGLRenderer({ alpha: true, antialias: true });
    renderer.setSize(window.innerWidth, window.innerHeight);
    renderer.setPixelRatio(window.devicePixelRatio);
    if (container) container.appendChild(renderer.domElement);

    const ambientLight = new THREE.AmbientLight(0xffffff, 0.6);
    scene.add(ambientLight);
    const dirLight = new THREE.DirectionalLight(0xffffff, 0.8);
    dirLight.position.set(5, 5, 5);
    scene.add(dirLight);

    animate();
  }

  function animate() {
    animationFrameId = requestAnimationFrame(animate);

    if (currentMesh) {
      if (!isSpinning) {
        currentMesh.rotation.x = THREE.MathUtils.lerp(
          currentMesh.rotation.x,
          0.2,
          0.05,
        );
        currentMesh.rotation.y = THREE.MathUtils.lerp(
          currentMesh.rotation.y,
          0,
          0.05,
        );
      } else {
        const elapsed = Date.now() - spinStartTime;

        if (spatialType === "instant") {
          const delay = 500;
          const showTime = 600;
          if (elapsed < delay) {
            currentMesh.rotation.y = 0;
          } else if (elapsed < delay + showTime) {
            currentMesh.rotation.y = Math.PI;
          } else {
            currentMesh.rotation.y = 0;
            isSpinning = false;
          }
        } else {
          const duration = 1500;
          if (elapsed < duration) {
            const progress = elapsed / duration;
            const angle = Math.sin(progress * Math.PI) * Math.PI;
            currentMesh.rotation.y = angle;
          } else {
            isSpinning = false;
            currentMesh.rotation.y = 0;
          }
        }
      }
    }
    if (renderer && scene && camera) renderer.render(scene, camera);
  }

  function onResize() {
    if (!camera || !renderer) return;
    camera.aspect = window.innerWidth / window.innerHeight;
    camera.updateProjectionMatrix();
    renderer.setSize(window.innerWidth, window.innerHeight);
  }

  function getHoleTexture() {
    if (holeTexture) return holeTexture;
    const canvas = document.createElement("canvas");
    canvas.width = 256;
    canvas.height = 256;
    const ctx = canvas.getContext("2d");
    if (ctx) {
      ctx.fillStyle = "#FFFFFF";
      ctx.fillRect(0, 0, 256, 256);
      ctx.globalCompositeOperation = "destination-out";
      ctx.beginPath();
      ctx.arc(128, 128, 90, 0, Math.PI * 2);
      ctx.fill();
    }
    holeTexture = new THREE.CanvasTexture(canvas);
    return holeTexture;
  }

  // --- Perspective & Logic ---

  function generateCyclicMapping(values: string[]): Record<string, string> {
    let shuffled = [...values];
    let isValid = false;
    while (!isValid) {
      shuffled = [...values].sort(() => Math.random() - 0.5);
      isValid = true;
      for (let i = 0; i < values.length; i++) {
        if (values[i] === shuffled[i]) {
          isValid = false;
          break;
        }
      }
    }
    const map: Record<string, string> = {};
    for (let i = 0; i < values.length; i++) {
      map[values[i]] = shuffled[i];
    }
    return map;
  }

  function scramblePerspectiveRules() {
    perspectiveMapping = {
      color: generateCyclicMapping(COLORS),
      shape: generateCyclicMapping(SHAPES),
      size: generateCyclicMapping(SIZES),
      position: generateCyclicMapping(POSITIONS),
    };
  }

  // --- CORE MATCHING LOGIC ---
  function getTransformedValue(stim: Stimulus, rule: Attribute): any {
    // 1. No Perspective Mode
    if (!perspectiveMode) return stim[rule];

    // 2. Symbolic Mode
    if (perspectiveType === "symbolic") {
      if (stim.observerPos === "me") return stim[rule];
      const rawValue = stim[rule];
      // @ts-ignore
      return perspectiveMapping[rule]
        ? perspectiveMapping[rule][rawValue]
        : rawValue;
    }

    // 3. Spatial Mode
    if (perspectiveType === "spatial") {
      // 3a. Position (Mirroring)
      if (rule === "position") {
        if (spatialMatchMode === "object") {
          // In Object mode, position is less relevant to identity, 
          // but we return the raw position for consistency.
          return stim.position;
        }
        // Empathy Mode:
        if (stim.observerPos === "me") return stim.position;
        if (stim.position === "left") return "right";
        if (stim.position === "right") return "left";
        return "center";
      }

      // 3b. Color Logic
      if (rule === "color") {
        if (spatialMatchMode === "object") {
          // OBJECT IDENTITY MODE:
          // Return a sorted signature of the pair. 
          // Red/Green is the same object as Green/Red.
          const colors = [stim.color, stim.backColor].sort();
          return colors.join("-");
        } else {
          // VIEW / EMPATHY MODE:
          // Return what the specific observer sees.
          if (stim.observerPos === "me") return stim.color; 
          return stim.backColor; 
        }
      }

      // 3c. Shape/Size (Invariant)
      return stim[rule];
    }

    return stim[rule];
  }

  function generateStimulus(): Stimulus {
    const color = COLORS[Math.floor(Math.random() * COLORS.length)];
    let backColor = COLORS[Math.floor(Math.random() * COLORS.length)];
    while (backColor === color)
      backColor = COLORS[Math.floor(Math.random() * COLORS.length)];

    const netColors = Array(6)
      .fill("")
      .map(() => COLORS[Math.floor(Math.random() * COLORS.length)]);
    netColors[2] = color;
    netColors[5] = backColor;

    const varyShape = activeRules.includes("shape") || distractorsEnabled;
    let shape = varyShape
      ? SHAPES[Math.floor(Math.random() * SHAPES.length)]
      : "cube";

    if (perspectiveMode && perspectiveType === "spatial") {
      if (spatialType === "cutout") {
        shape = Math.random() > 0.5 ? "cube" : "sphere";
      } else {
        shape = "cube";
      }
    }

    const varySize = activeRules.includes("size") || distractorsEnabled;
    const size = varySize
      ? SIZES[Math.floor(Math.random() * SIZES.length)]
      : "medium";

    const varyPos = activeRules.includes("position") || distractorsEnabled;
    const position = varyPos
      ? POSITIONS[Math.floor(Math.random() * POSITIONS.length)]
      : "center";

    const observerPos: ObserverPos =
      Math.random() > 0.5 ? "me" : "opposite";

    let stroopText =
      STROOP_WORDS[Math.floor(Math.random() * STROOP_WORDS.length)];
    if (stroopText === lastStroopText)
      stroopText =
        STROOP_WORDS[Math.floor(Math.random() * STROOP_WORDS.length)];
    lastStroopText = stroopText;
    const stroopInk = COLORS[Math.floor(Math.random() * COLORS.length)];

    return {
      color,
      backColor,
      netColors,
      shape,
      size,
      position,
      id: Date.now(),
      stroopText,
      stroopInk,
      observerPos,
    };
  }

  function updateMesh(stim: Stimulus) {
    if (currentMesh) {
      scene.remove(currentMesh);
      cleanMesh(currentMesh);
    }
    const matSettings = { roughness: 0.4, metalness: 0.1 };
    const scale = SIZE_MAP[stim.size];
    const useDualColor =
      perspectiveMode &&
      perspectiveType === "spatial" &&
      activeRules.includes("color");

    if (useDualColor) {
      const matFront = new THREE.MeshStandardMaterial({
        ...matSettings,
        color: COLOR_MAP[stim.color],
      });
      const matBack = new THREE.MeshStandardMaterial({
        ...matSettings,
        color: COLOR_MAP[stim.backColor],
      });

      if (spatialType === "cutout") {
        const matInnerBright = new THREE.MeshBasicMaterial({
          color: COLOR_MAP[stim.backColor],
          side: THREE.BackSide,
        });
        const matEdge = new THREE.MeshBasicMaterial({ color: 0x000000 });

        if (stim.shape === "cube") {
          const matHole = new THREE.MeshStandardMaterial({
            ...matSettings,
            color: COLOR_MAP[stim.color],
            alphaMap: getHoleTexture(),
            transparent: true,
            side: THREE.FrontSide,
          });
          const outerGeo = new THREE.BoxGeometry();
          const matSide = new THREE.MeshStandardMaterial({
            ...matSettings,
            color: COLOR_MAP[stim.color],
          });
          const matsOuter = [
            matSide,
            matSide,
            matSide,
            matSide,
            matHole,
            matSide,
          ];
          const outerMesh = new THREE.Mesh(outerGeo, matsOuter);
          const innerGeo = new THREE.BoxGeometry(0.95, 0.95, 0.95);
          const innerMesh = new THREE.Mesh(innerGeo, matInnerBright);
          const edgeGeo = new THREE.TorusGeometry(0.35, 0.02, 16, 48);
          const edgeMesh = new THREE.Mesh(edgeGeo, matEdge);
          edgeMesh.position.z = 0.5;

          const group = new THREE.Group();
          group.add(outerMesh, innerMesh, edgeMesh);
          currentMesh = group;
        } else {
          const R = 0.7;
          const cutAngle = 0.6;
          const outerGeo = new THREE.SphereGeometry(
            R,
            32,
            32,
            0,
            Math.PI * 2,
            cutAngle,
            Math.PI - cutAngle,
          );
          const outerMesh = new THREE.Mesh(outerGeo, matFront);
          outerMesh.rotation.x = Math.PI / 2;
          const innerGeo = new THREE.SphereGeometry(R - 0.02, 32, 32);
          const innerMesh = new THREE.Mesh(innerGeo, matInnerBright);
          const ringRadius = R * Math.sin(cutAngle);
          const ringY = R * Math.cos(cutAngle);
          const edgeGeo = new THREE.TorusGeometry(ringRadius, 0.02, 16, 48);
          const edgeMesh = new THREE.Mesh(edgeGeo, matEdge);
          edgeMesh.position.z = ringY;

          const group = new THREE.Group();
          group.add(outerMesh, innerMesh, edgeMesh);
          currentMesh = group;
        }
        isSpinning = false;
      } else if (spatialType === "folding") {
        const c = stim.netColors;
        const materials = [
          new THREE.MeshStandardMaterial({
            ...matSettings,
            color: COLOR_MAP[c[3]],
          }),
          new THREE.MeshStandardMaterial({
            ...matSettings,
            color: COLOR_MAP[c[1]],
          }),
          new THREE.MeshStandardMaterial({
            ...matSettings,
            color: COLOR_MAP[c[0]],
          }),
          new THREE.MeshStandardMaterial({
            ...matSettings,
            color: COLOR_MAP[c[4]],
          }),
          new THREE.MeshStandardMaterial({
            ...matSettings,
            color: COLOR_MAP[c[2]],
          }),
          new THREE.MeshStandardMaterial({
            ...matSettings,
            color: COLOR_MAP[c[5]],
          }),
        ];
        currentMesh = new THREE.Mesh(new THREE.BoxGeometry(), materials);
        isSpinning = false;
      } else {
        const matSide = new THREE.MeshStandardMaterial({
          ...matSettings,
          color: 0x888888,
        });
        const materials = [
          matSide,
          matSide,
          matSide,
          matSide,
          matFront,
          matBack,
        ];
        currentMesh = new THREE.Mesh(new THREE.BoxGeometry(), materials);
        isSpinning = true;
        spinStartTime = Date.now();
      }
    } else {
      let geometry: THREE.BufferGeometry;
      if (stim.shape === "cube") geometry = new THREE.BoxGeometry();
      else if (stim.shape === "sphere")
        geometry = new THREE.SphereGeometry(0.7, 32, 32);
      else {
        geometry = new THREE.TetrahedronGeometry(1.2);
        geometry.rotateX(-Math.atan(Math.sqrt(2)));
        geometry.rotateY(Math.PI / 4);
        geometry.rotateZ(Math.PI / 4);
      }
      const material = new THREE.MeshStandardMaterial({
        ...matSettings,
        color: COLOR_MAP[stim.color],
      });
      currentMesh = new THREE.Mesh(geometry, material);
      isSpinning = false;
    }

    if (currentMesh) {
      currentMesh.scale.set(scale, scale, scale);
      currentMesh.position.x = POS_MAP[stim.position];
      scene.add(currentMesh);
    }
  }

  // --- Game Mechanics ---

  function checkProgression() {
    if (!autoProgress || blockTurnCount < BLOCK_SIZE || useTimer) return;
    const accuracy = ((blockTurnCount - blockMistakes) / blockTurnCount) * 100;
    if (accuracy >= 80) {
      nBack++;
      levelUpMsg = `Awesome! Level Up: N=${nBack}`;
      flashFeedback("LEVEL UP!", "text-accent");
    } else if (accuracy < 50 && nBack > 1) {
      nBack--;
      levelUpMsg = `Too fast? Level Down: N=${nBack}`;
      flashFeedback("Level Down", "text-red-500");
    } else {
      levelUpMsg = `Maintaining Level: N=${nBack} (${Math.round(accuracy)}%)`;
    }
    setTimeout(() => (levelUpMsg = ""), 3000);
    blockTurnCount = 0;
    blockMistakes = 0;
  }

  function checkMissedTarget() {
    if (stimulusHistory.length > nBack && !hasResponded) {
      score -= 5;
      totalMistakes++;
      blockMistakes++;
      flashFeedback("Too Slow!", "text-red-500");
    }
  }

  function nextTurn() {
    if (!isPlaying) return;
    checkMissedTarget();
    checkProgression();

    if (controlSwapEnabled) isSwapped = Math.random() < 0.35;
    else isSwapped = false;

    const randomIndex = Math.floor(Math.random() * activeRules.length);
    currentRule = activeRules[randomIndex];

    const stim = generateStimulus();
    stimulusHistory.push(stim);
    currentStimulus = stim;
    updateMesh(stim);

    if (useTimer && !gameTimerInterval) {
      if (stimulusHistory.length > nBack) {
        startTimer();
      }
    }

    totalTurns++;
    blockTurnCount++;
    hasResponded = false;

    if (timer) clearTimeout(timer);
    timer = setTimeout(nextTurn, INTERVAL_MS);
  }

  function startTimer() {
    if (gameTimerInterval) clearInterval(gameTimerInterval);
    timeLeftSeconds = gameDurationSeconds;
    gameTimerInterval = setInterval(() => {
      timeLeftSeconds -= 1;
      if (timeLeftSeconds <= 0) endGame(true);
    }, 1000);
  }

  function updateDurationFromInputs() {
    if (configMinutes < 0) configMinutes = 0;
    if (configSeconds < 0) configSeconds = 0;
    if (configSeconds > 59) configSeconds = 59;
    let total = configMinutes * 60 + configSeconds;
    if (total < 10) {
      total = 10;
      configSeconds = 10;
      configMinutes = 0;
    }
    gameDurationSeconds = total;
    timeLeftSeconds = total;
    updateSettings();
  }

  function endGame(timedOut: boolean) {
    isPlaying = false;
    clearTimeout(timer);
    if (gameTimerInterval) clearInterval(gameTimerInterval);
    gameTimerInterval = undefined;
    const accuracy =
      totalTurns > 0 ? ((totalTurns - totalMistakes) / totalTurns) * 100 : 0;
    const isSuccess = accuracy >= 80;

    if (timedOut && autoProgress && totalTurns > 0) {
      if (isSuccess) {
        nBack++;
        levelUpMsg = `TIME UP! SUCCESS! Level Up to ${nBack} (${Math.round(accuracy)}%)`;
        flashFeedback("LEVEL UP!", "text-green-500");
      } else if (nBack > 1 && accuracy < 50) {
        nBack--;
        levelUpMsg = `TIME UP! Level Down to ${nBack} (${Math.round(accuracy)}%)`;
        flashFeedback("Level Down", "text-red-500");
      } else {
        levelUpMsg = `TIME UP! Maintaining N=${nBack} (${Math.round(accuracy)}%)`;
        flashFeedback("Time Expired", "text-accent");
      }
    } else {
      levelUpMsg = `GAME ENDED. Accuracy: ${Math.round(accuracy)}%`;
      flashFeedback("Game Ended", "text-accent");
    }
    setTimeout(() => (levelUpMsg = ""), 5000);
    if (timedOut) saveHistory();
  }

  function startGame() {
    if (perspectiveMode && perspectiveType === "spatial") {
      activeRules = activeRules.filter((r) => r !== "shape" && r !== "size");
      if (activeRules.length === 0) activeRules = ["color"];
    }
    if (activeRules.length === 0) {
      flashFeedback("Select Rules First!", "text-red-500");
      return;
    }
    if (perspectiveMode) scramblePerspectiveRules();

    isPlaying = true;
    score = 0;
    totalMistakes = 0;
    blockMistakes = 0;
    totalTurns = 0;
    blockTurnCount = 0;
    stimulusHistory = [];
    hasResponded = false;

    if (gameTimerInterval) clearInterval(gameTimerInterval);
    gameTimerInterval = undefined;
    timeLeftSeconds = gameDurationSeconds;
    nextTurn();
  }

  function stopGame() {
    isPlaying = false;
    clearTimeout(timer);
    if (gameTimerInterval) clearInterval(gameTimerInterval);
    gameTimerInterval = undefined;
    levelUpMsg = "";
    flashFeedback("Stopped", "text-accent");
  }

  function handlePhysicalSide(side: PhysicalSide) {
    if (!isPlaying) return;
    let action: UserAction;
    if (!isSwapped) action = side === "left" ? "match" : "non_match";
    else action = side === "left" ? "non_match" : "match";
    handleInput(action);
  }

  function handleInput(action: UserAction) {
    if (!isPlaying || stimulusHistory.length <= nBack || hasResponded) return;
    hasResponded = true;

    const currentStim = stimulusHistory[stimulusHistory.length - 1];
    const targetStim = stimulusHistory[stimulusHistory.length - 1 - nBack];

    const val1 = getTransformedValue(currentStim, currentRule);
    const val2 = getTransformedValue(targetStim, currentRule);

    const isMatch = val1 === val2;
    let correct = false;
    let feedback = "";

    if (action === "match") {
      if (isMatch) {
        correct = true;
        feedback = "Correct!";
      } else {
        correct = false;
        feedback = "False Alarm!";
      }
    } else {
      if (!isMatch) {
        correct = true;
        feedback = "Correct Rejection!";
      } else {
        correct = false;
        feedback = "Miss!";
      }
    }

    if (correct) {
      const reactionTime = Date.now() - currentStimulus.id;
      const speedBonus = Math.max(
        0,
        Math.min(10, Math.floor((2000 - reactionTime) / 150)),
      );
      score += 10 + nBack * 5 + speedBonus;
      if (speedBonus > 5)
        flashFeedback(`FAST! +${speedBonus}`, "text-green-500");
      else flashFeedback("Correct!", "text-green-500");
    } else {
      totalMistakes++;
      blockMistakes++;
      score -= 5;
      flashFeedback(
        feedback === "Miss!" ? "Missed!" : "Incorrect!",
        "text-red-500",
      );
    }
    if (timer) clearTimeout(timer);
    timer = setTimeout(nextTurn, 500);
  }

  function handleKeydown(e: KeyboardEvent) {
    if (!isPlaying) return;
    if (e.code === "ArrowLeft" || e.code === "KeyD") {
      e.preventDefault();
      handlePhysicalSide("left");
    }
    if (e.code === "ArrowRight" || e.code === "KeyJ") {
      e.preventDefault();
      handlePhysicalSide("right");
    }
  }

  function onRightClick(e: MouseEvent) {
    e.preventDefault();
    handlePhysicalSide("right");
  }

  function manualLevelChange(delta: number) {
    const newLevel = nBack + delta;
    if (newLevel >= 1) {
      nBack = newLevel;
      blockTurnCount = 0;
      blockMistakes = 0;
    }
  }

  function flashFeedback(msg: string, colorClass: string) {
    feedbackMsg = msg;
    feedbackColor = colorClass;
    setTimeout(() => (feedbackMsg = ""), 800);
  }

  function loadSettings() {
    const savedRules = localStorage.getItem("rft_fluency_3d_rules");
    const savedMode = localStorage.getItem("rft_fluency_3d_perspective");
    const savedType = localStorage.getItem("rft_fluency_3d_persp_type");
    const savedSpatialType = localStorage.getItem("rft_fluency_3d_spatial_type");
    const savedMatchMode = localStorage.getItem("rft_fluency_3d_match_mode"); 
    const savedStroop = localStorage.getItem("rft_fluency_3d_stroop");
    const savedDistractors = localStorage.getItem("rft_fluency_3d_distractors");
    const savedSwap = localStorage.getItem("rft_fluency_3d_control_swap");
    const savedUseTimer = localStorage.getItem("rft_fluency_3d_use_timer");
    const savedDuration = localStorage.getItem("rft_fluency_3d_duration");

    if (savedMode) perspectiveMode = savedMode === "true";
    if (savedType) perspectiveType = savedType as PerspectiveType;
    if (savedSpatialType) spatialType = savedSpatialType as SpatialRepType;
    if (savedMatchMode) spatialMatchMode = savedMatchMode as SpatialMatchMode; 

    if (savedRules) {
      try {
        const parsed = JSON.parse(savedRules);
        if (Array.isArray(parsed) && parsed.length > 0) activeRules = parsed;
      } catch (e) {}
    }

    if (perspectiveMode && perspectiveType === "spatial") {
      activeRules = activeRules.filter((r) => r !== "shape" && r !== "size");
      if (activeRules.length === 0) activeRules = ["color"];
    }

    if (savedStroop !== null) showStroop = savedStroop === "true";
    if (savedDistractors !== null)
      distractorsEnabled = savedDistractors === "true";
    if (savedSwap !== null) controlSwapEnabled = savedSwap === "true";
    if (savedUseTimer !== null) useTimer = savedUseTimer === "true";
    if (savedDuration) {
      gameDurationSeconds = parseInt(savedDuration, 10);
      if (isNaN(gameDurationSeconds)) gameDurationSeconds = 60;
    }
    configMinutes = Math.floor(gameDurationSeconds / 60);
    configSeconds = gameDurationSeconds % 60;
    timeLeftSeconds = gameDurationSeconds;
  }

  function toggleRule(rule: Attribute) {
    if (perspectiveMode && perspectiveType === "spatial") {
      if (rule === "shape" || rule === "size") return;
    }
    if (activeRules.includes(rule)) {
      if (activeRules.length > 1)
        activeRules = activeRules.filter((r) => r !== rule);
    } else {
      activeRules = [...activeRules, rule];
    }
    localStorage.setItem("rft_fluency_3d_rules", JSON.stringify(activeRules));
  }

  function updateSettings() {
    localStorage.setItem("rft_fluency_3d_perspective", String(perspectiveMode));
    localStorage.setItem("rft_fluency_3d_persp_type", perspectiveType);
    localStorage.setItem("rft_fluency_3d_spatial_type", spatialType);
    localStorage.setItem("rft_fluency_3d_match_mode", spatialMatchMode); 
    localStorage.setItem("rft_fluency_3d_stroop", String(showStroop));
    localStorage.setItem(
      "rft_fluency_3d_distractors",
      String(distractorsEnabled),
    );
    localStorage.setItem(
      "rft_fluency_3d_control_swap",
      String(controlSwapEnabled),
    );
    localStorage.setItem("rft_fluency_3d_use_timer", String(useTimer));
    localStorage.setItem(
      "rft_fluency_3d_duration",
      String(gameDurationSeconds),
    );
  }

  function saveHistory() {
    if (totalTurns === 0) return;
    const accuracy = Math.round(
      ((totalTurns - totalMistakes) / totalTurns) * 100,
    );
    const now = new Date();
    const shortDate = `${now.getMonth() + 1}/${now.getDate()} ${now.getHours()}:${now.getMinutes().toString().padStart(2, "0")}`;
    const entry: GameHistory = {
      date: shortDate,
      fullDate: now.toLocaleString(),
      score,
      accuracy,
      maxNBack: nBack,
      perspectiveMode,
      perspectiveType,
      activeRules: [...activeRules],
    };
    const existing = JSON.parse(
      localStorage.getItem("rft_fluency_3d_history") || "[]",
    );
    existing.push(entry);
    if (existing.length > 20) existing.shift();
    localStorage.setItem("rft_fluency_3d_history", JSON.stringify(existing));
  }

  function toggleHistory() {
    showHistory = !showHistory;
    if (showHistory) setTimeout(renderChart, 50);
    else if (chartInstance) {
      chartInstance.destroy();
      chartInstance = null;
    }
  }

  function toggleSettings() {
    showSettings = !showSettings;
  }

  function renderChart() {
    if (!canvasChart) return;
    const history: GameHistory[] = JSON.parse(
      localStorage.getItem("rft_fluency_3d_history") || "[]",
    );
    if (chartInstance) chartInstance.destroy();
    chartInstance = new Chart(canvasChart, {
      type: "line",
      data: {
        labels: history.map((h) => h.date),
        datasets: [
          {
            label: "Score",
            data: history.map((h) => h.score),
            borderColor: "rgb(75, 192, 192)",
            tension: 0.1,
            yAxisID: "y",
          },
          {
            label: "Max N-Back",
            data: history.map((h) => h.maxNBack),
            borderColor: "rgb(54, 162, 235)",
            borderDash: [5, 5],
            tension: 0.1,
            yAxisID: "y1",
          },
        ],
      },
      options: {
        responsive: true,
        interaction: { mode: "index", intersect: false },
        scales: {
          y: {
            type: "linear",
            display: true,
            position: "left",
            title: { display: true, text: "Score" },
          },
          y1: {
            type: "linear",
            display: true,
            position: "right",
            min: 0,
            max: 10,
            grid: { drawOnChartArea: false },
            title: { display: true, text: "N-Back Level" },
          },
        },
      },
    });
  }
</script>

<main>
  <div bind:this={container} class="three-container"></div>

  {#if isPlaying && currentStimulus && showStroop}
    <div
      class="stroop-overlay"
      style="color: {CSS_COLOR_MAP[currentStimulus.stroopInk]}"
    >
      {currentStimulus.stroopText}
    </div>
  {/if}

  <div class="ui-layer">
    <div class="hud surface">
      <div class="stat-group">
        <div class="stat">
          <span class="label">Rule</span>
          <span class="value active-rule">{currentRule.toUpperCase()}</span>
        </div>
        <div class="stat">
          <span class="label">Score</span>
          <span class="value">{score}</span>
        </div>
      </div>

      {#if useTimer}
        <div class="stat">
          <span class="label">Time Left</span>
          <span class="value timer-value">
            {Math.floor(timeLeftSeconds / 60)}:{timeLeftSeconds % 60 < 10
              ? "0"
              : ""}{timeLeftSeconds % 60}
          </span>
        </div>
      {/if}

      {#if isPlaying && perspectiveMode && currentStimulus}
        {#if perspectiveType === "symbolic"}
            <div class="clue-box">
                <div class="clue-title">SYMBOLIC KEY</div>
                 {#if currentStimulus.observerPos === "me"}
                   <div class="clue-row"><span class="you-val">{currentStimulus[currentRule].toUpperCase()}</span></div>
                   <div class="clue-sub">(Match What You See)</div>
                 {:else}
                   <div class="clue-row"><span class="you-val">DECODE</span></div>
                   <div class="clue-sub">(Use Key)</div>
                 {/if}
            </div>
        {:else if perspectiveType === "spatial"}
          {#if spatialMatchMode === 'object'}
            <div class="clue-box spatial" style="border-color: #e91e63;">
                 <div class="clue-title">OBJECT CONSTANCY</div>
                 <div class="clue-row">
                    <span class="you-val" style="color: #e91e63;">PAIR</span>
                 </div>
                 <div class="clue-sub">(Match The Cube Pair)</div>
             </div>
          {:else}
             <div class="clue-box spatial">
                <div class="clue-title">SEAT POSITION</div>
                <div class="clue-row">
                    <span class="you-val">{currentStimulus.observerPos.toUpperCase()}</span>
                </div>
                 <div class="clue-sub">
                     {#if currentStimulus.observerPos === "opposite"}
                         (Match Their View)
                     {:else}
                         (Match Your View)
                     {/if}
                 </div>
              </div>
          {/if}
        {/if}
      {/if}

      <div class="controls-group">
        <div class="level-control">
          <span class="label">N-Back: {nBack}</span>
          <div class="btn-row">
            <button
              class="btn-mini"
              disabled={autoProgress}
              on:click={() => manualLevelChange(-1)}>-</button
            >
            <button
              class="btn-mini"
              disabled={autoProgress}
              on:click={() => manualLevelChange(1)}>+</button
            >
          </div>
        </div>
        <div class="toggle-control">
          <label class="switch-label">
            <input type="checkbox" bind:checked={autoProgress} />
            <span class="label-text">Auto-Level</span>
          </label>
        </div>
        {#if !isPlaying}
          <button class="btn-primary" on:click={startGame}>Start</button>
          <button class="btn-secondary" on:click={toggleSettings}>Config</button
          >
          <button class="btn-secondary" on:click={toggleHistory}>History</button
          >
        {:else}
          <button class="btn-danger" on:click={stopGame}>Stop</button>
        {/if}
      </div>
    </div>

    {#if isPlaying && perspectiveMode && currentStimulus && perspectiveType === "spatial" && spatialType === "folding"}
      <div class="net-key-container">
        <div class="clue-title">MENTAL FOLD KEY</div>
        <div class="net-key">
          <div class="net-row">
            <div class="net-cell empty"></div>
            <div
              class="net-cell"
              style="background-color: {CSS_COLOR_MAP[
                currentStimulus.netColors[0]
              ]}"
            ></div>
            <div class="net-cell empty"></div>
          </div>
          <div class="net-row">
            <div
              class="net-cell"
              style="background-color: {CSS_COLOR_MAP[
                currentStimulus.netColors[1]
              ]}"
            ></div>
            <div
              class="net-cell front-face"
              style="background-color: {CSS_COLOR_MAP[
                currentStimulus.netColors[2]
              ]}"
            >
              F
            </div>
            <div
              class="net-cell"
              style="background-color: {CSS_COLOR_MAP[
                currentStimulus.netColors[3]
              ]}"
            ></div>
          </div>
          <div class="net-row">
            <div class="net-cell empty"></div>
            <div
              class="net-cell"
              style="background-color: {CSS_COLOR_MAP[
                currentStimulus.netColors[4]
              ]}"
            ></div>
            <div class="net-cell empty"></div>
          </div>
          <div class="net-row">
            <div class="net-cell empty"></div>
            <div
              class="net-cell"
              style="background-color: {CSS_COLOR_MAP[
                currentStimulus.netColors[5]
              ]}"
            ></div>
            <div class="net-cell empty"></div>
          </div>
        </div>
        {#if currentStimulus.observerPos === "opposite"}
          <div class="clue-sub warning">FIND THE BACK FACE!</div>
        {:else}
          <div class="clue-sub">USE FRONT FACE</div>
        {/if}
      </div>
    {/if}

    {#if isPlaying && perspectiveMode && currentStimulus}
      <div class="avatar-layer">
        <svg width="0" height="0" style="position: absolute;">
          <defs>
            <linearGradient id="avatarGrad" x1="0%" y1="0%" x2="100%" y2="0%">
              <stop offset="0%" style="stop-color:#343a40;stop-opacity:1" />
              <stop offset="100%" style="stop-color:#212529;stop-opacity:1" />
            </linearGradient>
          </defs>
        </svg>
        {#if currentStimulus.observerPos === "opposite"}
          <div class="avatar-top">
            <svg viewBox="0 0 100 100" class="avatar-svg">
              <path d="M 10 0 Q 50 80 90 0" fill="url(#avatarGrad)" />
              <circle cx="50" cy="60" r="30" fill="#212529" />
            </svg>
            <div class="avatar-label">THEM</div>
          </div>
        {:else}
          <div class="avatar-bottom">
            <svg viewBox="0 0 100 100" class="avatar-svg">
              <path d="M 10 100 Q 50 20 90 100" fill="url(#avatarGrad)" />
              <circle cx="50" cy="40" r="30" fill="#212529" />
            </svg>
            <div class="avatar-label">ME</div>
          </div>
        {/if}
      </div>
    {/if}

    {#if levelUpMsg}
      <div class="toast-notification surface">{levelUpMsg}</div>
    {/if}
    {#if feedbackMsg}
      <div class="feedback {feedbackColor}">{feedbackMsg}</div>
    {/if}

    {#if isPlaying}
      <div
        class="tap-zone left"
        on:click={() => handlePhysicalSide("left")}
        role="button"
        tabindex="0"
      >
        <div class="indicator-spot {isSwapped ? 'red-spot' : 'green-spot'}">
          {#if isSwapped}NO{:else}YES{/if}
        </div>
      </div>
      <div
        class="tap-zone right"
        on:click={() => handlePhysicalSide("right")}
        on:contextmenu={onRightClick}
        role="button"
        tabindex="0"
      >
        <div class="indicator-spot {isSwapped ? 'green-spot' : 'red-spot'}">
          {#if isSwapped}YES{:else}NO{/if}
        </div>
      </div>
    {/if}

    {#if showSettings}
      <div class="modal-overlay">
        <div class="modal surface">
          <h2>Configure Rules</h2>

          <div class="settings-section">
            <h3>Session Timer</h3>
            <div class="persp-controls">
                <label class="rule-checkbox">
                    <input type="checkbox" bind:checked={useTimer} on:change={() => { updateSettings(); timeLeftSeconds = gameDurationSeconds; }}>
                    <div>
                        <span class="rule-name">Enable Timer (Required for History)</span>
                        <div class="sub-text">Timer starts once the N-Back warmup turns are done.</div>
                    </div>
                </label>
                
                {#if useTimer}
                <div class="timer-adjust-group">
                    <span class="sub-text" style="margin: 0; padding-bottom: 5px; width: 80px;">Duration:</span>
                    
                    <div class="time-inputs-row">
                        <div class="time-field">
                            <input type="number" min="0" max="59" 
                                bind:value={configMinutes} 
                                on:change={updateDurationFromInputs}
                                class="time-input"
                            />
                            <span class="time-label">min</span>
                        </div>
                        
                        <span class="time-colon">:</span>

                        <div class="time-field">
                            <input type="number" min="0" max="59" 
                                bind:value={configSeconds} 
                                on:change={updateDurationFromInputs}
                                class="time-input"
                            />
                            <span class="time-label">sec</span>
                        </div>
                    </div>
                </div>
                {/if}
            </div>
          </div>

          <div class="settings-section">
            <h3>Active Frames</h3>
            <div class="rules-grid">
              {#each ALL_ATTRIBUTES as rule}
                {#if perspectiveMode && perspectiveType === "spatial" && (rule === "shape" || rule === "size")}
                  <label
                    class="rule-checkbox disabled"
                    title="Invariant in Spatial Mode"
                  >
                    <input type="checkbox" disabled checked={false} />
                    <span class="rule-name">{rule.toUpperCase()}</span>
                  </label>
                {:else}
                  <label class="rule-checkbox">
                    <input
                      type="checkbox"
                      checked={activeRules.includes(rule)}
                      on:change={() => toggleRule(rule)}
                    />
                    <span class="rule-name">{rule.toUpperCase()}</span>
                  </label>
                {/if}
              {/each}
            </div>
          </div>

          <div class="settings-section">
            <h3>Distractors</h3>
            <div class="persp-controls">
              <label class="rule-checkbox">
                <input
                  type="checkbox"
                  bind:checked={showStroop}
                  on:change={updateSettings}
                />
                <span class="rule-name">Stroop Overlay (Text)</span>
              </label>
              <label class="rule-checkbox">
                <input
                  type="checkbox"
                  bind:checked={distractorsEnabled}
                  on:change={updateSettings}
                />
                <div>
                  <span class="rule-name">Visual Noise</span>
                  <div class="sub-text">
                    Vary features (Shape/Size) even if not active rules.
                  </div>
                </div>
              </label>
              <label class="rule-checkbox">
                <input
                  type="checkbox"
                  bind:checked={controlSwapEnabled}
                  on:change={updateSettings}
                />
                <div>
                  <span class="rule-name">Random Control Swap</span>
                  <div class="sub-text">
                    35% chance controls flip (Left=No, Right=Yes).
                  </div>
                </div>
              </label>
            </div>
          </div>

          <div class="settings-section">
            <h3>Perspective Mode</h3>
            <div class="persp-controls">
              <label class="rule-checkbox">
                <input
                  type="checkbox"
                  bind:checked={perspectiveMode}
                  on:change={updateSettings}
                />
                <div><span class="rule-name">Enable Perspective</span></div>
              </label>

              {#if perspectiveMode}
                <div class="type-select">
                  <span class="sub-text">Type:</span>

                  <div class="tooltip-container">
                    <button
                      class:selected={perspectiveType === "symbolic"}
                      on:click={() => {
                        perspectiveType = "symbolic";
                        updateSettings();
                      }}
                    >
                      Symbolic
                    </button>
                    <div class="tooltip-card">
                        <h4>Abstract Logic</h4>
                        <p><strong>Rule Transformation:</strong></p>
                        <ul><li><strong>Me:</strong> Stay.</li><li><strong>Them:</strong> Flip using Key.</li></ul>
                        <p class="t-hint">Match <u>Decoded</u> Result.</p>
                    </div>
                  </div>

                  <div class="tooltip-container">
                    <button
                      class:selected={perspectiveType === "spatial"}
                      on:click={() => {
                        perspectiveType = "spatial";
                        activeRules = activeRules.filter(
                          (r) => r !== "shape" && r !== "size",
                        );
                        if (activeRules.length === 0) activeRules = ["color"];
                        updateSettings();
                      }}
                    >
                      Spatial
                    </button>
                    <div class="tooltip-card wide">
                        <h4>3D Perspective</h4>
                        <p><strong>Whose eyes are looking?</strong></p>
                        <ul>
                          <li><strong>ME:</strong> Stay in seat.</li>
                          <li><strong>THEM:</strong> Jump to opposite seat.</li>
                        </ul>
                        <p class="t-hint">Match <u>Observer's</u> Experience.</p>
                    </div>
                  </div>
                </div>

                {#if perspectiveType === 'spatial'}
                <div class="type-select" style="margin-top:15px; border-top:1px solid #333; padding-top:10px;">
                    <span class="sub-text">Strategy:</span>
                    
                    <div class="tooltip-container">
                        <button
                          class:selected={spatialMatchMode === "view"}
                          on:click={() => { spatialMatchMode = "view"; updateSettings(); }}
                        >
                          Observer View
                        </button>
                        <div class="tooltip-card">
                            <h4>Spatial Empathy</h4>
                            <p>Match what the <strong>EYES</strong> see.</p>
                            <p class="t-hint">Red View ≠ Green View</p>
                        </div>
                    </div>

                    <div class="tooltip-container">
                        <button
                          class:selected={spatialMatchMode === "object"}
                          on:click={() => { spatialMatchMode = "object"; updateSettings(); }}
                        >
                          Object Identity
                        </button>
                        <div class="tooltip-card">
                            <h4>Object Constancy</h4>
                            <p>Match the <strong>CUBE</strong> itself.</p>
                            <p class="t-hint">Red/Green Cube = Red/Green Cube</p>
                        </div>
                    </div>
                </div>

                  <div class="type-select" style="margin-top: 10px;">
                    <span class="sub-text">Vis:</span>
                    <button
                      class:selected={spatialType === "rotation"}
                      on:click={() => { spatialType = "rotation"; updateSettings(); }}>Rotation</button
                    >
                    <button
                      class:selected={spatialType === "instant"}
                      on:click={() => { spatialType = "instant"; updateSettings(); }}>Instant</button
                    >
                    <button
                      class:selected={spatialType === "folding"}
                      on:click={() => { spatialType = "folding"; updateSettings(); }}>Folding</button
                    >
                    <button
                      class:selected={spatialType === "cutout"}
                      on:click={() => { spatialType = "cutout"; updateSettings(); }}>Cutout</button
                    >
                  </div>
                {/if}
              {/if}
            </div>
          </div>
          <button class="btn-primary close-btn" on:click={toggleSettings}>Done</button>
        </div>
      </div>
    {/if}

    {#if showHistory}
        <div class="modal-overlay">
            <div class="modal surface">
                <h2>Progress</h2>
                <div class="chart-container"><canvas bind:this={canvasChart}></canvas></div>
                <button class="btn-secondary close-btn" on:click={toggleHistory}>Close</button>
            </div>
        </div>
    {/if}
  </div>
</main>

<style>
  /* --- BASE STYLES --- */
  main {
    position: relative;
    width: 100vw;
    height: 100vh;
    overflow: hidden;
    background-color: var(--bg-primary);
    color: var(--text-primary);
    font-family: "Inter", sans-serif;
  }
  .three-container {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    z-index: 1;
  }
  .stroop-overlay {
    position: absolute;
    top: 50%;
    left: 50%;
    transform: translate(-50%, -50%);
    font-size: 5rem;
    font-weight: 900;
    opacity: 0.35;
    z-index: 2;
    pointer-events: none;
    user-select: none;
    letter-spacing: 5px;
    text-transform: uppercase;
  }
  .ui-layer {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    z-index: 10;
    pointer-events: none;
  }
  .surface {
    background-color: var(--bg-surface);
    border: 1px solid var(--border-color);
    border-radius: 12px;
    box-shadow: 0 4px 12px rgba(0, 0, 0, 0.1);
    pointer-events: auto;
  }
  .hud {
    position: absolute;
    top: 20px;
    left: 50%;
    transform: translateX(-50%);
    display: flex;
    gap: 1.5rem;
    padding: 0.75rem 1.5rem;
    align-items: center;
    white-space: nowrap;
    z-index: 50;
  }

  /* --- TOOLTIPS --- */
  .tooltip-container {
    position: relative;
    display: inline-block;
  }
  .tooltip-card {
    position: absolute;
    bottom: 125%;
    left: 50%;
    transform: translateX(-50%) translateY(10px);
    width: 200px;
    background-color: rgba(15, 15, 20, 0.95);
    color: #fff;
    border: 1px solid rgba(255, 255, 255, 0.15);
    box-shadow: 0 8px 20px rgba(0, 0, 0, 0.5);
    padding: 1rem;
    border-radius: 8px;
    z-index: 200;
    opacity: 0;
    visibility: hidden;
    pointer-events: none;
    transition: all 0.2s cubic-bezier(0.25, 0.8, 0.25, 1);
    text-align: left;
    white-space: normal; /* Fix for wide text */
  }
  .tooltip-card.wide { width: 240px; }
  .tooltip-card::after {
    content: "";
    position: absolute;
    top: 100%;
    left: 50%;
    margin-left: -6px;
    border-width: 6px;
    border-style: solid;
    border-color: rgba(15, 15, 20, 0.95) transparent transparent transparent;
  }
  .tooltip-container:hover .tooltip-card {
    opacity: 1;
    visibility: visible;
    transform: translateX(-50%) translateY(0);
  }
  .tooltip-card h4 {
    margin: 0 0 0.5rem 0;
    color: var(--accent-primary);
    font-size: 0.9rem;
    text-transform: uppercase;
    letter-spacing: 1px;
    border-bottom: 1px solid rgba(255,255,255,0.1);
    padding-bottom: 0.25rem;
  }
  .tooltip-card p {
    margin: 0.5rem 0 0.25rem 0;
    font-size: 0.75rem;
    color: #ddd;
  }
  .tooltip-card ul {
    margin: 0;
    padding-left: 1rem;
    list-style-type: disc;
  }
  .tooltip-card li {
    font-size: 0.75rem;
    color: #bbb;
    margin-bottom: 0.2rem;
    line-height: 1.3;
  }
  .tooltip-card .t-hint {
    margin-top: 0.75rem;
    font-style: italic;
    color: var(--accent-primary);
    text-align: center;
    font-weight: bold;
  }
  .tooltip-card .t-hint u {
    text-decoration-color: var(--accent-primary);
    text-decoration-thickness: 2px;
  }


  /* --- UI ELEMENTS --- */
  .clue-box {
    background: var(--bg-secondary);
    border: 2px solid var(--accent-primary);
    padding: 0.5rem 1rem;
    border-radius: 8px;
    text-align: center;
    min-width: 140px;
    animation: pulse 2s infinite;
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 4px;
  }
  .clue-box.spatial {
    border-color: #9c27b0;
    animation: pulse-purple 2s infinite;
  }
  .clue-title { font-size: 0.6rem; color: var(--text-secondary); font-weight: bold; letter-spacing: 1px; }
  .clue-row { display: flex; align-items: center; gap: 8px; font-weight: 800; font-size: 1.1rem; }
  .you-val { color: var(--accent-primary); font-size: 1.2rem; }
  .clue-sub { font-size: 0.6rem; color: var(--text-primary); font-style: italic; }
  .clue-sub.warning { color: #dc3545; font-weight: bold; }

  /* NET KEY */
  .net-key-container { position: absolute; top: 20px; left: 20px; z-index: 60; background: var(--bg-surface); border: 2px solid #9c27b0; padding: 0.75rem; border-radius: 12px; text-align: center; box-shadow: 0 4px 12px rgba(0, 0, 0, 0.3); animation: pulse-purple 3s infinite; pointer-events: auto; }
  .net-key { display: flex; flex-direction: column; gap: 2px; margin-top: 8px; margin-bottom: 8px; }
  .net-row { display: flex; justify-content: center; gap: 2px; }
  .net-cell { width: 24px; height: 24px; border: 1px solid rgba(0, 0, 0, 0.2); border-radius: 3px; }
  .net-cell.empty { background: transparent; border: none; }
  .net-cell.front-face { color: white; font-size: 0.8rem; font-weight: 900; display: flex; justify-content: center; align-items: center; text-shadow: 0 1px 2px black; border: 2px solid white; }

  .stat-group { display: flex; gap: 1.5rem; border-right: 1px solid var(--border-color); padding-right: 1.5rem; }
  .controls-group { display: flex; gap: 1rem; align-items: center; }
  .stat { display: flex; flex-direction: column; align-items: center; }
  .label { font-size: 0.7rem; color: var(--text-secondary); text-transform: uppercase; font-weight: 600; }
  .value { font-size: 1.2rem; font-weight: 700; }
  .active-rule { color: var(--accent-primary); }
  .timer-value { color: #ff9800; }
  
  .level-control { display: flex; flex-direction: column; align-items: center; }
  .btn-row { display: flex; gap: 4px; margin-top: 2px; }
  .btn-mini { padding: 2px 8px; font-size: 0.8rem; background: var(--bg-secondary); color: var(--text-primary); border: 1px solid var(--border-color); border-radius: 4px; cursor: pointer; }
  .btn-mini:disabled { opacity: 0.3; cursor: not-allowed; }
  
  .toggle-control { display: flex; align-items: center; }
  .switch-label { display: flex; flex-direction: column; align-items: center; cursor: pointer; }
  .label-text { font-size: 0.6rem; color: var(--text-secondary); margin-top: 2px; }

  button { padding: 0.5rem 1rem; border-radius: 6px; border: none; font-weight: 600; cursor: pointer; transition: opacity 0.2s; }
  button:hover { opacity: 0.9; }
  .btn-primary { background-color: var(--accent-primary); color: var(--text-on-accent); }
  .btn-secondary { background-color: var(--bg-secondary); color: var(--text-secondary); }
  .btn-danger { background-color: #dc3545; color: white; }

  /* FEEDBACK */
  .toast-notification { position: absolute; top: 100px; left: 50%; transform: translateX(-50%); padding: 0.5rem 1rem; font-weight: 600; color: var(--text-primary); animation: slideDown 0.3s ease-out; }
  .feedback { color: white; position: absolute; top: 40%; left: 50%; transform: translate(-50%, -50%); font-size: 3rem; font-weight: 900; text-shadow: 0 2px 10px rgba(0, 0, 0, 0.2); animation: popIn 0.2s ease-out; z-index: 20; }
  .text-green-500 { color: #28a745; }
  .text-red-500 { color: #dc3545; }
  .text-accent { color: var(--accent-primary); }

  /* TAP ZONES */
  .tap-zone { position: absolute; bottom: 0; width: 50%; height: 70%; pointer-events: auto; cursor: pointer; transition: background-color 0.1s; display: flex; justify-content: center; align-items: flex-end; padding-bottom: 4rem; }
  .tap-zone:hover { background-color: rgba(255, 255, 255, 0.05); }
  .tap-zone.left { left: 0; border-right: 1px solid rgba(255, 255, 255, 0.1); }
  .tap-zone.right { right: 0; }
  .indicator-spot { width: 80px; height: 80px; border-radius: 50%; display: flex; justify-content: center; align-items: center; font-weight: 900; color: white; font-size: 1.2rem; text-shadow: 0 2px 4px rgba(0, 0, 0, 0.3); box-shadow: 0 0 20px rgba(0, 0, 0, 0.2); }
  .green-spot { background-color: rgba(40, 167, 69, 0.6); border: 3px solid #28a745; }
  .red-spot { background-color: rgba(220, 53, 69, 0.6); border: 3px solid #dc3545; }

  /* AVATAR */
  .avatar-layer { position: absolute; top: 0; left: 0; width: 100%; height: 100%; pointer-events: none; z-index: 5; }
  .avatar-top { position: absolute; top: 100px; left: 50%; transform: translateX(-50%); display: flex; flex-direction: column; align-items: center; animation: slideDown 0.5s ease-out; }
  .avatar-bottom { position: absolute; bottom: 0; left: 50%; transform: translateX(-50%); display: flex; flex-direction: column; align-items: center; animation: slideUp 0.5s ease-out; }
  .avatar-svg { width: 120px; height: 120px; filter: drop-shadow(0 4px 6px rgba(0, 0, 0, 0.3)); }
  .avatar-label { font-size: 1rem; font-weight: 900; color: var(--text-primary); background: var(--bg-surface); padding: 2px 8px; border-radius: 4px; margin-top: -10px; box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1); }

  /* SETTINGS */
  .modal-overlay { position: fixed; top: 0; left: 0; width: 100%; height: 100%; background: rgba(0, 0, 0, 0.5); display: flex; justify-content: center; align-items: center; pointer-events: auto; z-index: 100; }
  .modal { padding: 2rem; width: 90%; max-width: 600px; color: var(--text-primary); max-height: 80vh; overflow-y: auto; }
  .settings-section { margin-bottom: 1.5rem; }
  .settings-section h3 { font-size: 0.9rem; text-transform: uppercase; color: var(--text-secondary); border-bottom: 1px solid var(--border-color); padding-bottom: 0.5rem; margin-bottom: 1rem; }
  .sub-text { font-size: 0.8rem; color: var(--text-secondary); display: block; margin-top: 0.5rem; }
  .persp-controls { display: flex; flex-direction: column; gap: 1rem; }
  .type-select { display: flex; gap: 1rem; align-items: center; flex-wrap: wrap; }
  .type-select button { background: transparent; border: 1px solid var(--border-color); color: var(--text-secondary); }
  .type-select button.selected { background: var(--accent-primary); color: white; border-color: var(--accent-primary); }
  .rule-checkbox { display: flex; align-items: flex-start; gap: 0.5rem; cursor: pointer; padding: 0.5rem; border: 1px solid var(--border-color); border-radius: 6px; }
  .rule-name { font-size: 0.9rem; font-weight: 600; }
  .rule-checkbox.disabled { opacity: 0.5; cursor: not-allowed; background-color: var(--bg-secondary); }
  .rules-grid { display: grid; grid-template-columns: repeat(auto-fill, minmax(100px, 1fr)); gap: 1rem; }
  .close-btn { width: 100%; margin-top: 1rem; }
  .chart-container { position: relative; height: 300px; width: 100%; margin: 1rem 0; }

  /* TIMER CONTROLS */
  .timer-adjust-group { display: flex; gap: 10px; align-items: center; margin-top: 10px; }
  .time-inputs-row { display: flex; align-items: center; gap: 8px; }
  .time-field { display: flex; flex-direction: column; align-items: center; }
  .time-input { padding: 5px; border: 1px solid var(--border-color); border-radius: 4px; background: var(--bg-secondary); color: var(--text-primary); width: 50px; text-align: center; font-size: 1rem; font-weight: bold; }
  .time-colon { font-size: 1.2rem; font-weight: bold; margin-bottom: 12px; color: var(--text-secondary); }
  .time-label { font-size: 0.6rem; color: var(--text-secondary); margin-top: 2px; }

  /* ANIMATIONS */
  @keyframes slideDown { from { transform: translate(-50%, -20px); opacity: 0; } to { transform: translate(-50%, 0); opacity: 1; } }
  @keyframes slideUp { from { transform: translate(-50%, 20px); opacity: 0; } to { transform: translate(-50%, 0); opacity: 1; } }
  @keyframes popIn { from { transform: translate(-50%, -50%) scale(0.5); opacity: 0; } to { transform: translate(-50%, -50%) scale(1); opacity: 1; } }
  @keyframes pulse { 0% { box-shadow: 0 0 0 0 rgba(0, 123, 255, 0.4); } 70% { box-shadow: 0 0 0 10px rgba(0, 123, 255, 0); } 100% { box-shadow: 0 0 0 0 rgba(0, 123, 255, 0); } }
  @keyframes pulse-purple { 0% { box-shadow: 0 0 0 0 rgba(156, 39, 176, 0.4); } 70% { box-shadow: 0 0 0 10px rgba(156, 39, 176, 0); } 100% { box-shadow: 0 0 0 0 rgba(156, 39, 176, 0); } }
</style>