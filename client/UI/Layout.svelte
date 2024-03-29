<script context="module">
  const is_browser = typeof window !== "undefined";

  import CodeMirror, { set, update } from "svelte-codemirror";
  import "codemirror/lib/codemirror.css";

  if (is_browser) {
    import("../utils/codeMirrorPlugins");
  }
</script>

<script>
	import { onMount, onDestroy } from 'svelte';
	import Inspect from 'svelte-inspect';

  import {  grammarEditorValue, 
            modelEditorValue, 
            grammarCompiledParser, 
            grammarCompilationErrors, 
            liveCodeEditorValue,
            liveCodeParseResults,
            liveCodeParseErrors,
            liveCodeAbstractSyntaxTree,
            dspCode,
            selectedLayout, 
            layoutOptions,
            helloWorld
  } from "../store.js";

  import { 
    playAudio,
    stopAudio,
    evalDSP
  } from '../audioEngine/audioEngineController.js';

  
  import * as nearley from 'nearley/lib/nearley.js'
  import compile from '../compiler/compiler';

  import Quadrants from './layouts/Quadrants.svelte';
  import Tutorial from './layouts/Tutorial.svelte';
  import Dashboard from './layouts/Dashboard.svelte';
  import Live from './layouts/Live.svelte';
  import Editor from './Editor.svelte';


  let codeMirror1, codeMirror2; // Live layout [Hidden]

  let codeMirror3, codeMirror4, codeMirror5; // []

  let codeMirror6, codeMirror7;

  export let layoutTemplate = 1;

  let liveContainerDisplay = "initial";
  let dashboardContainerDisplay = "initial";
  let quadrantsContainerDisplay = "initial";
  let tutorialContainerDisplay = "initial";

  $: doubled = changeLayout(layoutTemplate);

  function changeLayout (layoutIndex) {
    switch (layoutIndex) {
      case 1:
        liveContainerDisplay =      "none"; 
        quadrantsContainerDisplay = "none";  
        dashboardContainerDisplay = "none";  
        tutorialContainerDisplay = "initial";
        break;
      case 2:
        liveContainerDisplay =      "none";
        quadrantsContainerDisplay = "initial"; 
        dashboardContainerDisplay = "none"; 
        tutorialContainerDisplay = "none";
        break;
      case 3:
        liveContainerDisplay =      "none"; 
        quadrantsContainerDisplay = "none";  
        dashboardContainerDisplay = "initial";  
        tutorialContainerDisplay = "none";
        break;
      case 4:
        liveContainerDisplay =      "initial";
        quadrantsContainerDisplay = "none"; 
        dashboardContainerDisplay = "none";
        tutorialContainerDisplay = "none";
        break;
      default:
        liveContainerDisplay =      "initial";
        quadrantsContainerDisplay = "initial";  
        dashboardContainerDisplay = "initial";  
        tutorialContainerDisplay = "initial";
        break;
    }
  }

  const unsubscribe = selectedLayout.subscribe(value => {
    // console.log("DEBUG:Layout:selectedlayout: ", value.id);
    changeLayout(value.id);
  })  
	// onDestroy(unsubscribe); // Prevent memory leaks by disposing the component
  
  const unsubscribe2 = grammarEditorValue.subscribe(value => {
    // console.log("DEBUG:Layout:grammarEditorValue: ", value);
    //  grammarCompiledParser 
    // let liveParser = new nearley.Parser(nearley.Grammar.fromCompiled(grammarCompiled));
    // let c = compile(value)
    // let {errors, output} = c; 
    // console.log("DEBUG:Layout:grammarEditorValue: ", errors);
    // changeLayout(value.id);
  }) 

  onMount(async () => {
    codeMirror1.set($grammarEditorValue, "ebnf");
    codeMirror2.set($liveCodeEditorValue, "svelte");
    codeMirror3.set($liveCodeEditorValue, "svelte");
    codeMirror4.set($grammarEditorValue, "ebnf");
    codeMirror5.set($modelEditorValue, "js"); 
    // codeMirror6.set($grammarEditorValue, "ebnf");
    // codeMirror7.set($modelEditorValue, "js"); 
	});

  let log = (e) => { console.log(e.detail.value); }
  
  let nil = (e) => { }

  
  let parseLiveCode = e => { 
  
    if(window.Worker){

      let parserWorker = new Worker('../../parser.worker.js');

      let parserWorkerAsync = new Promise( (res, rej) => {

        parserWorker.postMessage({liveCodeSource: $liveCodeEditorValue, parserSource: $grammarCompiledParser, type:'parse'});

        let timeout = setTimeout(() => {
            parserWorker.terminate()
            parserWorker = new Worker('../../parser.worker.js')
            // rej('Possible infinite loop detected! Check your grammar for infinite recursion.')
        }, 5000);

        parserWorker.onmessage = e => {
          if(e.data.message !== undefined){
            // console.log('DEBUG:Layout:parserWorkerAsync:onmessage')
            // console.log(e); 
            $liveCodeParseErrors = e.data.message; 
          }
          else if(e.data !== undefined && e.data.length != 0){
            res(e.data);
          }
          clearTimeout(timeout);
        }

      })
      .then(outputs => {

        const {parserOutputs, parserResults} = outputs;
        
        // $liveCodeParseResults = outputs;
        $liveCodeParseResults = parserResults;

        $liveCodeAbstractSyntaxTree = parserOutputs;
        // $liveCodeAbstractSyntaxTree = JSON.parse(JSON.stringify(parserOutputs));
        $liveCodeParseErrors = "";
        // console.log('DEBUG:Layout:parserWorkerAsync'); 
      })
      .catch(e => { 
        console.log('DEBUG:Layout:parserWorkerAsync:catch') 
        console.log(e); 
      });
    }
  }


  let compileGrammarOnChange = e => { 
    let {errors, output} = compile(e.detail.value);
    $grammarCompiledParser = output; 
    $grammarCompilationErrors = errors;

    if($grammarCompiledParser && $liveCodeEditorValue){
      $liveCodeEditorValue = e.detail.value;
      parseLiveCode(); 
    }

    // console.log('DEBUG:Layout:compileGrammarOnChange');
    // console.log(e); 
  }


  let parseLiveCodeOnChange = e => {
    if($grammarCompiledParser){
      $liveCodeEditorValue = e.detail.value;
      parseLiveCode(); 
    }
    
    // console.log('DEBUG:Layout:parseLiveCodeOnChange');
    // console.log(e); 
  }

 

  let translateILtoDSPasync = e => {  // [NOTE:FB] Note the 'async'

    if(window.Worker){

      let iLWorker = new Worker('../../il.worker.js');
      let iLWorkerAsync = new Promise( (res, rej) => {

        iLWorker.postMessage({ liveCodeAbstractSyntaxTree: $liveCodeParseResults, type:'ASTtoDSP'});    
      
        let timeout = setTimeout(() => {
            iLWorker.terminate()
            iLWorker = new Worker('../../il.worker.js')
            // rej('Possible infinite loop detected or worse! Check bugs in ILtoTree.')
        }, 5000);

        iLWorker.onmessage = e => {
          if(e.data !== undefined){
            // console.log('DEBUG:Layout:translateILtoDSP:onmessage')
            // console.log(e); 
            // $dspCode = e.data.message;
            res(e.data); 
          }
          else if(e.data !== undefined && e.data.length != 0){
            res(e.data);
          }
          clearTimeout(timeout);
        }
      })      
      .then(outputs => {
        $dspCode = outputs;
        evalDSP($dspCode);
        
        // $liveCodeParseErrors = "";
        console.log('DEBUG:Layout:translateILtoDSPasync');
        console.log($dspCode);  
      })
      .catch(e => { 
        console.log('DEBUG:Layout:translateILtoDSPasync:catch') 
        console.log(e); 
      });      
    }    
  }

  let cmdEnter = () => {
    console.log('DEBUG:Layout:cmdEnter') 
    console.log($liveCodeAbstractSyntaxTree);     
    if($grammarCompiledParser && $liveCodeEditorValue && $liveCodeAbstractSyntaxTree){
      translateILtoDSPasync();
    }
  }
  
  let cmdPeriod = () => playAudio();

  let ctrlEnter = () => console.log("ctrl-enter");

</script>


<style>

  .layout-template-container {
    height: 100vh;
  }

	.scrollable {
		flex: 1 1 auto;
		border-top: 1px solid #eee;
		margin: 0 0 0.5em 0;
		overflow-y: auto;
	}

  .codemirror-container {
    position: relative;
    width: 100%;
    height: 100%;
    border: none;
    line-height: 1.5;
    overflow: hidden;
  }

  .codemirror-container :global(.CodeMirror) {
    height: 100%;
    background: transparent;
    font: 400 14px/1.7 var(--font-mono);
    color: var(--base);
  }

  .codemirror-container.flex :global(.CodeMirror) {
    height: auto;
  }

  .codemirror-container.flex :global(.CodeMirror-lines) {
    padding: 0;
  }

  .codemirror-container :global(.CodeMirror-gutters) {
    padding: 0 16px 0 8px;
    border: none;
  }

  .codemirror-container :global(.error-loc) {
    position: relative;
    border-bottom: 2px solid #da106e;
  }

  .codemirror-container :global(.error-line) {
    background-color: rgba(200, 0, 0, 0.05);
  }

	.scrollable {
		flex: 1 1 auto;
		border-top: 1px solid #eee;
		margin: 0 0 0.5em 0;
		overflow-y: auto;
	}
</style>


<!-- <div class="layout-template-container" contenteditable="true" bind:innerHTML={layoutTemplate}> -->
<div class="layout-template-container scrollable">

  <div class="tutorial-container" style="display:{tutorialContainerDisplay}">
    
    <Tutorial>
      <div slot="grammarEditor" class="codemirror-container flex scrollable">
        <CodeMirror bind:this={codeMirror1}  bind:value={$grammarEditorValue} tab={false} lineNumbers={true}  on:change={compileGrammarOnChange}  /> 
      </div>
      
      <div slot="liveCodeEditor" class="codemirror-container flex scrollable">
        <CodeMirror bind:this={codeMirror2}  bind:value={$liveCodeEditorValue} tab={false} lineNumbers={true} on:change={parseLiveCodeOnChange} cmdEnter={cmdEnter} ctrlEnter={ctrlEnter} cmdPeriod={cmdPeriod}/> 
      </div>

      <div slot="liveCodeCompilerOutput">
      {#if $grammarCompilationErrors !== ""}
        <div style="overflow-y: scroll; height:auto;">
          <strong style="color:red; margin:15px 0 15px 5px">Go work on your grammar!</strong>
        </div>
      {:else if $liveCodeAbstractSyntaxTree && $liveCodeAbstractSyntaxTree.length && $liveCodeParseErrors === ""}
        <div style="overflow-y: scroll; height:auto;">
          <strong style="color:green; margin:15px 0 15px 5px">Abstract Syntax Tree:</strong>
          <br>
          <div style="margin-left:5px">
          <!-- <div style="overflow-y: scroll; height:auto;"> -->
            <Inspect.Value value={$liveCodeAbstractSyntaxTree[0]['@lang']} depth={1} />
          </div>  
        </div>
      {:else}
        <div style="overflow-y: scroll; height:auto;">
          <strong style="color: red; margin:15px 0 10px 5px">SyntaxError: Invalid or unexpected token!</strong>
          <br>
          <div style="margin-left:5px">
          <!-- <div style="overflow-y: scroll; height:auto;"> -->
            <span style="white-space: pre-wrap">{ $liveCodeParseErrors } </span>
          </div>
        </div> 
      {/if}
      </div>


      <div slot="grammarOutput">
      {#if $grammarCompilationErrors !== ""}
        <div style="overflow-y: scroll; height:auto;">
          <strong style="color:red; margin:15px 0 15px 5px">Grammar compilation errors:</strong>
          <br>
          <div style="margin-left:5px">
          <!-- <div style="overflow-y: scroll; height:auto;"> -->
            <span style="white-space: pre-wrap">{ $grammarCompilationErrors } </span>
          </div>  
        </div>
      {:else}
        <div style="overflow-y: scroll; height:auto;">
          <strong style="color: green; margin:15px 0 10px 5px">Grammar validated and parser generated!</strong>
        </div> 
      {/if}
      </div>


    </Tutorial>
  </div>

  <div class="dashboard-container" style="display:{dashboardContainerDisplay}" >
    <!-- <Dashboard liveCodeEditorValue={value} grammarEditorValue={value} modelEditorValue={value} /> -->
    <Dashboard>
    </Dashboard> 
  </div>

  <div class="quadrants-container" style="display:{quadrantsContainerDisplay}">
    <!-- <Quadrants liveCodeEditorValue={value} grammarEditorValue={value} modelEditorValue={value}  /> -->
    <Quadrants>
      <div slot="viz">
        <!-- <Oscilloscope></Oscilloscope>
        <Spectrogram></Spectrogram> -->
      </div>
      <div slot="liveCodeEditor" class="codemirror-container flex scrollable">
        <CodeMirror bind:this={codeMirror3}  bind:value={$liveCodeEditorValue} lineNumbers={true} on:change={nil} /> 
      </div>
      <div slot="grammarEditor" class="codemirror-container flex scrollable">
        <CodeMirror bind:this={codeMirror4}  bind:value={$grammarEditorValue} lineNumbers={true} on:change={nil} /> 
      </div> 
      <div slot="modelEditor" class="codemirror-container flex scrollable">
        <CodeMirror bind:this={codeMirror5}  bind:value={$modelEditorValue} lineNumbers={true}  on:change={nil} /> 
      </div> 
    </Quadrants>
  </div>




  <div class="live-container" style="display:{liveContainerDisplay}">
    <Live>
      <div slot="liveCodeEditor" class="codemirror-container flex scrollable">
        <CodeMirror bind:this={codeMirror6}  bind:value={$liveCodeEditorValue} lineNumbers={true}  on:change={nil} /> 
      </div>
      <div slot="grammarEditor" class="codemirror-container flex scrollable">
        <CodeMirror bind:this={codeMirror7}  bind:value={$grammarEditorValue} lineNumbers={true}  on:change={nil} /> 
      </div>
    </Live>
  </div>


</div>
