<script>
	import { createEventDispatcher } from 'svelte';
  import { selectedLayout, layoutOptions} from '../store.js';

	const dispatch = createEventDispatcher();

  let selectedLanguage = 1;

  let languageOptions = [
		{ id: 1, text: `Default` },
		{ id: 2, text: `Bits` },
		{ id: 3, text: `IXI` },
		{ id: 4, text: `Maya` },
	];

	function sendLanguageSelect() {
    console.log("selectedLanguage: ", selectedLanguage);
    dispatch('message', {
			language: selectedLanguage.id
		});
	}

</script>

<style>
  
  .sidebar {
    background-color: rgb(255, 255, 151);
    width: 140px;
    height: 100%;
  }

</style>


<div class="sidebar">
  <div>
    <span>Sidebar</span>
  </div>
  <br/>
  <div class="layout-combobox-container">
    <div>
      <span>Layout</span>
    </div>
  	<select bind:value={$selectedLayout} >
  		{#each layoutOptions as layoutOption}
  			<option value={layoutOption}>
  				{layoutOption.text}
  			</option>
  		{/each}
  	</select>    
  </div>
  <br/>
  <div class="language-combobox-container">
    <div>
      <span>Language</span>
    </div>
  	<select bind:value={selectedLanguage} on:change="{ () => sendLanguageSelect() }">
  		{#each languageOptions as languageOption}
  			<option value={languageOption}>
  				{languageOption.text}
  			</option>
  		{/each}
  	</select>    
  </div>

</div>