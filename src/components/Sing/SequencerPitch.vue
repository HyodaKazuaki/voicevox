<template>
  <div ref="canvasContainer" class="canvas-container"></div>
</template>

<script setup lang="ts">
import { ref, watch, computed, Ref } from "vue";
import * as PIXI from "pixi.js";
import AsyncLock from "async-lock";
import { useStore } from "@/store";
import {
  UNVOICED_PHONEMES,
  VALUE_INDICATING_NO_DATA,
  convertToFramePhonemes,
  frequencyToNoteNumber,
  secondToTick,
} from "@/sing/domain";
import {
  PitchData,
  PitchDataHash,
  calculatePitchDataHash,
  noteNumberToBaseY,
  tickToBaseX,
} from "@/sing/viewHelper";
import { Color, LineStrip } from "@/sing/graphics/lineStrip";
import {
  onMountedOrActivated,
  onUnmountedOrDeactivated,
} from "@/composables/onMountOrActivate";
import { ExhaustiveError } from "@/type/utility";
import { createLogger } from "@/helpers/log";
import { getLast } from "@/sing/utility";
import { getOrThrow } from "@/helpers/mapHelper";
import { EditorFrameAudioQuery } from "@/store/type";

type PitchLine = {
  color: Ref<Color>;
  readonly width: number;
  readonly pitchDataMap: Map<PitchDataHash, PitchData>;
  readonly lineStripMap: Map<PitchDataHash, LineStrip>;
};

const props = defineProps<{
  offsetX: number;
  offsetY: number;
  previewPitchEdit?:
    | { type: "draw"; data: number[]; startFrame: number }
    | { type: "erase"; startFrame: number; frameLength: number };
}>();

const { warn, error } = createLogger("SequencerPitch");
const store = useStore();
const tpqn = computed(() => store.state.tpqn);
const isDark = computed(() => store.state.currentTheme === "Dark");
const tempos = computed(() => store.state.tempos);
const pitchEditData = computed(() => {
  return store.getters.SELECTED_TRACK.pitchEditData;
});
const previewPitchEdit = computed(() => props.previewPitchEdit);
const selectedTrackId = computed(() => store.getters.SELECTED_TRACK_ID);
const editorFrameRate = computed(() => store.state.editorFrameRate);
const singingGuidesInSelectedTrack = computed(() => {
  const singingGuides: {
    query: EditorFrameAudioQuery;
    startTime: number;
  }[] = [];
  for (const phrase of store.state.phrases.values()) {
    if (phrase.trackId !== selectedTrackId.value) {
      continue;
    }
    if (phrase.queryKey == undefined) {
      continue;
    }
    const phraseQuery = getOrThrow(store.state.phraseQueries, phrase.queryKey);
    singingGuides.push({
      startTime: phrase.startTime,
      query: phraseQuery,
    });
  }
  return singingGuides;
});

// NOTE: ピッチラインの色をテーマに応じて調節する
// 動的カラースキーマに対応後、テーマに応じた色をオブジェクトから取得できるようにする
const originalPitchLineColorLight = new Color(156, 158, 156, 255);
const originalPitchLineColorDark = new Color(114, 116, 114, 255);
const originalPitchLineColor = ref(
  isDark.value ? originalPitchLineColorDark : originalPitchLineColorLight,
);
const originalPitchLine: PitchLine = {
  color: originalPitchLineColor,
  width: 1.125,
  pitchDataMap: new Map(),
  lineStripMap: new Map(),
};
const pitchEditLineColorLight = new Color(0, 167, 63, 255);
const pitchEditLineColorDark = new Color(95, 188, 117, 255);
const pitchEditLineColor = ref(
  isDark.value ? pitchEditLineColorDark : pitchEditLineColorLight,
);
const pitchEditLine: PitchLine = {
  color: pitchEditLineColor,
  width: 2.25,
  pitchDataMap: new Map(),
  lineStripMap: new Map(),
};

const canvasContainer = ref<HTMLElement | null>(null);
let resizeObserver: ResizeObserver | undefined;
let canvasWidth: number | undefined;
let canvasHeight: number | undefined;

let renderer: PIXI.Renderer | undefined;
let stage: PIXI.Container | undefined;
let requestId: number | undefined;
let renderInNextFrame = false;

const updateLineStrips = (pitchLine: PitchLine) => {
  if (stage == undefined) {
    throw new Error("stage is undefined.");
  }
  if (canvasWidth == undefined) {
    throw new Error("canvasWidth is undefined.");
  }
  const tpqn = store.state.tpqn;
  const canvasWidthValue = canvasWidth;
  const zoomX = store.state.sequencerZoomX;
  const zoomY = store.state.sequencerZoomY;
  const offsetX = props.offsetX;
  const offsetY = props.offsetY;

  const removedLineStrips: LineStrip[] = [];

  // 無くなったピッチデータを調べて、そのピッチデータに対応するLineStripを削除する
  for (const [key, lineStrip] of pitchLine.lineStripMap) {
    if (!pitchLine.pitchDataMap.has(key)) {
      stage.removeChild(lineStrip.displayObject);
      removedLineStrips.push(lineStrip);
      pitchLine.lineStripMap.delete(key);
    }
  }

  // ピッチデータに対応するLineStripが無かったら作成する
  for (const [key, pitchData] of pitchLine.pitchDataMap) {
    if (pitchLine.lineStripMap.has(key)) {
      const currentLineStrip = pitchLine.lineStripMap.get(key)!;
      // テーマなど色が変更された場合、LineStripを再作成する
      if (!currentLineStrip.color.equals(pitchLine.color.value)) {
        stage.removeChild(currentLineStrip.displayObject);
        currentLineStrip.destroy();
        pitchLine.lineStripMap.delete(key);
      } else {
        continue;
      }
    }
    const dataLength = pitchData.data.length;

    // 再利用できるLineStripがあれば再利用し、なければLineStripを作成する
    let lineStrip = removedLineStrips.pop();
    if (lineStrip != undefined) {
      if (
        !lineStrip.color.equals(pitchLine.color.value) ||
        lineStrip.width !== pitchLine.width
      ) {
        throw new Error("Color or width does not match.");
      }
      lineStrip.numOfPoints = dataLength;
    } else {
      lineStrip = new LineStrip(
        dataLength,
        pitchLine.color.value,
        pitchLine.width,
      );
    }
    // pitchEditLineの場合は最後に追加する（originalより前面に表示）
    if (pitchLine === pitchEditLine) {
      stage.addChild(lineStrip.displayObject);
    } else {
      // originalLineは最初に追加する（EditLineの背面に表示）
      stage.addChildAt(lineStrip.displayObject, 0);
    }
    pitchLine.lineStripMap.set(key, lineStrip);
  }

  // 再利用されなかったLineStripは破棄する
  for (const lineStrip of removedLineStrips) {
    lineStrip.destroy();
  }

  // LineStripを更新
  for (const [key, pitchData] of pitchLine.pitchDataMap) {
    const lineStrip = pitchLine.lineStripMap.get(key);
    if (lineStrip == undefined) {
      throw new Error("lineStrip is undefined.");
    }

    // カリングを行う
    const startTicks = pitchData.ticksArray[0];
    const startBaseX = tickToBaseX(startTicks, tpqn);
    const startX = startBaseX * zoomX - offsetX;
    const lastTicks = getLast(pitchData.ticksArray);
    const lastBaseX = tickToBaseX(lastTicks, tpqn);
    const lastX = lastBaseX * zoomX - offsetX;
    if (startX >= canvasWidthValue || lastX <= 0) {
      lineStrip.renderable = false;
      continue;
    }
    lineStrip.renderable = true;

    // ポイントを計算してlineStripに設定＆更新
    for (let i = 0; i < pitchData.data.length; i++) {
      const ticks = pitchData.ticksArray[i];
      const baseX = tickToBaseX(ticks, tpqn);
      const x = baseX * zoomX - offsetX;
      const freq = pitchData.data[i];
      const noteNumber = frequencyToNoteNumber(freq);
      const baseY = noteNumberToBaseY(noteNumber);
      const y = baseY * zoomY - offsetY;
      lineStrip.setPoint(i, x, y);
    }
    lineStrip.update();
  }
};

const render = () => {
  if (renderer == undefined) {
    throw new Error("renderer is undefined.");
  }
  if (stage == undefined) {
    throw new Error("stage is undefined.");
  }

  // シンガーが未設定の場合はピッチラインをすべて非表示にして終了
  const singer = store.getters.SELECTED_TRACK.singer;
  if (!singer) {
    for (const lineStrip of originalPitchLine.lineStripMap.values()) {
      lineStrip.renderable = false;
    }
    for (const lineStrip of pitchEditLine.lineStripMap.values()) {
      lineStrip.renderable = false;
    }
    renderer.render(stage);
    return;
  }

  // ピッチラインのLineStripを更新する
  updateLineStrips(originalPitchLine);
  updateLineStrips(pitchEditLine);

  renderer.render(stage);
};

const toPitchData = (framewiseData: number[], frameRate: number): PitchData => {
  const data = framewiseData;
  const ticksArray: number[] = [];
  for (let i = 0; i < data.length; i++) {
    const ticks = secondToTick(i / frameRate, tempos.value, tpqn.value);
    ticksArray.push(ticks);
  }
  return { ticksArray, data };
};

const splitPitchData = (pitchData: PitchData, delimiter: number) => {
  const ticksArray = pitchData.ticksArray;
  const data = pitchData.data;
  const pitchDataArray: PitchData[] = [];
  for (let i = 0; i < data.length; i++) {
    if (data[i] !== delimiter) {
      if (i === 0 || data[i - 1] === delimiter) {
        pitchDataArray.push({ ticksArray: [], data: [] });
      }
      const lastPitchData = getLast(pitchDataArray);
      lastPitchData.ticksArray.push(ticksArray[i]);
      lastPitchData.data.push(data[i]);
    }
  }
  return pitchDataArray;
};

const setPitchDataToPitchLine = async (
  pitchData: PitchData,
  pitchLine: PitchLine,
) => {
  const partialPitchDataArray = splitPitchData(
    pitchData,
    VALUE_INDICATING_NO_DATA,
  ).filter((value) => value.data.length >= 2);

  pitchLine.pitchDataMap.clear();
  for (const partialPitchData of partialPitchDataArray) {
    const hash = await calculatePitchDataHash(partialPitchData);
    pitchLine.pitchDataMap.set(hash, partialPitchData);
  }
};

const generateOriginalPitchData = () => {
  const unvoicedPhonemes = UNVOICED_PHONEMES;
  const frameRate = editorFrameRate.value; // f0（元のピッチ）はエディターのフレームレートで表示する

  // 選択中のトラックで使われている歌い方のf0を結合してピッチデータを生成する
  const tempData = [];
  for (const singingGuide of singingGuidesInSelectedTrack.value) {
    // TODO: 補間を行うようにする
    if (singingGuide.query.frameRate !== frameRate) {
      throw new Error(
        "The frame rate between the singing guide and the edit does not match.",
      );
    }
    const phonemes = singingGuide.query.phonemes;
    if (phonemes.length === 0) {
      throw new Error("phonemes.length is 0.");
    }
    const f0 = singingGuide.query.f0;

    // 各フレームの音素の配列を生成する
    const framePhonemes = convertToFramePhonemes(phonemes);
    if (f0.length !== framePhonemes.length) {
      throw new Error("f0.length and framePhonemes.length do not match.");
    }

    // 歌い方の開始フレームと終了フレームを計算する
    const singingGuideFrameLength = f0.length;
    const singingGuideStartFrame = Math.round(
      singingGuide.startTime * frameRate,
    );
    const singingGuideEndFrame =
      singingGuideStartFrame + singingGuideFrameLength;

    // 無声子音区間以外のf0をtempDataにコピーする
    // NOTE: 無声子音区間は音程が無く、f0の値が大きく上下するので表示しない
    if (tempData.length < singingGuideEndFrame) {
      const valuesToPush = new Array(
        singingGuideEndFrame - tempData.length,
      ).fill(VALUE_INDICATING_NO_DATA);
      tempData.push(...valuesToPush);
    }
    const startFrame = Math.max(0, singingGuideStartFrame);
    const endFrame = singingGuideEndFrame;
    for (let i = startFrame; i < endFrame; i++) {
      const phoneme = framePhonemes[i - singingGuideStartFrame];
      const unvoiced = unvoicedPhonemes.includes(phoneme);
      if (!unvoiced) {
        tempData[i] = f0[i - singingGuideStartFrame];
      }
    }
  }
  return toPitchData(tempData, frameRate);
};

const generatePitchEditData = () => {
  const frameRate = editorFrameRate.value;

  const tempData = [...pitchEditData.value];
  // プレビュー中のピッチ編集があれば、適用する
  if (previewPitchEdit.value != undefined) {
    const previewPitchEditType = previewPitchEdit.value.type;
    if (previewPitchEditType === "draw") {
      const previewData = previewPitchEdit.value.data;
      const previewStartFrame = previewPitchEdit.value.startFrame;
      const previewEndFrame = previewStartFrame + previewData.length;
      if (tempData.length < previewEndFrame) {
        const valuesToPush = new Array(previewEndFrame - tempData.length).fill(
          VALUE_INDICATING_NO_DATA,
        );
        tempData.push(...valuesToPush);
      }
      for (let i = 0; i < previewData.length; i++) {
        tempData[previewStartFrame + i] = previewData[i];
      }
    } else if (previewPitchEditType === "erase") {
      const startFrame = previewPitchEdit.value.startFrame;
      const endFrame = Math.min(
        startFrame + previewPitchEdit.value.frameLength,
        tempData.length,
      );
      for (let i = startFrame; i < endFrame; i++) {
        tempData[i] = VALUE_INDICATING_NO_DATA;
      }
    } else {
      throw new ExhaustiveError(previewPitchEditType);
    }
  }
  return toPitchData(tempData, frameRate);
};

const asyncLock = new AsyncLock({ maxPending: 1 });

watch(
  [singingGuidesInSelectedTrack, tempos, tpqn],
  async () => {
    asyncLock.acquire(
      "originalPitch",
      async () => {
        const originalPitchData = generateOriginalPitchData();
        await setPitchDataToPitchLine(originalPitchData, originalPitchLine);
        renderInNextFrame = true;
      },
      (err) => {
        if (err != undefined) {
          warn(`An error occurred.`, err);
        }
      },
    );
  },
  { immediate: true },
);

watch(
  [pitchEditData, previewPitchEdit, tempos, tpqn],
  async () => {
    asyncLock.acquire(
      "pitchEdit",
      async () => {
        const pitchEditData = generatePitchEditData();
        await setPitchDataToPitchLine(pitchEditData, pitchEditLine);
        renderInNextFrame = true;
      },
      (err) => {
        if (err != undefined) {
          warn(`An error occurred.`, err);
        }
      },
    );
  },
  { immediate: true },
);

// ピッチラインカラーをテーマに合わせて変更
watch(
  [isDark],
  () => {
    const newOriginalPitchLineColor = isDark.value
      ? originalPitchLineColorDark
      : originalPitchLineColorLight;
    const newPitchEditLineColor = isDark.value
      ? pitchEditLineColorDark
      : pitchEditLineColorLight;
    originalPitchLineColor.value = newOriginalPitchLineColor;
    pitchEditLineColor.value = newPitchEditLineColor;
    renderInNextFrame = true;
  },
  { immediate: true },
);

watch(
  () => [
    store.state.sequencerZoomX,
    store.state.sequencerZoomY,
    props.offsetX,
    props.offsetY,
  ],
  () => {
    renderInNextFrame = true;
  },
);

onMountedOrActivated(() => {
  const canvasContainerElement = canvasContainer.value;
  if (!canvasContainerElement) {
    throw new Error("canvasContainerElement is null.");
  }

  canvasWidth = canvasContainerElement.clientWidth;
  canvasHeight = canvasContainerElement.clientHeight;

  const canvasElement = document.createElement("canvas");
  canvasElement.width = canvasWidth;
  canvasElement.height = canvasHeight;
  canvasContainerElement.appendChild(canvasElement);

  renderer = new PIXI.Renderer({
    view: canvasElement,
    backgroundAlpha: 0,
    antialias: true,
    resolution: window.devicePixelRatio || 1,
    autoDensity: true,
  });
  stage = new PIXI.Container();

  // webGLVersionをチェックする
  // 2未満の場合、ピッチの表示ができないのでエラーとしてロギングする
  const webGLVersion = renderer.context.webGLVersion;
  if (webGLVersion < 2) {
    error(`webGLVersion is less than 2. webGLVersion: ${webGLVersion}`);
  }

  const callback = () => {
    if (renderInNextFrame) {
      render();
      renderInNextFrame = false;
    }
    requestId = window.requestAnimationFrame(callback);
  };
  requestId = window.requestAnimationFrame(callback);
  renderInNextFrame = true;

  resizeObserver = new ResizeObserver(() => {
    if (renderer == undefined) {
      throw new Error("renderer is undefined.");
    }
    const canvasContainerWidth = canvasContainerElement.clientWidth;
    const canvasContainerHeight = canvasContainerElement.clientHeight;

    if (canvasContainerWidth > 0 && canvasContainerHeight > 0) {
      canvasWidth = canvasContainerWidth;
      canvasHeight = canvasContainerHeight;
      renderer.resize(canvasWidth, canvasHeight);
      renderInNextFrame = true;
    }
  });
  resizeObserver.observe(canvasContainerElement);
});

onUnmountedOrDeactivated(() => {
  if (requestId != undefined) {
    window.cancelAnimationFrame(requestId);
  }
  stage?.destroy();
  originalPitchLine.lineStripMap.forEach((value) => value.destroy());
  originalPitchLine.lineStripMap.clear();
  pitchEditLine.lineStripMap.forEach((value) => value.destroy());
  pitchEditLine.lineStripMap.clear();
  renderer?.destroy(true);
  resizeObserver?.disconnect();
});
</script>

<style scoped lang="scss">
@use "@/styles/v2/variables" as vars;

.canvas-container {
  overflow: hidden;
  z-index: vars.$z-index-sing-pitch;
  pointer-events: none;
  position: relative;

  contain: strict; // canvasのサイズが変わるのを無視する
}
</style>
