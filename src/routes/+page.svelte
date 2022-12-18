<script lang="ts">
  interface StoredMusicConfig {
    filename: string;
    artist: string;
    name: string;
    category: string;
    points: number;
    questionTimings: { start: number; end: number };
    answerTimings: { start: number; end: number };
  }

  interface MusicConfig extends StoredMusicConfig {
    audio: HTMLAudioElement;
    stopTimeout: ReturnType<typeof setTimeout> | null;
  }

  let musicConfigs: MusicConfig[] | null = null;
  let configsFileHandle: FileSystemFileHandle | null = null;
  const categories = ['Первая', 'Вторая', 'Третья', 'Четвёртая', 'Пятая'];
  const points = [100, 200, 300, 400, 500];

  async function startAudio(config: MusicConfig, timings: 'question' | 'answer') {
    console.log('Start playing');
    const timingsKey = `${timings}Timings` as const;
    config.audio.currentTime = config[timingsKey].start;
    await config.audio.play();

    config.stopTimeout = setTimeout(
      () => stopAudio(config),
      (config[timingsKey].end - config[timingsKey].start) * 1000,
    );

    musicConfigs = musicConfigs;
  }

  function stopAudio(config: MusicConfig) {
    if (config.stopTimeout) clearTimeout(config.stopTimeout);
    config.audio.pause();
    config.stopTimeout = null;
    musicConfigs = musicConfigs;
  }

  function sortConfigs() {
    if (!musicConfigs) return;

    musicConfigs = musicConfigs.sort((a, b) => {
      return categories.indexOf(a.category) - categories.indexOf(b.category) || a.points - b.points;
    });
  }

  async function saveConfigs() {
    if (!configsFileHandle || !musicConfigs) return;
    const stream = await configsFileHandle.createWritable({ keepExistingData: false });
    await stream.write(
      JSON.stringify(
        musicConfigs.map(
          (c): StoredMusicConfig => ({
            filename: c.filename,
            artist: c.artist,
            name: c.name,
            category: c.category,
            points: c.points,
            questionTimings: c.questionTimings,
            answerTimings: c.answerTimings,
          }),
        ),
      ),
    );
    await stream.close();
  }

  async function pickFolder() {
    const dir = await window.showDirectoryPicker();

    if ((await dir.requestPermission({ mode: 'readwrite' })) === 'denied') {
      alert('А всё, я этот сайт больше не работает');
    }

    configsFileHandle = await dir.getFileHandle('config.json', { create: true });

    const musicConfigsFromFile: StoredMusicConfig[] = await configsFileHandle
      .getFile()
      .then((configFile) => configFile.text())
      .then((str) => JSON.parse(str))
      .catch(() => []);

    const musicFilesHandles: FileSystemFileHandle[] = [];

    for await (const handle of dir.values()) {
      if (handle.isDirectory || !handle.name.endsWith('.mp3')) continue;
      musicFilesHandles.push(handle);
    }

    musicConfigs = await Promise.all(
      musicFilesHandles.map(async (handle): Promise<MusicConfig> => {
        const file = await handle.getFile();
        const audio = new Audio(URL.createObjectURL(file));

        await new Promise<void>((res) => {
          const listener = () => {
            audio.removeEventListener('loadedmetadata', listener);
            res();
          };
          audio.addEventListener('loadedmetadata', listener);
        });

        return {
          filename: file.name,
          artist: '',
          name: file.name,
          audio,
          category: '',
          points: 0,
          questionTimings: { start: 0, end: audio.duration },
          answerTimings: { start: 0, end: audio.duration },
          stopTimeout: null,
          ...musicConfigsFromFile.find((c) => c.filename === handle.name),
        };
      }),
    );

    sortConfigs();
    await saveConfigs();
  }
</script>

<main>
  {#if musicConfigs?.length}
    <button on:click={saveConfigs}>Сохранить конфигурацию</button>
  {:else}
    <button on:click={pickFolder}>Выбрать папку с музыкой</button>
  {/if}

  {#if musicConfigs?.length}
    <div class="grid">
      <h3>Категория:</h3>
      <h3>Очки:</h3>
      <h3>Исполнитель:</h3>
      <h3>Название:</h3>
      <h3>Вопрос:</h3>
      <h3>Ответ:</h3>
      {#each musicConfigs as config}
        <select bind:value={config.category} on:change={sortConfigs}>
          {#each categories as category}
            <option value={category}>{category}</option>
          {/each}
        </select>
        <select bind:value={config.points} on:change={sortConfigs}>
          {#each points as point}
            <option value={point}>{point}</option>
          {/each}
        </select>
        <input type="text" bind:value={config.artist} />
        <input type="text" bind:value={config.name} />
        <div class="flex">
          от: <input
            class="full"
            type="number"
            step={1}
            bind:value={config.questionTimings.start}
          />
          до: <input class="full" type="number" step={1} bind:value={config.questionTimings.end} />
          {#if config.stopTimeout}
            <button on:click={() => stopAudio(config)}>⏹️</button>
          {:else}
            <button on:click={() => startAudio(config, 'question')}>▶️</button>
          {/if}
        </div>
        <div class="flex">
          от: <input class="full" type="number" step={1} bind:value={config.answerTimings.start} />
          до: <input class="full" type="number" step={1} bind:value={config.answerTimings.end} />
          {#if config.stopTimeout}
            <button on:click={() => stopAudio(config)}>⏹️</button>
          {:else}
            <button on:click={() => startAudio(config, 'answer')}>▶️</button>
          {/if}
        </div>
      {/each}
    </div>
  {/if}
</main>

<style>
  main {
    margin: 2rem;
    font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
  }
  input,
  option,
  select,
  button {
    font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
    font-size: 1rem;
  }
  .grid {
    display: grid;
    grid-template-columns: 2fr 1fr 2fr 3fr 5fr 5fr;
    row-gap: 1rem;
    column-gap: 2rem;
    margin-top: 2rem;
  }
  .flex {
    display: flex;
    gap: 0.5rem;
  }
  .full {
    width: 100%;
  }
</style>
