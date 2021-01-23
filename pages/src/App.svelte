<script lang="ts">
  import Handlebars from 'handlebars';
  import { local } from './stores';
  import { promises as fs } from 'fs';
  import path from 'path';
  import { onMount } from 'svelte';
  import { ipcRenderer } from 'electron';
  interface form {
    disc: { value: string; modified: boolean };
    topic: { value: string; modified: boolean };
    pdfURL: { value: string; modified: boolean };
    desmosURL: { value: string; modified: boolean };
    repoURL: { value: string; modified: boolean };
    outputDir: { value: string; modified: boolean };
  }

  let form: form = {
    disc: { value: '', modified: false },
    topic: { value: '', modified: false },
    pdfURL: { value: '', modified: false },
    desmosURL: { value: '', modified: false },
    repoURL: { value: '', modified: false },
    outputDir: { value: '', modified: false },
  };

  const message = [
    'This is <b>discussion</b>',
    ", and we're talking <b>about</b>",
    '. The link to <b>pdf</b> is',
    ', and link to <b>desmos</b> is',
    'We\'re <i>uploading</i> it to',
    "and it <i>outputting</i> to ",
  ];

  onMount(() => {
    (Object.keys(form) as (keyof form)[]).forEach((key) => {
      const t = local.getItem(key);
      if (t) {
        form[key].value = t;
      }
    });
  });

  $: formEntries = Object.keys(form) as (keyof form)[];

  // Take in key of form
  const isLocalChange = (key: keyof form): boolean =>
    local.getItem(key) === form[key].value ? false : true;

  const getColor = (bool: boolean): string => (bool ? '#9DBF9E' : '#987284');

  // Writes file to corresponding locations
  const make = async () => {
    // Load the handlebars file from wherever necessary
    const hbs = (await ipcRenderer.invoke('isPackaged'))
      ? path.join(process.resourcesPath, 'app', 'pages', 'src', 'index.hbs')
      : path.join(path.resolve(), 'pages', 'src', 'html.hbs');
    // Also load the latex file from wherever necessary
    const tex = (await ipcRenderer.invoke('isPackaged'))
      ? path.join(process.resourcesPath, 'app', 'pages', 'src', 'tex.tex')
      : path.join(path.resolve(), 'pages', 'src', 'tex.tex');

    // Use handlebars to compile the html template for discussion
    const html_template = Handlebars.compile(await fs.readFile(hbs, 'utf-8'));
    // Decide on the folderpath based on the topics
    const folderpath = path.join(
      form.outputDir.value,
      `disc-${form.disc.value}`,
      `disc-${form.disc.value}-${form.topic.value}`
    );
    // Decide on the filepath based on the topics
    const html_filepath = path.join(
      folderpath,
      `disc-${form.disc.value}-${form.topic.value}.html`
    );

    // Just load the entire latex file I guess
    let tex_template = await fs.readFile(tex, 'utf-8');
    // Update tex date and topic
    tex_template = tex_template
      .replaceAll(/\\title{.*}/g, `\\title{${form.topic.value}}`)
      .replaceAll(/\\date{.*}/g, `\\date{${getMLADate()}}`);

    // Decide on latex filepath
    const tex_filepath = path.join(
      folderpath,
      `disc-${form.disc.value}-${form.topic.value}.tex`
    );

    // Create directory if it doesn't exist
    await fs.mkdir(folderpath, { recursive: true });
    // Write the file(s)
    await fs.writeFile(html_filepath, html_template(form));
    await fs.writeFile(tex_filepath, tex_template);
    // Update local storage
    Object.entries(form).forEach(([key, value]) => {
      local.setItem(key, value.value);
    });
    console.log(`Successfully added ${tex_filepath} and ${html_filepath}`);
  };

  const getMLADate = (): string => {
    const date = new Date();
    return `${date.getDate()} ${date.toLocaleString('default', {
      month: 'long',
    })} ${date.getFullYear()}`;
  };
</script>

<main>
  <div class="form">
    <div class="main">
      {#each formEntries.slice(0, formEntries.length - 2) as key, i}
        <span>
          <span>
            {@html message[i]}
          </span>
          <input
            type="text"
            placeholder={key}
            bind:value={form[key].value}
            on:input={() => (form[key].modified = isLocalChange(key))}
          />
          <span style="color: {getColor(form[key].modified)}">
            {#if form[key].modified}😍{:else}🤔{/if}
            {local.getItem(key)}
          </span>
        </span>
      {/each}
    </div>
    <div class="off">
      {#each formEntries.slice(formEntries.length - 2, formEntries.length) as key, i}
        <span>
          <span>
            {@html message[i + formEntries.length - 2]}
          </span>
          <input
            type="text"
            placeholder={key}
            bind:value={form[key].value}
            on:input={() => (form[key].modified = isLocalChange(key))}
          />
        </span>
      {/each}
    </div>
  </div>
  <div class="action"><button on:click={make}>MAKE</button></div>
</main>

<style lang="scss">
  :global(body) {
    margin: 0;
    padding: 0;
  }
  main {
    // Max out content
    width: 100%;
    height: 100%;
    max-width: 100%;
    max-height: 100%;
    display: grid;
    grid-template:
      'form action' /
      minmax(0, 2fr) minmax(0, 1fr);
    // Make it more native, turn it on later if needed
    user-select: none;
  }

  .form {
    grid-area: form;
    padding: 1rem;
    display: grid;
    grid-template:
      'main' minmax(0, 2fr)
      'off' minmax(0, 1fr);
  }

  .main {
    grid-area: main;
  }

  .off {
    grid-area: off;
    font-size: 0.2rem;
  }
  // .repoURL,
  // .outputDir {
  //   font-size: 0.2rem;
  //   // grid-area: off
  // }

  .action {
    grid-area: action;
    padding: 1rem;
    display: grid;
    min-height: 0; /* NEW */
    min-width: 0;
  }
</style>
