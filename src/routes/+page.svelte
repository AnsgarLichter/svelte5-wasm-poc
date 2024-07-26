<script lang="ts">
  import { tweened } from "svelte/motion";
  import { fade } from "svelte/transition";

  import { FFmpeg } from "@ffmpeg/ffmpeg";
  import type { Log } from "@ffmpeg/types";

  type Format = {
    demuxingSupported: boolean;
    muxingSupported: boolean;
    abbreviation: string;
    name: string;
  };

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
  let formats = $state<Format[]>([]);
  let selectedOutputFormat = $state("");

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

    ffmpeg.on("log", (event: Log) => {
      console.log(event.message);
    });

    const loggedFormats: Array<string> = [];
    const logFormatOutput = (event: Log) => {
      loggedFormats.push(event.message);
    };
    ffmpeg.on("log", logFormatOutput);

    await ffmpeg.exec(["-formats"]);

    formats = getFormatsFromLog(loggedFormats);
    const outputFormats = formats.filter((f) => f.muxingSupported);
    if(outputFormats.length > 0) {
      selectedOutputFormat = outputFormats[0].abbreviation;
    }
  };

  const getFormatsFromLog = (logMessages: Array<string>) => {
    const formatLineRegex = /^(D|DE|E)\s+(\S+)\s+(.+)$/;
    const results = [];

    for (const line of logMessages) {
      const trimmedLine = line.trim();

      // Skip lines that don't start with 'D', 'DE', or 'E'
      if (!formatLineRegex.test(trimmedLine)) {
        continue;
      }

      // Use a regex to match and capture support, abbreviation, and name
      const match = trimmedLine.match(formatLineRegex);
      if (!match) {
        continue;
      }

      const [, support, abbreviation, name] = match;
      results.push({
        demuxingSupported: support.includes("D"),
        muxingSupported: support.includes("E"),
        abbreviation,
        name,
      });
    }

    return results;
  };

  const fetchFile = async (file: File): Promise<Uint8Array> => {
    return new Uint8Array(await file.arrayBuffer());
  };

  const convertVideo = async (video: File) => {
    status = "convert.start";
    error = "";

    const videoData = await fetchFile(video);
    await ffmpeg.writeFile("input.webm", videoData);
    const result = await ffmpeg.exec(["-i", "input.webm", `output.${selectedOutputFormat}`]);
    if(result !== 0) {
      error = "Failed to convert video";
      status = "convert.error";
      return;
    }

    const data = await ffmpeg.readFile("output.mp4");
    status = "convert.done";

    return data;
  };

  const download = (data: Uint8Array | string) => {
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
    const fileFormat = file.name.split(".").pop() || "";
    const isFileFormatSupported = formats
      .filter((f) => f.demuxingSupported)
      .find((f) => file.name.endsWith(fileFormat));
    if (!fileFormat || !isFileFormatSupported) {
      error = `Data format ${fileFormat} not supported`;
      return;
    }

    const data = await convertVideo(file);
    if(!data) {
      return;
    }
    
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

<div class="formats">
  {#if formats.length === 0}
    <p>Loading supported formats...</p>
  {:else}
    <h2>Available Output Formats</h2>

    <fieldset>
      <legend>Select your output format:</legend>
      <div class="outputFormats">
        {#each formats as format}
          {#if format.muxingSupported}
            <div>
              <input
                type="radio"
                name="formats"
                id={format.abbreviation}
                value={format.abbreviation}
                bind:group={selectedOutputFormat}
              />
              <label for={format.abbreviation}
                >{format.name} ({format.abbreviation})</label
              >
            </div>
          {/if}
        {/each}
      </div>
    </fieldset>
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

  p,
  label {
    font:
      1rem "Fira Sans",
      sans-serif;
  }

  input {
    margin: 0.4rem;
  }
</style>
