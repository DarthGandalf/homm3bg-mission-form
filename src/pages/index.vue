<template>
  <v-container v-if="stage === 1">
    <v-text-field v-model="name" label="Scenario name" required/>
    <v-text-field v-model="victory_cond" label="Victory condition" required/>
    <v-slider v-model="starting_gold" label="Starting gold" :min="5" :max="30" :step="5" thumb-label="always"/>
    <v-btn @click="gotostage1()">Next</v-btn>
  </v-container>
  <v-overlay
      :model-value="overlay"
      class="align-center justify-center">
    <h1>Preparing LaTeX engine...</h1>
    <v-progress-circular
        color="primary"
        size="64"
        indeterminate
      ></v-progress-circular>
  </v-overlay>
  <v-container v-if="stage === 2">
    This is preview. You can edit the code as you wish, then click the Submit button, after that we'll look at it, and will help you making it look nicer.
    <v-textarea v-model="latex"/>
    <embed :src="embedpdf" width="800px" height="400px" type="application/pdf"/><br>
    <v-btn @click="submit()">Submit</v-btn>
    <v-text-field v-model="status" label="Status"/>
    <v-textarea v-model="log" label="Log"/>
  </v-container>
</template>

<script lang="ts" setup>
import { ref, watch } from 'vue';
import Engine from './PdfTeXEngine.js'
const Eng: any = Engine;

const name = ref('');
const victory_cond = ref('');
const starting_gold = ref(10);

const overlay = ref(false);
const stage = ref(1);

const latex = ref('');
let engine: any;
const log = ref('');
const status = ref('');
const embedpdf = ref('');

let compiling = false;
let need_update = false;

async function gotostage1() {
  overlay.value = true;
  engine = new Eng.PdfTeXEngine();
  await engine.loadEngine();
  latex.value = `\\textbf{Name:} ${name.value}

\\textbf{Victory Condition:} ${victory_cond.value}

\\textbf{Starting Resources:} ${starting_gold.value} gold
  `;

  engine.writeMemFSFile("main.tex", `\\documentclass{article}

\\usepackage[T1]{fontenc}
\\usepackage[utf8]{inputenc}

\\begin{document}
\\input{mission.tex}
\\end{document}
  `);
  engine.setEngineMainFile("main.tex");
  await updatePdf();
  overlay.value = false;
  stage.value = 2;
}

async function updatePdf() {
  engine.writeMemFSFile("mission.tex", latex.value);
  compiling = true;
  need_update = false;
  const render = await engine.compileLaTeX();
  status.value = render.status;
  log.value = render.log;
  const pdfblob = new Blob([render.pdf], {type : 'application/pdf'});
  const objectURL = URL.createObjectURL(pdfblob);
  setTimeout(()=>{
    URL.revokeObjectURL(objectURL);
  }, 30000);
  embedpdf.value = objectURL;
  compiling = false;
  maybeUpdate();
}

function maybeUpdate() {
  if (need_update && !compiling) {
    setTimeout(async ()=>{
      await updatePdf();
    }, 0);
  }
}

function submit() {
  alert("not implemented!");
}

watch(latex, (newValue, oldValue) => {
  need_update = true;
  maybeUpdate();
});

</script>
