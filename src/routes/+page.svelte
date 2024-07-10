<script lang="ts">
  import { tweened } from "svelte/motion";
  import { fade } from "svelte/transition";

  import { FFmpeg } from "@ffmpeg/ffmpeg";

  type Status =
    | "loading"
    | "loaded"
    | "convert.start"
    | "convert.done"
    | "convert.error";

  let status = $state<Status>("loading");
  let error = $state("");
  let ffmpeg: FFmpeg;
  let progress = tweened(0);

  const loadFFmpeg = async () => {
    const baseURL = "https://unpkg.com/@ffmpeg/core@0.12.6/dist/esm";
    ffmpeg = new FFmpeg();

    ffmpeg.on("progress", (event: { progress: number }) => {
      progress.set(event.progress * 100);
    });

    status = "loading";
    await ffmpeg.load({
      coreURL: `${baseURL}/ffmpeg-core.js`,
      wasmURL: `${baseURL}/ffmpeg-core.wasm`,
    });
    status = "loaded";
  };

  const fetchFile = async (file: File): Promise<Uint8Array> => {
    return new Uint8Array(await file.arrayBuffer());
  };

  const convertVideo = async (video: File) => {
    status = "convert.start";
    error = "";

    const videoData = await fetchFile(video);
    await ffmpeg.writeFile("input.webm", videoData);
    await ffmpeg.exec(["-i", "input.webm", "output.mp4"]);

    const data = await ffmpeg.readFile("output.mp4");
    status = "convert.done";

    return data;
  };

  const download = (data: Uint8Array) => {
    const a = document.createElement("a");
    a.href = URL.createObjectURL(new Blob([data], { type: "video/mp4" }));
    a.download = "output.mp4";
    a.click();
  };

  const onDropped = async (event: DragEvent) => {
    event.preventDefault();

    const files = event.dataTransfer?.files || [];
    if (files.length === 0) {
      return;
    }

    if (files.length > 1) {
      error = "Only one file is allowed";
      return;
    }

    const file = files[0];
    if (file.type !== "video/webm") {
      error = "Only WebM files are allowed";
      return;
    }

    const data = await convertVideo(file);
    download(data);
  };

  const onDraggedOver = (event: DragEvent) => {
    event.preventDefault();
  };

  $effect(() => {
    loadFFmpeg();
  });
</script>

<h1 class="title">WebM to MP4 Converter</h1>

<!-- svelte-ignore a11y_no_static_element_interactions -->
<div
  ondrop={onDropped}
  ondragover={onDraggedOver}
  data-status={status}
  class="drop"
>
  {#if status === "loading"}
    <p in:fade>Loading FFMpeg ....</p>
  {/if}

  {#if status === "loaded"}
    <p in:fade>Drag video here</p>
  {/if}

  {#if status === "convert.start"}
    <p in:fade>Converting...</p>

    <div class="progress-bar">
      <div class="progress" style:--progress="{$progress}%">
        {$progress.toFixed(0)}%
      </div>
    </div>
  {/if}

  {#if status === "convert.done"}
    <p in:fade>Done!</p>
  {/if}

  {#if error}
    <p in:fade class="error">{error}</p>
  {/if}
</div>

<style>
  .title {
    text-align: center;
  }

  .drop {
    width: 600px;
    height: 400px;
    display: grid;
    place-content: center;
    margin-block-start: 2rem;
    border: 10px dashed green;
    border-radius: 8px;

    & p {
      font-size: 2rem;
      text-align: center;

      &.error {
        color: hsl(9 100% 64%);
      }
    }
  }

  .progress-bar {
    --progress-bar-clr: hsl(180 100% 50%);
    --progress-txt-clr: hsl(0 0% 0%);

    width: 300px;
    height: 40px;
    position: relative;
    font-weight: 700;
    background-color: hsl(200 10% 14%);
    border-radius: 8px;

    & .progress {
      width: var(--progress);
      height: 100%;
      position: absolute;
      left: 0px;
      display: grid;
      place-content: center;
      background: var(--progress-bar-clr);
      color: var(--progress-txt-clr);
      border-radius: 8px;
    }
  }
</style>
